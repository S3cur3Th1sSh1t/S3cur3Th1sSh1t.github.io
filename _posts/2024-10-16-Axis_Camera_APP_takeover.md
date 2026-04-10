---
title: "Axis Camera APP takeover"
layout: "post"
---

In 2018, Tenable published a blog post on how to get Remote Code Execution (RCE) on an Axis IP Camera with administrative credentials for the web application. By uploading a malicious APP file with the EAP extension, it's possible to execute code on the operating system level for persistence or data exfiltration.

<!--more-->

This blog post was written and published on my employer's website, where it can be found here:

- [https://www.r-tec.net/r-tec-blog-axis-camera-app-takeover.html](https://www.r-tec.net/r-tec-blog-axis-camera-app-takeover.html)

r-tec recently analysed an Axis IP Camera of the model `F9111` in a penetration test for one of our customers. We already had administrative credentials for the web interface of the camera, but the published [exploit](https://github.com/rapid7/metasploit-framework/blob/master/modules/exploits/linux/http/axis_app_install.rb) failed to takeover the operating system. This blog post describes our analysis steps and how we still took over the operating system via a slightly different way.

## 1. Initial PoC failure and analysis

According to the initial publication, the `EAP` APP files which can be installed on a camera with administrative privileges consist of a tar compressed elf binary plus several config files and HTML code. This tar compressed file is gzip compressed and just named with the extension `EAP`. The APP being analysed in the initial blog was "AXIS Video Motion Detection". Replacing the elf binary named `vmd` in that application did not lead to the attacker code getting executed at all, but instead the `POSTINSTALLSCRIPT=""` parameter from the `package.conf` file was manipulated to execute an arbitrary `/bin/sh` script for code execution.

A look at the published exploit shows that most of the values in the config file are random values, except for the `APPTYPE` which has the value lua:

<p align="center">
          <img src="/assets/posts/AxisCameraTakeover/Bild-1-conf-20240823151002.webp">
</p>

*Figure 1: Exploit package.conf configuration*

According to comments on the original exploit, this `APPTYPE` is used, because for other `APPTYPES` some required libraries are checked during the installation:

```ruby
# The eap registers as a "lua" apptype, because the binary version (armv7hf) gets checked
# for some required libraries whereas the lua version is just accepted.
```

Trying to use this exploit in our project however failed, as no proper Session was created:

<p align="center">
          <img src="/assets/posts/AxisCameraTakeover/Bild-2-no-session-20240823151156.webp">
</p>

*Figure 2: No session created*

In the camera's APP menu, it's possible to check the logs for installed applications. Changing the `appname` GET parameter to the name of our failed malicious APP gives us an indication of why the exploit failed:

<p align="center">
          <img src="/assets/posts/AxisCameraTakeover/Bild-3-missing-package-20240823123550.webp">
</p>

*Figure 3: Missing package error in APP logs*

So in this case, although `lua` was used as `APPTYPE`, we still get the message that packages are missing. At this point it was not acceptable to not get a session via this exploit. The APP installation feature is still there, it's still possible to upload unsigned `EAP` applications, this still has to be exploitable - or so was our thought.

## 2. The alternative

As the initial publication showed the config of one exemplary APP only, it was not the best to just 'guess' alternative `APPTYPES` for our `EAP` file. But where to get alternatives or more samples? Fortunately, the manufacturer itself provides a sort of [APP-Store](https://www.axis.com/products/analytics) for applications that can be installed on the camera.

Going through some of those, the "AXIS Live Privacy Shield" APP looked like a promising candidate for our malicious APP. The `package.conf` file looked like this:

```bash
PACKAGENAME="AXIS Live Privacy Shield"
APPTYPE="aarch64"
APPNAME="liveprivacyshield"
APPID="346005"
LICENSENAME="Available"
LICENSEPAGE="none"
VENDOR="Axis Communications"
REQEMBDEVVERSION="3.0"
APPMAJORVERSION="2"
APPMINORVERSION="5"
APPMICROVERSION="18"
APPGRP="sdk"
APPUSR="sdk"
APPOPTS=""
OTHERFILES=""
SETTINGSPAGEFILE="index.html"
SETTINGSPAGETEXT=""
VENDORHOMEPAGELINK='<a href="www.axis.com" target="_blank">www.axis.com</a>'
PREUPGRADESCRIPT=""
POSTINSTALLSCRIPT=""
STARTMODE="respawn"
HTTPCGIPATHS="cgi.conf"
```

In this case, the `APPTYPE` used was `aarch64` instead of `armv7hf` from the "Video Motion Detection" APP. First, we tried to embed our own shell script and execute it via the `POSTINSTALLSCRIPT` configuration parameter. But that unfortunately failed with different variations. After the project was already finished, we noticed another comment in the exploit, which states that `sync` and `sleep` commands are needed in between the malicious payload, so this might still work and is worth a try for anyone reading this blog. ;-)

<p align="center">
          <img src="/assets/posts/AxisCameraTakeover/Bild-5-sync-sleep-20240823125348.webp">
</p>

*Figure 5: Sync and sleep commands in exploit*

But instead of just accepting what was written in the initial publication, we also tried to replace the `liveprivacyshield` elf binary with our own malicious one as a next step:

<p align="center">
          <img src="/assets/posts/AxisCameraTakeover/Bild-6-liveprivacysheld-20240823125532.webp">
</p>

*Figure 6: Replacing the liveprivacyshield binary*

To get our final `EAP` application, we first need to compress our files into the tar format and then `gzip` compress them as mentioned before:

<p align="center">
          <img src="/assets/posts/AxisCameraTakeover/Bild-7-gzip-20240823125631.webp">
</p>

*Figure 7: Creating the EAP file with tar and gzip*

**Be aware: If you don't rename the `.gz` file to `.eap`, the installation of the APP will fail.**

The APP can then be uploaded and launched automatically or manually:

<p align="center">
          <img src="/assets/posts/AxisCameraTakeover/Bild-8-App-20240823125802.webp">
</p>

*Figure 8: APP uploaded and installed on the camera*

As you might see, it's possible to disallow unsigned APPs for installation. This might be configured on your target system as well and is therefore another source for a non working exploit. But in that case, it should be easily possible to just plug this option so that unsigned APPs are allowed again.

In our case, running the application leads to an incoming Meterpreter session:

<p align="center">
          <img src="/assets/posts/AxisCameraTakeover/Bild-9-meterpreter-20240823135740.webp">
</p>

*Figure 9: Incoming Meterpreter session*

However, the user is not `root` as in the original release, but a low-privileged, newly created service account. Later in this project, we found that different APPS run under different user accounts. However, in our case, manipulating the `APPUSR` parameter in the configuration did not result in a different user account being used. We leave it as an exercise for the reader to try other application and `APPTYPE` combinations to get `root` here.

## 3. Conclusion & Measures

**For all the Red:**

In general, if you're in a penetration test and you're struggling to get an exploit to work as it **should** in theory, don't give up and dig deeper. It's not about scanning and throwing PoCs at target systems, it's about showing the client the biggest impact and risks to your scope. The next time you see an Axis camera in your project and are faced with the same behaviour, you will know what to do.

**For the Blue:**

The ability to get RCE by uploading a malicious APP is unlikely to be fixed at any point in the future, as it's "by design" and has been known to the vendor for over six years. However, there are several things you can do to prevent this scenario in your environment.

1. Ensure that the management interfaces for the camera are placed on a secure network, accessible only by jump hosts or the administrators themselves. This will prevent most potential attackers from accessing them in the first place - therefore no vulnerability can be exploited. **Don't put the web interface on the perimeter so that it is accessible from the Internet** (yes, we've seen this before).
2. Use strong, complex passwords for administrative accounts to prevent attackers from guessing or brute forcing credentials. Doing these two things will reduce the risk of misuse **a lot**.

---
*This blog post was originally published on [r-tec.net](https://www.r-tec.net/r-tec-blog-axis-camera-app-takeover.html).*