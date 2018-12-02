---
title: RealWorldFinal
date: 2018-12-02 12:26:58
tags:
---

标杆比赛，一个不会，任重道远，加油。。。

# Pwn

## Sstation Escape

```
My treasure is yours for the taking, But you'll have to find it first. I left everything I own in host.

Both the username and password for the host and vm are rwctf
Try to spawn a caculator on the stage.
You need to replace /usr/lib/vmware/bin/vmware-vmx by the vmware-vmx-patched on the host desktop
download
hint 1:The vm is just an environment for debugging. The key point is the vmware-vmx-patched binary. If you wanna convey the challenge to remote players, it is enough to build a normal ubuntu 18.04 and run a vmware workstation in it then replace the file.(All files you need are on the desktop :)
```

文件下载不下来

## Engine for Neophytes

```
I've heard that @tsuro will precipitate the culmination of intricacy for browser exploitation in CTF games with some choreographed and elaborated bugs in the short run. Thus, here is an excellent opportunity for novices to originate and scrutinize your foolproof exploit for this painless bug.
download

NOTE: The challenge can be solved by spawning a calculator on the stage like Pwn2Own. The malformed page should be served in you notebook.
```

## The Pwnable Link

```
Hey guys, we've brought you the real IoT hacking challenge. You can find a home camera in the cabinet beside the table. Please hack it with the latest firmware under default configurations (except that the web admin password will be modified). This challenge requires you to demo the exploit on the stage. The demonstration needs to meet the requirements as below:

Attack from LAN 
Steal live video streaming from the camera and show it
Turn the camera up/down/left/right as the judges want
NOTE: Before you go on the stage, please make sure your debug information will not appear on the screen in case other teams see it.
There are two versions of the IP camera and you can check your camera's version on the camera body or in the web server page.
Please check the image in attachment to identify the version number.
Version 3.0's firmware is v3
Version 4.0's firmware is v4
The difference does not effect the solution. And we will prepare both versions of the camera for the demo. 
Some cameras are not in the latest version.So please update the firmware automatically or manually.
```

## router

```
There are many ways to manage a router, and I choose SNMP.

NOTE: The target device is inside the cabinet beside your table. The login information is root:rwctf. This weak credential is for debugging only.

Demonstration Requirements:

Attack from LAN
Get root shell of the router and show it
Hijack the domain realworldctf.com to a webpage saying hacked by <your team name>
NOTE: Before you go on the stage, please make sure your debug information will not appear on the screen in case other teams see it
```

## OBC Box

```
Is OBD Box Safe? There is a real car on the stage which has an OBD Box installed. This OBD Box will connect to its server and this server has the ability to control the car using the OBD Port.

I will give it to you the binary on the server and also on the OBD Box. A CAN Bus to USB Device hardware will also be offered to you if you really need this hardware to build a simulation environment in your own system, please go to the Operation & Maintenance Area to apply for it. (This challenge can be solved without the hardware.)

Now let’s have a full chain security test on it. Your target is to control some indicator lights on the meter panel to flash in a special pattern. (See demo video in the attachment). Reversing the binary on the OBD box would be helpful. You need to request for a demonstration when you are ready.

Note: The server is running windows 10 1703(OS version 15063.0).
```

## KitKot

```
Please don't get lost in the ACG site. Try remotely compromise it and show us your Calculator on the stage

download

Target will be running under Windows 10 1703(OS version 15063.0).
No user interaction is involved during demonstration.
Hint 1: A process with command line parameter --server is running on the target.
```



## frawler

```
Well, it turns out that the time machine we used to pwn suanjike is not a realworld thing :( Let's try something from the future without time traveling.
The flag is located at /pkg/data/flag in frawler's namespace.

nc 100.100.0.103 31337
No undisclosed bug in public codes required. (Technically they should not be called "0-day" as the entire stack is in experimental state anyway.)
```

文件下载不下来

# Web

## The Return of One Line PHP Challenge

```
What happens if I turn off session.upload? This challenge is almost identical to HITCON CTF 2018's challenge One Line PHP Challenge (Tribute to 🍊). Plz read the docker file and show me your shell. 
http://100.100.0.3
Please get shell in your local environment first, and then try on the remote server.
```

有dockfile,可以做

## Magic Tunnel

```
Must be a submarine to cross the English Channel?
```

## The Last Guardian

```
You need to host a webpage. By open it using our latest safari on macOS, can you run the following JavaScript code in browser under the specified domain?
alert('team@'+document.domain)

NOTE: You can use http://100.100.0.101 for debugging. To get you challenge solved, please register for demonstration.
```

## flaglab

```
You might need a 0 day
```

## Rmi

```
Remote Method Invocation? No, it's Remote Calculator Invocation.

Try to hack this RMI server and spawn a calculator on the stage.
Download and run the jar file with Oracle JDK 8u191 default configuration.
```



# misc

## rwext5

