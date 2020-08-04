---
layout: post
title: Home Assistant - PC Sleep with PSUniversal
subtitle: Time for some home automation!
tags: [PowerShell,PSUniversal,Home Assistant,SmartHome]
comments: true
---

I've always had an interest in the smart home technology, and I've used some pretty basic wifi switches with Amazon Echo Dots over the years. But recently, I setup a Home Assistant server on a Raspberry Pi to start taking that up a notch. 

For anyone not aware of what Home Assistant is, it's a open source Home Automation platform with tons of great integrations and plugins. Many manufacturers have built in support for it and many in the community have expanded that even further. There are customizable dashboards to see your home automation stuff in one view, as well as Scripts to execute steps for devices and automations to turn of and on things based off various triggers. I encourage any DIY tech enthusiast with an interest in the Smart Home area to take a look! https://www.home-assistant.io/

Back on the project at hand. I've been toying with fun automations, such as being able to shutdown my monitors at the end of the day with a simple command or scheduling it to come on automatically at the start of my day. But I've always just left my personal computer on. It's not a bad thing, but I really wanted to be able to turn it off and on at whim. I opted for utilizing the Sleep functionality of Windows 10, because this allows me to easily use Home Assistant's Wake-On-Lan functionality to send a packet and wake up the computer anytime I wanted.

There are many guides for a number of different ways to achieve the shutdown aspect, but all I tried were unsuccessful or wildy complicated. And then it hit me, PSUniversal by Ironman Software. I've been using the original release called Universal Dashboard for the last year at work to programatically create webpages via PowerShell code, and it has been amazing. Especially for this guy, who has no WebUI experience.

PSUniversal is the next iteration of this, providing all the functionality Universal Dashboard did, but also some great automation functionalities and API functionality. This ability to create the API call is what I decided to use.

First, the sleep command to be run. Unfortunately, no easy PowerShell command to do it. There are some Windows cmds to do it, but honestly I just wanted something simple. Enter [Nircmd](https://www.nirsoft.net/utils/nircmd.html), a great utility I've used in the past to expand some great functionalities not built into Windows. In this case, it's as simple as this command.

```
nircmd.exe standby
```

Next, I needed PSUniversal installed. You can find the download options [here](https://ironmansoftware.com/downloads/). I am in no way paid to recommend any of the software here today, but I do encourage you to take a look. In my case, I'm going to utilize the Windows MSI install. It installs all the files, sets up a service and starts it. The service runs the website, which can be found at localhost:5000 on the system you installed on. Default install login is just admin and any password (it just needs it to pass the admin to login).

![LoginPage](/img/ha-pcsleep/msedge_2020-08-03_20-11-16.png)

Once logged in, we are going to head to the API tab on the left. Then hit Add Endpoints in the top right corner. The following window will popup. I'm going to name mine simply /sleep-pc and make it a POST method. The free version of PSUniversal does not allow authentication in the classic sense, but you could build some checks into the actual code of you want. Once this is filled out, hit OK.

![APISetup](/img/ha-pcsleep/msedge_2020-08-03_20-12-22.png)

Once created, it takes you to the view page for the API. It shows the script that is executed on the computer PSUniversal is installed on when the API call is made. I'm going to hit Edit in the top right, which will allow me to edit the PowerShell code right in the same window.

![ScriptSetup](/img/ha-pcsleep/msedge_2020-08-03_20-15-35.png)

In this case, the script is pretty simple.

```
Start-Process -FilePath F:\nircmd-x64\nircmd.exe -ArgumentList 'standby'
```

Hit save in the top right corner.  You are now done with this part! This can easily be tested out by making the appropriate API call against the system. Back on the main API endpoints page, you can hit the Copy URL button next to your new API to get the exact path you need for the call.

![APIUrl](/img/ha-pcsleep/msedge_2020-08-03_20-18-26.png)

If you have another computer in the network, you can run the command from there. Or you can just run it on the same system in PowerShell and it should put the computer to sleep.

```
Invoke-RestMethod -uri 'http://x.x.x.x:5000/sleep-pc' -Method POST
```

Once you've confirmed that part, it's time to set this up in Home Assistant! I'm going to make an assumption for this part, that you are already involved within Home Assistant and have some fore knowledge on how to take certain steps before you dig into this further. If you don't, I strongly encourage you take a step back from this particular project and do some initial familiarization with the platform as a whole.

We are going to head into the configuration.yaml file to add the entries for both the Wake-on-lan and the sleep api call. I installed the File Editor plugin that lets me do this right inside the Home Assistant webpage.

In a section of your yaml file, wherever you prefer from an organization standpoint, you will use the following code. Be sure to put in the MAC address to the NIC on the system you want to wake up, as well as the IP to that same system which should be running PSUniversal. Feel free to give them whatever names you prefer. You'll notice we are using curl for the API call, since this is on a raspberry pi linux image.

```
switch:
  - platform: wake_on_lan
    mac: '00:00:00:00:00:00'
    name: "PC wake up"
    
  - platform: command_line
    switches:
        pc_sleep:
            command_off: 'curl -X POST -k http://x.x.x.x:5000/sleep-pc -d " "'

```

Once this is done, save the configuration.yaml and restart Home Assistant. You will now be able to reference these switch entities in automation or scripts to either put the PC to sleep or to wake it back up.

The last part is fairly straight forward, you can now add these both as switches on your Home Assistant dashboard. I took it a step further and formatted them simply as buttons on my dashboard, and here is that code.

Wakeup Button looks like this:

![Wakeup](/img/ha-pcsleep/msedge_2020-08-03_20-29-06.png)
```
entity: switch.pc_wake_up
hold_action:
  action: more-info
icon: 'mdi:power-standby'
show_icon: true
show_name: true
show_state: false
tap_action:
  action: toggle
type: button
```

And the Sleep button like this:

![Sleep](/img/ha-pcsleep/msedge_2020-08-03_20-29-40.png)
```
entity: switch.pc_sleep
hold_action:
  action: more-info
icon: 'mdi:power-sleep'
name: PC Sleep
show_icon: true
show_name: true
show_state: false
tap_action:
  action: toggle
type: button

```

The great thing about this merging of PSUniversal and Home Assistant is it opens up other automation opportunities. I plan to pursue more scenarios myself. I hope that this article has been informative and helpful, or at least stoke some creative thinking when it comes to utilizing tools like these for automation and ease!