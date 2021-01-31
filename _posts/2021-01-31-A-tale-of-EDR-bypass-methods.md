---
title: "A tale of EDR bypass methods"
layout: "post"
---

In a time full of ransomware as well as Advanced persistent Thread (APT) incidents the importance of detecting those attacking groups has become increasingly important. Some years ago the best tools/techniques for security incident detection and response included a SIEM-system filled with logs from IPS/IDS systems, proxies, firewalls, AV-logs and so on. In the recent years, an in my personal opinion increasingly relevant component has been added - "Endpoint detection and response - EDR" systems and or features. The features of those EDR systems include live monitoring of endpoints, data analysis, Threat-detection and blocking as well as Threat-hunting capabilities. In both, penetration tests and red-team engagements, these systems `can` make it difficult to use the public offensive security toolings, as they are more often detected and blocked. However, theese systems have a weakness which allows attackers to bypass the protection. In this blog post I'm gonna summarize all EDR bypass methods I found so far. <!--more--> The tools/techniques listed may not be exhaustive, but are certainly helpful to get a good overview and, if necessary, a better understanding of how to use them.     

## Introduction

All those of you, who follow the Offensive-Security community will have come across the terms `Userland hooking`, `Syscalls`, `P/Invoke`/`D-Invoke` and so on again and again over the last two years. I myself came across several blog posts and tools which I didn't understand fully. I sometimes had the feeling that I need to build up my knowledge from scratch. As I did not need those "new" techniques in many cases, I postponed the study of these topics for some months. 

Due to the increasing number of security incidents, more and more companies build up a Security-Operations-Center (SOC) or Computer emergency response team (CERT). Another term is the "Cyber Defense Center". The main purpose of these units is to analyse emerging security incidents and to identify and block potential attackers. EDR systems are increasingly being implemented and used for analysis here in addition to the SIEM. Meanwhile the EDR bypass topics have become more and more relevant for us Offensive-Security guys. Long story short: I had to dig into those topics now for myself, to be able reproducing and using the public techniques. And I thought the best way to motivate myself is writing a blog post about that topic. The tools and techniques, which are actually published are much older than the references I`m gonna refer to in this post. They were already <a href="https://www.cyberbit.com/blog/endpoint-security/malware-mitigation-when-direct-system-calls-are-used/">actively used by malware</a> in the wild before. This blog post will be a summarization of the tools/techniques I found public. I highly recommend you to read all those other blog posts linked here. They contain way more information and background knowledge. Before we dive into the main topic, we have to take a look at some Windows Operating System architecture basics as well as a small part about assembler code. Feel free to skip that part.

## Assembler code

If you are writing a program, independent from the programming language, you will most likely use a compiler to build the program from the coresponding source code. The source code snippets are basically translated to Machine Language, which is in the very end binary code like `01010011 00110011 01100011 01110101 01110010 00110011`, which can be directly executed by a CPU:

<p align="center">
          <img src="/assets/posts/EDRBypass/SourceCodeToMachineLanguage.JPG">
</p>

Some compilers, like `gcc` for example, produce assembler code before translating to Machine Code. Assembler Code instructions actually have an 1-to-1 mapping with Machine Code. So this is the closest to Machine Code and looks for example like that:

<p align="center">
          <img src="/assets/posts/EDRBypass/Assembler.JPG">
</p>

By disassembling a program via <a href="https://www.hex-rays.com/products/ida/">IDA Pro</a> or <a href="https://ghidra-sre.org/">Ghidra</a> you will also get assembler code back from an already compiled source code.

## Windows OS architecture

Programmers typically don't want to reinvent the wheel, so basic functions are imported from existing libraries. For example `printf()` is imported from the library `stdio.h` in the C-Language. For example Windows developers are using an application programming interface (API), which can also be imported in a program. The so called Win32 API is <a href="https://docs.microsoft.com/en-us/windows/win32/api/">documented</a> and consists of several library files (DLL-Files), located in the `C:\windows\system32\`  folder, like for example `kernel32.dll` , `User32.dll` and so on:

<p align="center">
          <img src="/assets/posts/EDRBypass/Architecture.JPG">
</p>

`NTDLL.dll` is not part of the Win32 API and is not officially documented.

## User-mode / Kernel-mode

The Windows OS has two different privilege levels, that were implemented to protect the Operating System from for example crashes caused by installed applications. All applications installed on a Windows System run in the so called `User-mode`. The kernel and device drivers run in the so called `Kernel-mode`. Applications in the User-mode cannot access or manipulate memory sections in the Kernel-mode. AV/EDR systems can only monitor application behaviour in the User-mode, due to the <a href="https://en.wikipedia.org/wiki/Kernel_Patch_Protection">Kernel Patch Protection</a>. And the very last instance in the User-mode are the Windows API functions from `NTDLL.dll`. If any function from `NTDLL.dll` is called, the CPU switches to Kernel-mode next, which cannot be monitored by AV/EDR vendors anymore. The single functions of `NTDLL.dll` are called `Syscalls`.

## Why should I care?

Where do we, as simulated attackers, need or use the Windows API? If we, for example, want to write specific bytes, such as shellcode, into a process we can import `WriteProcessMemory` from the file `kernel32.dll` with the following C#-Code snippet:

```batch
[DllImport("kernel32.dll", SetLastError = true)]
static extern bool WriteProcessMemory(IntPtr hProcess, IntPtr lpBaseAddress, byte[] lpBuffer, uint nSize, out UIntPtr lpNumberOfBytesWritten);
```

One example of how to write shellcode into a remote process using `kernel32.dll` functions can be found <a href="https://github.com/S3cur3Th1sSh1t/Creds/blob/master/Csharp/CreateRemoteThread.cs">here</a>.

Another thing most of us make heavy use of are PE-Loaders. In the most situations we like to stay in memory with our implants as long as possible, to not leave any traces on disk and for AV-Evasion. So `Mimikatz` or any other C-written toolings have to be loaded from memory, which is done via PE-Loaders. Powersploits <a href="https://github.com/S3cur3Th1sSh1t/Creds/blob/master/PowershellScripts/Invoke-ReflectivePEInjection.ps1">Invoke-ReflectivePEInjection</a> or Casey Smith's <a href="https://github.com/S3cur3Th1sSh1t/Creds/blob/master/Csharp/PEloader.cs">C# PE-Loader</a> make heavy use of Windows API functions like `CreateRemoteThread`, `GetProcAddress`, `CreateThread` from `kernel32.dll`.

Last but not least - depending on which Command & Control framework you are using - most of them use Windows API functions for their modules.

But the functions included in the Win32 API files like `kernel32.dll`, `User32.dll` and so on don't have a direct translation to Machine Code, but are instead mapped to other functions from the native API `NTDLL.dll`. For example `writeProcessMemory`  from `kernel32.dll`  resolves to `NtProtectVirtualMemory` -> `NtWriteVirtualMemory` -> `NtProtectVirtualMemory` from `NTDLL.dll`. The first `Syscall`, `NtProtectVirtualMemory`, sets new permissions for the process and makes it writable, the seccond one `NtWriteVirtualMemory` actually writes the bytes and the third call restores the old permissions for the process. 

`NTDLL.dll`, the Native API, is therefore the last instance in front of the operating system.

## Userland Hooking

As of the `NTDLL.dll` functions are the last intance, that can be monitored for suspicious activities from attackers or malware by AV/EDR vendors, they are typically doing exactly that. They inject a custom DLL-file into every new process. You can find DLL files, loaded into a process from AV/EDR Vendors via for example Sysinternals `procexp64.exe`. You need to check the `Show Lower Pane` button in the `View` menu and afterwards check the button to show DLLs loaded:

<p align="center">
          <img src="/assets/posts/EDRBypass/ProcExpDLL.JPG">
</p>

After selecting your prefered process you will see the loaded DLL-files in the Lower Pane view section. In this case we see the DLL-files loaded by McAfee AV for a `cmd.exe`:

<p align="center">
          <img src="/assets/posts/EDRBypass/McAfeeHook.JPG">
</p>

`Powershell.exe` has much more injected DLLs from McAfee, most likely because it's monitored for many more use-cases.

As you can see, there are three DLL-files injected by McAfee and one is called "Thin Hook Environment" - most likely the DLL that monitors Windows API calls.

So, theese loaded DLL-files monitor the process in which they are injected for specific Windows API calls. In my last blog posts I wrote about AV-Evasion in the form of signature changes, encryption and decryption at runtime and so on. If we encrypt our shellcode and decrypt that at runtime to write it into a remote process we can call `writeProcessMemory`, which under the hood calls `NtWriteVirtualMemory` at some point. One possible AV/EDR vendor goal can be to see what an attacker exactly loads into memory on runtime. So they can monitor `NtWriteVirtualMemory` calls. But how is this "monitoring" done? 

If a program loads a function like `NtWriteVirtualMemory` from `kernel32.dll`, a copy of `kernel32.dll` is placed into memory. The AV/EDR vendors typically manipulate the in memory copy of this file and add their own code into specific functions, like `NtWriteVirtualMemory`. When the function is called by the program, the AV/EDRs additional code is executed first, which does in the case of `NtWriteVirtualMemory` for example an analysis of the bytes, which shell be written into the remote process. By using this technique, they can see the cleartext shellcode bytes, because they are already decrypted in this moment. The AV/EDR vendors technique of embedding their own code in memory by patching API functions is called `Userland-Hooking`.

By loading a custom `Invoke-Mimikatz` version like I did in my seccond blog post <a href="https://s3cur3th1ssh1t.github.io/Bypass-AMSI-by-manual-modification-part-II/">Bypass AMSI by manual modification part II</a> with defender enabled on a system, the in-memory-scanner catches Mimikatz from memory after decryption and PE-loading. If you take a look at the code again - the decryption is done first, and the PE-Loader runs afterwards. We know now, that the PE-Loader calls several potentially suspicious Windows API calls. Those calls trigger the in-memory-scanner. So avoiding the calls will result in no memory scan at all.

The `Userland-Hooking` techniques made public till now, which I'm aware of are  `unhooking` the hook somehow, `re-patch it` in memory, `patching the AV/EDRs DLL`, or avoid loading Windows API function by using direct `Syscalls`.

## Patching the patch

There were blog posts by <a href="https://twitter.com/SpecialHoang">@SpecialHoang</a> and <a href="https://www.mdsec.co.uk/">MDsec</a> in the beginning of 2019 explaining how to bypass AV/EDR software by patching the patch:

* <a href="https://medium.com/@fsx30/bypass-edrs-memory-protection-introduction-to-hooking-2efb21acffd6">https://medium.com/@fsx30/bypass-edrs-memory-protection-introduction-to-hooking-2efb21acffd6</a>
* <a href="https://www.mdsec.co.uk/2019/03/silencing-cylance-a-case-study-in-modern-edrs/">https://www.mdsec.co.uk/2019/03/silencing-cylance-a-case-study-in-modern-edrs/</a>

If your implant or tool loads some functions from `kernel32.dll` or `NTDLL.dll`, a copy of the library file is loaded into memory. The AV/EDR vendors typically patch some of the functions from the in memory copy and place a `JMP` assembler instruction at the beginning of the code to redirect the Windows API function to some inspecting code from the AV/EDR software itself. So before calling the real Windows API function code, an analysis is done. If this analysis results in no suspicious/malicious behaviour and returns a clean result, the original Windows API function is called afterwards. If something malicious is found, the Windows API call is blocked or the process will be killed. I stole a nice picture from <a href="https://www.ired.team/offensive-security/defense-evasion/bypassing-cylance-and-other-avs-edrs-by-unhooking-windows-apis">ired.team</a>, which may help for understanding the process:

<p align="center">
          <img src="/assets/posts/EDRBypass/Hook.JPG">
</p>

Both blog posts focus on bypassing the EDR-software CylancePROTECT and build a PoC code for this specific software. By patching the additional `JMP` instruction from the manipulated `NTDLL.dll` in memory, the analysis code of Cylance will never be executed at all. Therefore no detections/blockings can take place:

<p align="center">
          <img src="/assets/posts/EDRBypass/CylanceBypass.JPG">
</p>

One disadvantage for this technique is, that you may have to change the patch for every different AV/EDR vendor. It is not very likely, that they all place an additional `JMP` instruction in front of the same functions at the same point. They will most likely hook different functions and maybe use another location for their patch. If you already know, which AV/EDR solution is in place in your target environment, you can use this technique and you will be fine bypassing the protection by patching the patch.

I also found a repo containing PDF-files with AV/EDR vendors and their corresponding hooked Windows API functions, take a look at this here if your interested:
* <a href="https://github.com/D3VI5H4/Antivirus-Artifacts">https://github.com/D3VI5H4/Antivirus-Artifacts</a>

## Outflankl's Dumpert and direct system calls

<a href="https://www.outflank.nl/">Outflanknl</a> released a tool called <a href="https://github.com/outflanknl/Dumpert">Dumpert</a> from a <a href="https://outflank.nl/blog/2019/06/19/red-team-tactics-combining-direct-system-calls-and-srdi-to-bypass-av-edr/">blog post</a> on June 19, 2019, in which they explain the use of direct system calls to bypass `Userland-Hooking`. I will not cover all details from the blog post but only sum up the most important facts to understand this topic. The goal of the technique used here is to not load any functions from `ntdll.dll` at runtime, but instead call them directly with the corresponding assembler code. By disassembling the `ntdll.dll` file it's possible to get the assembler code for every single function contained. 

One problem here is, that the assembler code is different at some points between Windows OS versions and sometimes even between service pack/built numbers. Google project Zero did some <a href="https://j00ru.vexillium.org/syscalls/nt/64/">research</a> about the differences, so that they can be looked up on the linked website. By embedding all different assembler code versions for all OS-Versions it´s possible to check for the underlying operating system on runtime and choose the correct assembler code for the needed Windows API function. Assembler code can be embeded in C-Projects via Visual Studio by using ASM-files. The Dumpert project is therefore using an ASM-file, which contains all nessesary Windows API functions in assembler code for each Windows version:

<a href="https://github.com/outflanknl/Dumpert/blob/master/Dumpert-DLL/Outflank-Dumpert-DLL/Syscalls.asm">https://github.com/outflanknl/Dumpert/blob/master/Dumpert-DLL/Outflank-Dumpert-DLL/Syscalls.asm</a> 

<p align="center">
          <img src="/assets/posts/EDRBypass/DumpertSyscall.JPG">
</p>

To use this technique you need to know the exact `NTDLL.dll` functions needed for your project and extract the corresponding assembler code for them via disassembling. Afterwards you need to build an ASM-file containing all different offsets for different Windows OS-Versions. Sounds complicated.

Using this technique also has some disadvantages: 

* Your binary will not work anymore, whenever a newer Windows version is released. Thats because the assembler code for each function has to be changed again. So you need to build a new implant/tool version whenever changes are released by Microsoft
* Disassembling all the Windows API functions is a lot of effort and needs a lot of time/work

But using this technique will enable us to bypass `Userland-Hooking` in general. This technique is independent from different vendors. They all will not see any Windows API function imports or calls at all. No function imports -> no patch/hook by the AV/EDR software -> stealth/bypass.

## Syswhispers

With the release of the tool <a href="https://github.com/jthuraisamy/SysWhispers">SysWhispers</a> it became much easier to create custom ASM-files with the corresponding C-Header files. The manual overhead for disassembling `ntdll.dll` is left out. Building the ASM and Header-File became straight forward by executing a single python script:

<p align="center">
   <img src="/assets/posts/EDRBypass/SysWhispers.JPG">
</p>

~1 Month ago <a href="https://github.com/jthuraisamy/SysWhispers2">SysWhispers2</a> was released, which reduces the size of ASM-files and makes use of randomized function name hashes on each generation. The first version will be deprecated in the future so you should use the supported version 2.

Dumpert, Syswhispers and Syswhispers2 currently only support x64 Syscalls. If you need x86 Syscalls, there is <a href="https://github.com/mai1zhi2/SysWhispers2_x86">SysWhispers2_x86</a> just released on Github.

If you don't want to write your toolings/implant in C, you can also get your hands dirty with <a href="https://github.com/ajpc500/NimlineWhispers">NimlineWhispers</a>, which builds the ASM-file and the header file for Nim-Code. <a href="https://twitter.com/ajpc500">@ajpc500</a> also wrote a good blog post about how to use NimlineWhispers for Shellcode Injection via Nim. Check the blog post out <a href="https://ajpc500.github.io/nim/Shellcode-Injection-using-Nim-and-Syscalls/">here</a>. I played with the Nim syscall Shellcode Injection PoCs for myself and it works like a charm! Be aware, that using the default `NTDLL.dll` function names will result in a binary containing them in cleartext, visible via any hexeditor:

<p align="center">
          <img src="/assets/posts/EDRBypass/NimlineWhispersASM.JPG">
</p>

Talking with <a href="https://twitter.com/IKalendarov">@IKalendarov</a> about NimlineWhispers, he found that Windows Defender with Cloud protection enabled executes the Shellcode successfully but throws an alert stating `Defensive Evasion detected` afterwards:

<p align="center">
          <img src="/assets/posts/EDRBypass/DefensiveEvasion.JPG">
</p>

I found, that this detection can easily be bypassed by renaming the Windows API functions in the ASM-File and of course also in the shellcode injection code. `NtAllocateVirtualMemory` becomes `NtAVM` for example and so on. If your shellcode itself or the code behind it contains any Windows API function imports - this can be detected again. So the shellcode loader and the shellcode itself should use Syscalls to stay undetected from `Userland-Hooks`.

## P/Invoke to D/Invoke

<a href="https://twitter.com/TheRealWover">@TheRealWover</a> released a C# library called D/Invoke. First this was added to <a href="https://github.com/cobbr/SharpSploit">SharpSploit</a>, but later on TheWover released a nuget package ready for import in any VisualStudio project <a href="https://github.com/TheWover/DInvoke">here</a>. There also is a corresponding <a href="https://thewover.github.io/Dynamic-Invoke/">blog post</a> from June 2020. If you are mostly coding in C#, this is actually the easiest way for you to go for `Userland-Hooking` bypasses. I'm just gonna pick small parts out of TheWovers post, as this blog post here would explode by explaining everything. If you are new to this topic his blog post may be a little bit too "heavy". I didn't understand half reading it the first time. <a href="https://twitter.com/Jean_Maes_1994">@Jean_Maes_1994</a> released a blog post which sums up all techniques used via D/Invoke <a href="https://blog.nviso.eu/2020/11/20/dynamic-invocation-in-net-to-bypass-hooks/">here</a>. The resulting PoC code <a href="https://github.com/NVISO-BE/DInvisibleRegistry">DInvisibleRegistry</a> can be used to look up different D/Invoke implementation methods and is in my opinion really usefull and understandable. 

P/Invoke is basically the default way for statically importing API calls from a Windows library file. The `WriteProcessMemory` import from `kernel32.dll` shown above is the P/Invoke approach. AV/EDR systems are able to patch the in memory copy of Windows library files like `NTDLL.dll` by using this method.

D/Invoke - in comparison to P/Invoke - is loading a Windows API function manually at runtime and calls the function using a pointer to its location in memory. The manual loading of a library file at runtime is at the time of writing not detected by AV/EDR hooks, so that they don´t patch the freshly imported functions and they stay original without hook/patch.

There are three different methods to avoid Userland-Hooking via D/Invoke:

* `Manual Mapping` - this method loads a full copy of the target library file into memory. Any functions can be exported from it afterwards.

```batch
DInvoke.Data.PE.PE_MANUAL_MAP mappedDLL = new DInvoke.Data.PE.PE_MANUAL_MAP();
mappedDLL = DInvoke.ManualMap.Map.MapModuleToMemory(@"C:\Windows\System32\ntdll.dll");
```

* `OverloadMapping` - in addition to Manual Mapping the payload stored in memory is backed by a legitimate file on disk. So the payload will appear to be executed from a legitimate, validly signed DLL on disk.

```batch
DInvoke.Data.PE.PE_MANUAL_MAP mappedDLL = DInvoke.ManualMap.Overload.OverloadModule(@"C:\Windows\System32\ntdll.dll");
```

* `Syscalls` - using this technique not the whole target library is mapped to memory but only a specified function is extracted from it. This method therefore offers more stealth than `Manual Mapping`.

```batch
IntPtr pAllocateSysCall = DInvoke.DynamicInvoke.Generic.GetSyscallStub("NtAllocateVirtualMemory");
NtAllocateVirtualMemory fSyscallAllocateMemory = (NtAllocateVirtualMemory)Marshal.GetDelegateForFunctionPointer(pAllocateSysCall, typeof(NtAllocateVirtualMemory));
```

For every of the three methods you also need to create unmanaged `Delegates` for every Windows API function in your code. I won´t cover the whole process here as you can just read the linked blog posts from @TheRealWhover or @Jean_Maes_1994.

Initially I planned to show, how to port a P/Invoke `CreateRemoteThread` C# shellcode injection PoC into a D/Invoke `Syscall` version. I was fiddling around with all those `NTDLL.dll` functions needed like `NtOpenProcess`, `NtAllocateVirtualMemory`, `NtWriteVirtualMemory` and `CreateThreadEx` but was unfortunately not able to successfully get my shellcode execution working. This was because I never used those `NTDLL.dll` functions before and struggled hard with the questions "which value should be placed in which function argument", "which kernel32.dll function resolves to which ntdll.dll function" and had a brainfuck many evenings trying to get this to work. In parallel I confronted the awesome <a href="https://twitter.com/_RastaMouse">@_RastaMouse</a> with all my questions about it. It took only a few days and he published a whole blog post covering exactly this topic: 

<a href="https://offensivedefence.co.uk/posts/dinvoke-syscalls/">https://offensivedefence.co.uk/posts/dinvoke-syscalls/</a> 

So there is no need to show this a seccond time - `¯\_(ツ)_/¯` - I got my PoC working with the information from his blog post. Just read it by yourself.

## NTDLL.dll unhooking in C++ or Nim

We learned that AV/EDR systems hook specific functions of `NTDLL.dll` to place their own code for analysis in it. There is a nice and short article on <a href="https://www.ired.team/offensive-security/defense-evasion/how-to-unhook-a-dll-using-c++">ired.team</a> which explains how to map a fresh copy of `NTDLL.dll` from disk to memory, copying the `.text` section from the fresh copy into the `.text` section of the hooked file in memory, so that the hook is undone by overwriting it:

<p align="center">
          <img src="/assets/posts/EDRBypass/CppUnhook.JPG">
</p>

A C++ PoC code for the unhooking process as well as a step by step guide is also included. Go ahead reading it if you didn't so far.

Again - if someone is not that familiar with C/C++ coding - I recently <a href="https://s3cur3th1ssh1t.github.io/Playing-with-OffensiveNim/">played with OffensiveNim</a> and the <a href="https://github.com/byt3bl33d3r/OffensiveNim">OffensiveNim</a> repo contains a template named `clr_host_cpp_embed_bin.nim` in which we can embed pure C++ code. We can take this template and embed the C++ PoC from the ired.team website into it and we have a working `NTDLL.dll` unhooking PoC in Nim:

```batch

when not defined(cpp):
    {.error: "Must be compiled in cpp mode"}
# Stolen from https://www.ired.team/offensive-security/defense-evasion/how-to-unhook-a-dll-using-c++

{.emit: """
#include <iostream>
#include <Windows.h>
#include <winternl.h>
#include <psapi.h>

int test()
{
    HANDLE process = GetCurrentProcess();
    MODULEINFO mi = {};
    HMODULE ntdllModule = GetModuleHandleA("ntdll.dll");
    
    GetModuleInformation(process, ntdllModule, &mi, sizeof(mi));
    LPVOID ntdllBase = (LPVOID)mi.lpBaseOfDll;
    HANDLE ntdllFile = CreateFileA("c:\\windows\\system32\\ntdll.dll", GENERIC_READ, FILE_SHARE_READ, NULL, OPEN_EXISTING, 0, NULL);
    HANDLE ntdllMapping = CreateFileMapping(ntdllFile, NULL, PAGE_READONLY | SEC_IMAGE, 0, 0, NULL);
    LPVOID ntdllMappingAddress = MapViewOfFile(ntdllMapping, FILE_MAP_READ, 0, 0, 0);

    PIMAGE_DOS_HEADER hookedDosHeader = (PIMAGE_DOS_HEADER)ntdllBase;
    PIMAGE_NT_HEADERS hookedNtHeader = (PIMAGE_NT_HEADERS)((DWORD_PTR)ntdllBase + hookedDosHeader->e_lfanew);

    for (WORD i = 0; i < hookedNtHeader->FileHeader.NumberOfSections; i++) {
        PIMAGE_SECTION_HEADER hookedSectionHeader = (PIMAGE_SECTION_HEADER)((DWORD_PTR)IMAGE_FIRST_SECTION(hookedNtHeader) + ((DWORD_PTR)IMAGE_SIZEOF_SECTION_HEADER * i));
        
        if (!strcmp((char*)hookedSectionHeader->Name, (char*)".text")) {
            DWORD oldProtection = 0;
            bool isProtected = VirtualProtect((LPVOID)((DWORD_PTR)ntdllBase + (DWORD_PTR)hookedSectionHeader->VirtualAddress), hookedSectionHeader->Misc.VirtualSize, PAGE_EXECUTE_READWRITE, &oldProtection);
            memcpy((LPVOID)((DWORD_PTR)ntdllBase + (DWORD_PTR)hookedSectionHeader->VirtualAddress), (LPVOID)((DWORD_PTR)ntdllMappingAddress + (DWORD_PTR)hookedSectionHeader->VirtualAddress), hookedSectionHeader->Misc.VirtualSize);
            isProtected = VirtualProtect((LPVOID)((DWORD_PTR)ntdllBase + (DWORD_PTR)hookedSectionHeader->VirtualAddress), hookedSectionHeader->Misc.VirtualSize, oldProtection, &oldProtection);
        }
    }
    
    CloseHandle(process);
    CloseHandle(ntdllFile);
    CloseHandle(ntdllMapping);
    FreeLibrary(ntdllModule);
    
    return 1;
}
""".}
proc unhook(): int
    {.importcpp: "test", nodecl.}
when isMainModule:
    var result = unhook()
    echo "[*] Assembly executed: ", bool(result)
    # Every code from here is not hooked / detected from Windows API imports at runtime anymore
```

If you are looking for a language independent solution of unhooking `NTDLL.dll` I can recommend <a href="https://twitter.com/slaeryan">@slaeryans</a> <a href="https://github.com/slaeryan/AQUARMOURY/tree/master/Shellycoat">Shellycoat</a> shellcode.

By injecting this shellcode first - which can be done in any language - the same process of replacing the `.text` section of the hooked `NTDLL.dll` is done. After injecting Shellycoat you can inject your implant code, which will not get detected by hooks anymore. Slaeryan also covers different methods of how to unhook `NTDLL.dll` in the repo with Pros & Cons, thats worth reading it.

## SharpBlock - Patching the Entrypoint

<a href="https://twitter.com/_EthicalChaos_">@_EthicalChaos_</a> had a new approach on bypassing EDR systems. This is explained in two blog posts, <a href="https://ethicalchaos.dev/2020/05/27/lets-create-an-edr-and-bypass-it-part-1/">Lets create an EDR and bypass it part I</a> and <a href="https://ethicalchaos.dev/2020/06/14/lets-create-an-edr-and-bypass-it-part-2/">Lets create an EDR and bypass it part II</a> - also from June 2020 - with the resulting tool <a href="https://github.com/CCob/SharpBlock">SharpBlock</a>.

SharpBlock is using a different approach in comparison to the others before. It´s creating a new process and is using the Windows Debug API to listen for `LOAD_DLL_DEBUG_EVENT` events. SharpBlock is looking for the EDR's DLL to be loaded via debug API and patches the `Entrypoint` of this newly injected DLL so that it just returns `TRUE` instead of doing anything else. The target DLL will therefore do nothing and exits -> no hooks/patches again.

SharpBlock enables us to specify a target DLLs file-name or Description to patch it's Entrypoint. Playing with SharpBlock for this blog post I tried blocking out McAfees `EpMPThe.dll` with the following command:

```batch
SharpBlock.exe -d "McAfee Endpoint Thin Hook Environment" --disable-bypass-amsi -e "C:\Windows\System32\cmd.exe" --disable-bypass-etw --disable-header-patch -w
```

This resulted in the following behaviour:

<p align="center">
          <img src="/assets/posts/EDRBypass/SharpBlockFail.png">
</p>

I asked @_EthicalChaos_ about a possible reason for this failed block and he told me that this will most likely be the first protection mechanism against SharpBlock. In particular <a href="https://github.com/CCob/SharpBlock/blob/254e327cace1cb1d46c8892b871119ed2d6a4be2/Program.cs#L123">line 123</a> failed to execute which is the `WriteProcessMemory` function. As described above `WriteProcessMemory` resolves to `NtProtectVirtualMemory` and `NtWriteVirtualMemory` from `NTDLL.dll` and McAfee seams to block processes from changing it's hooking DLL's memory protection to `RWX` via `NtProtectVirtualMemory` or writing into it via `NtWriteVirtualMemory`. So Sharpblock itself was hooked with `EpMPThe.dll` and not able to patch the McAfee hooking DLL because of a `Userland-Hook`. This blog is about `Userland-Hooking` bypass methods - one way doing that is using direct Syscalls instead of API imports, right?

Using `D/Invokes` method `GetSyscallStub`  @_EthicalChaos_ changed the `WriteProcessMemory` function to direct Syscalls in <a href="https://github.com/CCob/SharpBlock/commit/c604db863db7e3c2ec42d60fc5475ebf558dbdf8">another branch</a>. In this branch `NtProtectVirtualMemory` and `NtWriteVirtualMemory` are called directly without a hook, so that SharpBlock patches McAfees hooking DLL successfully again:

<p align="center">
          <img src="/assets/posts/EDRBypass/SharpBlockSuccess.png">
</p>      

And - tada - the DLL is not loaded anymore:

<p align="center">
          <img src="/assets/posts/EDRBypass/SharpBlockSuccess.JPG">
</p>

## Hell's Gate VX technique

<a href="https://twitter.com/am0nsec">@am0nsec</a> and <a href="https://twitter.com/smelly__vx">@smelly__vx</a> released another technique of using direct Syscalls for Shellcode execution. They released a PoC code <a href="https://github.com/am0nsec/HellsGate">written in c</a> as well as a PoC <a href="https://github.com/am0nsec/SharpHellsGate">written in .NET Core</a>.

As far as I unterstood this from skimming the <a href="https://github.com/am0nsec/HellsGate/blob/master/hells-gate.pdf">official paper</a> "only" the method of retrieving the correct Syscall for functions from `NTDLL.dll` or other library files is different. So they are not extracted from the file directly. But I have to admit this paper is written in a "heavy" language - so for people like me that are not really deep in this subject it's hard to unterstand. I'm gonna read it again in some months and maybe I'll unterstand the approach better - sometimes waiting and reading other posts/papers is the key.

## Conclusion

This post has gotten way bigger than I planned. It's by far the one, where I had to spend the most hours of research for understanding this topic by myself. But that was worth it. For those people reading the article, which were already familiar to the topic - I hope you were not bored. On the other hand, you wouldn't be reading this last part now if you were bored. :-)

I went through all those different tools/techniques made public in the last years for bypassing `Userland-Hooking`. As the AV/EDR solutions from today tend to monitor the last instance of the User-Land which is `NTDLL.dll`, they patch it's library functions in memory and put their own code into it for runtime analysis of potentially malicious code. We can undo this patch by loading a fresh copy of `NTDLL.dll` and overwrite the hooked functions, we can `patch out the hook via patch` or use direct system calls via different techniques. We found PoC codes in different coding languages, so that C/C++-implants, C#-implants or Nim-implants are covered with bypass code. I hope that I was able to explain this topic in a more or less "light" manner so that people without background knowledge learned something.

Feedback and additions or corrections are strongly encouraged. You can reach me via the above channels as always.

## Links & Resources

* Syscalls in the wild - <a href="https://www.cyberbit.com/blog/endpoint-security/malware-mitigation-when-direct-system-calls-are-used/">https://www.cyberbit.com/blog/endpoint-security/malware-mitigation-when-direct-system-calls-are-used/</a>
* IDA Disassembler - <a href="https://www.hex-rays.com/products/ida/">https://www.hex-rays.com/products/ida/</a>
* Ghidra Disassembler - <a href="https://ghidra-sre.org/">https://ghidra-sre.org/</a>
* Win32 API documentation - <a href="https://docs.microsoft.com/en-us/windows/win32/api/">https://docs.microsoft.com/en-us/windows/win32/api/</a>
* Kernel Patch protection - <a href="https://en.wikipedia.org/wiki/Kernel_Patch_Protection">https://en.wikipedia.org/wiki/Kernel_Patch_Protection</a>
* CreateRemoteThread C# - <a href="https://github.com/S3cur3Th1sSh1t/Creds/blob/master/Csharp/CreateRemoteThread.cs">https://github.com/S3cur3Th1sSh1t/Creds/blob/master/Csharp/CreateRemoteThread.cs</a>
* Invoke-ReflectivePEInjection - <a href="https://github.com/S3cur3Th1sSh1t/Creds/blob/master/PowershellScripts/Invoke-ReflectivePEInjection.ps1">https://github.com/S3cur3Th1sSh1t/Creds/blob/master/PowershellScripts/Invoke-ReflectivePEInjection.ps1</a>
* PEloader C# - <a href="https://github.com/S3cur3Th1sSh1t/Creds/blob/master/Csharp/PEloader.cs">https://github.com/S3cur3Th1sSh1t/Creds/blob/master/Csharp/PEloader.cs</a>
* Bypass AMSI by manual modification part II - <a href="https://s3cur3th1ssh1t.github.io/Bypass-AMSI-by-manual-modification-part-II/">https://s3cur3th1ssh1t.github.io/Bypass-AMSI-by-manual-modification-part-II/</a>
* Bypass Userland Hooking by patching the patch - <a href="https://medium.com/@fsx30/bypass-edrs-memory-protection-introduction-to-hooking-2efb21acffd6">https://medium.com/@fsx30/bypass-edrs-memory-protection-introduction-to-hooking-2efb21acffd6</a>
* Bypass Userland Hooking by patching the patch from Mdsec - <a href="https://www.mdsec.co.uk/2019/03/silencing-cylance-a-case-study-in-modern-edrs/">https://www.mdsec.co.uk/2019/03/silencing-cylance-a-case-study-in-modern-edrs/</a>
* Unhook windows apis - <a href="https://www.ired.team/offensive-security/defense-evasion/bypassing-cylance-and-other-avs-edrs-by-unhooking-windows-apis">https://www.ired.team/offensive-security/defense-evasion/bypassing-cylance-and-other-avs-edrs-by-unhooking-windows-apis</a>
* Antivirus Artifacts - <a href="https://github.com/D3VI5H4/Antivirus-Artifacts">https://github.com/D3VI5H4/Antivirus-Artifacts</a>
* Dumpert - <a href="https://github.com/outflanknl/Dumpert">https://github.com/outflanknl/Dumpert</a>
* Syscalls Blog Post from Outflanknl - <a href="https://outflank.nl/blog/2019/06/19/red-team-tactics-combining-direct-system-calls-and-srdi-to-bypass-av-edr/">https://outflank.nl/blog/2019/06/19/red-team-tactics-combining-direct-system-calls-and-srdi-to-bypass-av-edr/</a>
* Project Zero Syscall reference - <a href="https://j00ru.vexillium.org/syscalls/nt/64/">https://j00ru.vexillium.org/syscalls/nt/64/</a>
* Syswhispers2 - <a href="https://github.com/jthuraisamy/SysWhispers2">https://github.com/jthuraisamy/SysWhispers2</a>
* Syswhispers2_x86 - <a href="https://github.com/mai1zhi2/SysWhispers2_x86">https://github.com/mai1zhi2/SysWhispers2_x86</a>
* NimLineWhispers - <a href="https://github.com/ajpc500/NimlineWhispers">https://github.com/ajpc500/NimlineWhispers</a>
* NimlineWhispers blog post - <a href="https://ajpc500.github.io/nim/Shellcode-Injection-using-Nim-and-Syscalls/">https://ajpc500.github.io/nim/Shellcode-Injection-using-Nim-and-Syscalls/</a>
* Sharpsploit - <a href="https://github.com/cobbr/SharpSploit">https://github.com/cobbr/SharpSploit</a>
* DInvoke - <a href="https://github.com/TheWover/DInvoke">https://github.com/TheWover/DInvoke</a>
* DInvoke blog post - <a href="https://thewover.github.io/Dynamic-Invoke/">https://thewover.github.io/Dynamic-Invoke/</a>
* NVISO DInvoke blog post - <a href="https://blog.nviso.eu/2020/11/20/dynamic-invocation-in-net-to-bypass-hooks/">https://blog.nviso.eu/2020/11/20/dynamic-invocation-in-net-to-bypass-hooks/</a>
* DInvisibleRegistry - <a href="https://github.com/NVISO-BE/DInvisibleRegistry">https://github.com/NVISO-BE/DInvisibleRegistry</a>
* Rastamouse DInvoke Syscalls - <a href="https://offensivedefence.co.uk/posts/dinvoke-syscalls/</a>https://offensivedefence.co.uk/posts/dinvoke-syscalls/</a>
* Unhooking in C++ - <a href="https://www.ired.team/offensive-security/defense-evasion/how-to-unhook-a-dll-using-c++">https://www.ired.team/offensive-security/defense-evasion/how-to-unhook-a-dll-using-c++</a>
* Playing with OffensiveNim - <a href="https://s3cur3th1ssh1t.github.io/Playing-with-OffensiveNim/">https://s3cur3th1ssh1t.github.io/Playing-with-OffensiveNim/</a>
* OffensiveNim - <a href="https://github.com/byt3bl33d3r/OffensiveNim">https://github.com/byt3bl33d3r/OffensiveNim</a>
* Shellycoat - <a href="https://github.com/slaeryan/AQUARMOURY/tree/master/Shellycoat">https://github.com/slaeryan/AQUARMOURY/tree/master/Shellycoat</a>
* SharpBlock blog post part I - <a href="https://ethicalchaos.dev/2020/05/27/lets-create-an-edr-and-bypass-it-part-1/">https://ethicalchaos.dev/2020/05/27/lets-create-an-edr-and-bypass-it-part-1/</a>
* SharpBlock blog post part II - <a href="https://ethicalchaos.dev/2020/06/14/lets-create-an-edr-and-bypass-it-part-2/">https://ethicalchaos.dev/2020/06/14/lets-create-an-edr-and-bypass-it-part-2/</a>
* SharpBlock - <a href="https://github.com/CCob/SharpBlock">https://github.com/CCob/SharpBlock</a>
* Hellsgate - <a href="https://github.com/am0nsec/HellsGate">https://github.com/am0nsec/HellsGate</a>
* SharpHellsGate - <a href="https://github.com/am0nsec/SharpHellsGate">https://github.com/am0nsec/SharpHellsGate</a>