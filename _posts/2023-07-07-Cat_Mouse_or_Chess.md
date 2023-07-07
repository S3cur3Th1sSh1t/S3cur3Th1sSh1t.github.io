---
title: "Cat & Mouse - or Chess?"
layout: "post"
---

Last year I had the idea for a new approach to block EDR DLLs from loading into a newly spawned process. After several months this idea lead to a PoC, which was then published after presenting the topic at [x33fcon](https://www.x33fcon.com/#!s/FabianMosch.md) and [Troopers](https://troopers.de/troopers23/talks/juxtj7/) this year.

This post will cover the background and description of the technique.

<!--more-->


## How do EDRs typically detect malicious activities?

Endpoint detection and Response (EDR) systems detect malicious activities or software in various ways. The detections can occur from userland (where the user processes run) or from kernelland (operating system level).

Typical analysis/detections from userland include:
- Static & dynamic analysis
- Userland hooking
- Stack trace analysis

Whereas static analysis can be for example signatures for files (like any AV uses) or checking metadata such as certificates and their validity. Dynamic analysis can include active debugging of an executable, or putting it into a sandbox-like environment to see what it does on runtime.

Stack trace analysis can show, if an process was for example executing specific Windows APIs from an unbacked memory region (dynamic code in a private commit memory section, very likely shellcode) which is super suspicious.

Detection coming from kernelland typically make use of:
- Kernel Callbacks
- ETW Threat Intelligence (ETWti)

EDRs typically use a signed driver to also operate from kernelland. By doing this, they can check specific Kernel Callbacks for any running process to live intercept execution for those and execute their own code before the process resumes with whatever it will do afterward. If for example a new process is created, the EDR can intercept its execution and check what has to be executed with the Kernel Callback `PsSetCreateProcessNotifyRoutine()`. But they could also live intercept the creation of new threads with `PsSetCreateThreadNotifyRoutine()` to check their entrypoint for malicious code. As the code is running from the kernel, any mistake could lead to a system-wide bluescreen, which could be one reason for vendors to not make heavy use of it.

<p align="center">
          <img src="/assets/posts/RuyLopez/KernelCallbacks.png">
</p>

ETWti is an interface provided by Microsoft, where drivers can subscribe to receive special ETW events. These events are specifically meant to be used for detecting malicious activities and also include events such as for Process creation, Allocation of memory, Thread creation and much more:

<p align="center">
          <img src="/assets/posts/RuyLopez/ETWti.png">
</p>

For this blog post and technique, we will however focus on userland hook-based detections, as those at least from my experience are still the most relevant detections being mainly used by nearly all EDRs.

## What are userland hooks about?

For being able to do live analysis of userland processes, EDR vendors and most AV vendors as well load their own Dynamic Linked Library (DLL) into running processes on an operating system. After this DLL is loaded in a process, it will patch memory regions of chosen Windows APIs to place a hook, which is basically an JMP instruction going to their own DLLs memory region.

<p align="center">
          <img src="/assets/posts/RuyLopez/Hooks.png">
</p>

Via this, they can live intercept Windows APIs from being called on runtime of a process and inspect the input arguments for the Windows API being executed to check what it *wants to* do on runtime.

Let us imagine the following process for malware to execute shellcode in a remote process for a better understanding:

<p align="center">
          <img src="/assets/posts/RuyLopez/ProcInject.png">
</p>

First, the malware gets a Handle to the remote process by using `OpenProcess`. Afterwards it calls `VirtualAllocEx` to allocate memory in the remote process. `WriteProcessMemory` is used to actually write the Shellcode into the remote process newly allocated memory region. In the end, `CreateRemoteThread` is used to execute the Shellcode in the remote process in a newly created Thread. We also imagine, that the Shellcode was encrypted and decrypted on runtime before writing it into the remote process with `WriteProcessMemory` to avoid signature based detections. `CreateRemoteThread` will after being called itself call `NtCreateThreadEx` from `ntdll.dll`, which is the last function being called from userland. As `ntdll.dll` functions are the last ones being called from userland, many vendors tend to hook functions from this specific DLL.

An example definition for `NtCreateThreadEx` looks like the following:

<p align="center">
          <img src="/assets/posts/RuyLopez/Definition.png">
</p>

The EDR can inspect input arguments when hooked APIs are called on runtime, right? So in this case, the EDR could inspect the input parameters for `NtCreateThreadEx` and especially the `startAddress` input pointer. When a malware wants to start Shellcode in a new Thread, the `startAddress` will typically point to the already **decrypted** plain Shellcode, e.G. an C2 implant. Our EDR can now apply Yara-Rules (memory scan) on the `startAddress` memory region to find any **known malicious** C2 implants, such as CobaltStrike, Sliver, Covenant and so on. And if a rule matches, they know that malicious software wants to be called and therefore just kill the process.

So in the very end our malware has been stopped by the EDR on runtime, based on userland hook-based detections + verification of **known malicious** code.

## Existing Userland hook evasion techniques

Several different tools and techniques have been released over the last years, which are capable of bypassing userland hook-based detections. I will give a summary here, but won't go into depth as there are many other articles to read about or code to check in the links.

1. Unhooking
2. The usage of direct Syscalls
3. Using Hardware Breakpoints
4. Patching the DLL entrypoint

#### Unhooking

With unhooking, a fresh copy of `ntdll.dll` is grabbed from any location (Disk, KnownDlls) and the *EDR patched* memory region with jumps to the EDR DLL is replaced with the original value of `ntdll.dll`. This effectively bypasses hooks, as no more jumps will take place and no more input argument analysis is done.

<p align="center">
          <img src="/assets/posts/RuyLopez/Unhook.png">
</p>

#### Direct Syscalls

Instead of calling `ntdll.dll` functions the regular way, their content can also be retrieved or re-build on runtime and executed directly from the current process memory. If we have our own `ntdll.dll` functions in our own process memory, there will also be no hooks in place, as those are only in the `ntdll.dll` memory location, which also leads to a bypass here. Proof of Concepts for different retrieval techniques are for example the following:

1. Re-build `ntdll.dll` functions with information from Memory ([HellsGate](https://github.com/am0nsec/HellsGate), [RecycledGate](https://github.com/thefLink/RecycledGate),*-Gate)
2. Get a fresh `ntdll.dll` copy from Disk and put the function content to memory (GetSyscallStub, e.G. [C DInvoke](https://github.com/TheWover/DInvoke))
3. The Syscall Stubs are partially or completely embedded in the malware executable from the beginning - (Syswhispers [1](https://github.com/jthuraisamy/SysWhispers
),[2](https://github.com/jthuraisamy/SysWhispers2),[3](https://github.com/klezVirus/SysWhispers3))

#### Usage of Hardware Breakpoints

The first PoC I know about which was using Hardware Breakpoints to evade userland hooks was [TamperingSyscalls](https://github.com/rad9800/TamperingSyscalls). The process looks like the following:

<p align="center">
          <img src="/assets/posts/RuyLopez/HardwareBreakpoints.png">
</p>

By placing Hardware Breakpoints to the `ntdll.dll` function we want to call afterward, we can effectively intercept execution before the jump to the EDR DLL takes place. In that moment, we hide the input parameters for this specific function from the stack by replacing it with arbitrary values and we back the original values up in another location. Afterwards, execution is resumed and the EDR hook **will take place**. But the EDR won't see the original input parameters and therefore cannot detect/verify any malicious activities.

The first original PoC was then single stepping forward, till the Syscall instruction itself would be called. But before actually calling it, the original input parameters will be restored to the stack, so that the function will work as expected.

#### Patching the DLL entrypoint

[]() released a [blog Post](https://ethicalchaos.dev/2020/06/14/lets-create-an-edr-and-bypass-it-part-2/) and the tool [SharpBlock]() in 2020. This technique works as follows:
1. Create a new Process with the `DEBUG_ONLY_THIS_PROCESS` flag. This leads to the parent process being capable of acting as a debugger for the new process. As Debugger, we can intercept execution for specific events and execute code before resuming execution.
2. As Debugger, the parent process waits for `LOAD_DLL_DEBUG_EVENT` events, which appear **after** DLLs were loaded into a process but **before** something out of them was executed.
3. The parent process checks the DLL being loaded. If it was the EDR DLL, it will patch the Entrypoint from it with `0x3c - return`, so that the DLL will afterward instead of placing hooks just return and exit.

<p align="center">
          <img src="/assets/posts/RuyLopez/SharpBlock.png">
</p>

Effectively, no DLL is there anymore and no hooks are placed in the new process.

## The Idea for a new approach

I had the idea for a new approach after reading the following Blogpost by [Alejandro Pinna](https://twitter.com/frodosobon):

- https://waawaa.github.io/es/amsi_bypass-hooking-NtCreateSection/

The technique described here is capable of preventing DLLs from being loaded into a process by placing hooks ourself to API functions being involved into the process of loading a DLL and mapping it into memory. It can be used to e.G. prevent `amsi.dll` from loading into the current process before calling `assembly::load()`, so that it leads to an AMSI bypass.

In more detail the exemplary function `NtCreateSection` was hooked, so that when the target DLL memory section wants to get created we can intercept this process and return NTSTATUS fail. If `NtCreateSection` fails, the mapping to memory also cannot be done which in the very end leads to the DLL not getting loaded at all.

But this approach can only be applied to DLLs, that **were not loaded yet** in the current process.

#### The problem with EDR DLLs

EDRs basically act like the white player in a [chess game](https://bruteratel.com/release/2022/08/18/Release-Scandinavian-Defense/). They can after receiving a Kernel Callback for a new process getting created **directly** load their own DLL into the process memory. So in a good implementation (some vendors don't do that), the DLL will get loaded **instantly** after `ntdll.dll`. And it will get loaded into basically **any** userland process (besides you may find some [exceptions](my own tweet regarding to finding exceptions)).

This leads to the fact, that we cannot hook `NtCreateSection` to prevent EDR DLLs from being loaded, as they are always loaded already in our own process.

#### Solvable?

But - if we create a new suspended process, only `ntdll.dll` will be loaded:


```c
tProcPath = newWideCString(r"C:\windows\system32\windowspowershell\v1.0\powershell.exe")

status = CreateProcess(
    NULL,
    cast[LPWSTR](tProcPath),
    ps,
    ts, 
    FALSE,
    CREATE_SUSPENDED or CREATE_NEW_CONSOLE or EXTENDED_STARTUPINFO_PRESENT,
    NULL,
    r"C:\Windows\system32\",
    addr si.StartupInfo,
    addr pi)
```

<p align="center">
          <img src="/assets/posts/RuyLopez/Suspended.png">
</p>

So my idea was basically (setzt genau an dieser Stelle an). `ntdll.dll` is already loaded, `NtCreateSection` is contained there. We can therefore place a hook into our newly created suspended process and write custom self written Shellcode into it, which handles this hook to prevent specific DLLs from being loaded/mapped into its memory. All we need to do is the following:

1. Create a suspended process
2. Allocate memory for our custom Shellcode and write it into that region
3. Remotely place a hook on `NtCreateSection` which jumps to our Shellcode
4. Resume the process
5. We are in control for DLL-loads

<p align="center">
          <img src="/assets/posts/RuyLopez/RuyLopez.png">
</p>

Sounds simple? Well, as least for me in the actual implementation it wasnt.

## Challenges in the implementation

#### PIC Code

We have to write custom Shellcode. I personally never did that before in a project, so this was the first time. To write PIC-Code, we need to take care of some things:
- Everything should only exist in the `.text` section of the compiled executable. That is the dynamic (position independent) part of an executable, which we can extract to have PIC-Code.
- We cannot use any global variables anymore (they are not placed in the `.text` section) - we therefore have to find alternatives to exchange information in between different functions
- All Windows APIs being called need to be resolved dynamically
- The `mainCRTStartup` routine needs to be replaced with our Entrypoint for proper execution
- This is specific to **this** PoC only: We can only use `ntdll.dll` functions, because when our hook is hit the process is not even fully initialized yet. We also cannot load other DLLs at that moment, it will lead to a process crash. Alternatively, we could wait for process initialization to finish and only apply our logic from that moment on, but I didnt implement it like this myself.
- Many comfortable functions, which you may want to use are not usable for writing PIC-Code. Such as `charcmp`, `StrStrIA`, `strlen`, `memcpy` and more. So in my case, I manually coded the logic for these functions, or Github Copilot did after writing a comment with what I wanted to achieve ;-)
- I learned, that debugging PIC-Code is a pain. Especially when only being capable of using `ntdll.dll` functions. So I ended up writing a custom logger function, which can give some information about what happened in case of troubleshooting.

#### Original NtCreateSection value

As stated earlier, our technique will hook `NtCreateSection` in the new process. So when this process is resumed, the original value does not exist anymore in its process memory. But as we don't want to prevent each and every `NtCreateSection` call, we need to restore the original value at some point to still be able of creating Sections for other DLLs or object Handles. In the very end its a Syscall Stub, so you could use any existing technique for direct Syscall retrieval and execution such as [TartarusGate]() or [GetSyscallStub]() or whatever, but only by using `ntdll.dll` functions. For my initial PoC however, I decided to use an egghunter in the host process with an egg placed in the Shellcode. The original host process can retrieve the original `NtCreateSection` value before placing the hook and replace the egg in the Shellcode with that original value, so that the Shellcode itself can restore the original value on runtime.

```c
void originalBytes() { // used to store the original bytes of the function we are hooking. This function can be used in PIC to exchange information between functions, as global variables cannot be used. Thanks @Mr-Un1k0d3r for the hint.
    asm(".byte 0xDE, 0xAD, 0xBE, 0xEF, 0x13, 0x37, 0xDE, 0xAD, 0xBE, 0xEF, 0x13, 0x37, 0xDE, 0xAD, 0xBE, 0xEF, 0x13, 0x37, 0xDE, 0xAD, 0xBE, 0xEF, 0x13, 0x37 ");
}
```

<p align="center">
          <img src="/assets/posts/RuyLopez/EggHunt.png">
</p>

#### Stack Alignment?

If you take a look at existing PIC-Code implementations on Github, such as [Handlekatz](https://github.com/codewhitesec/HandleKatz), or [blog posts](https://bruteratel.com/research/feature-update/2021/01/30/OBJEXEC/), you will notice, that these have a small embedded ASM-Stub which looks like this:

```c
extern entryFunction
global alignstack

segment .text

alignstack:
    push rdi                    ; backup rdi since we will be using this as our main register
    mov rdi, rsp                ; save stack pointer to rdi
    and rsp, byte -0x10         ; align stack with 16 bytes
    sub rsp, byte +0x20         ; allocate some space for our C function
    call entryFunction          ; call the C function
    mov rsp, rdi                ; restore stack pointer
    pop rdi                     ; restore rdi
    ret                         ; return where we left
```

This is typically used to align the stack for 16-byte code. But this code cannot be used in our implementation, as it modifies the stack. And modifications to the stack lead to corrupting input arguments of `NtCreateSection`. When our Shellcode is called, we therefore need to not modify the stack at all but instead directly jump to our entrypoint function which handles the hook:

<p align="center">
          <img src="/assets/posts/RuyLopez/DirectJump.png">
</p>

Good news, we also don't need to align the stack at all in this case, as that is already done by the function calling `NtCreateSection`.

#### Choosing the correct NTSTATUS return value

Each process/software handles calls to `NtCreateSection`differently and also behaves differently depending on the returned NTSTATUS call. Some may just ignore the value, others will try to repeat the call infinitely, and so on. When fiddling around with an initial PoC to prevent `amsi.dll` from being loaded into a newly spawned Powershell process for example, I noticed, that when returning `NTSTATUS 0`, Powershell will crash. This is due to the Windows OS and or Microsoft believing, the Section was created successfully and therefore just continuing execution as normal and not handling the resulting crash. But when passing an error like `0xC0000054 - STATUS_FILE_LOCK_CONFLICT`, this error will get handled correctly and `amsi.dll` will not get loaded but Powershell will still start.

However, when using an error code for different executables or EDR DLLs, you may see an GUI error message with your STATUS call. As long as no one presses okay, execution is not resumed. I personally had the best experience with still returning NTSTATUS success for blocking EDR DLLs, but if you face some problems try fiddling around with that value.

## The Proof of Concept - Ruy Lopez

You might remember the EDR typically acts as the white player in a chess game? Well, with the approach from this post we as malware act like the white player, because we are even in the remote process **before** the EDR DLL is loaded and can prevent it from being loaded. Therefore I decided to name the initial Proof of Concept Ruy Lopez, a white player starting technique from chess. :-)

As mentioned in the last chapter already, the published Proof of Concept will spawn a new Powershell in suspended mode, write the Shellcode into that suspended process and place the hook and afterwards resume it. But as its a Proof of Concept, it will **only** block/prevent `amsi.dll` from being loaded effectively also leading to an AMSI bypass. If you want to block EDR DLLs, you have to modify my `HookForward` Code to do so, homework for the reader is always good for leaning purposes.

When I did test the technique against different EDR vendors some months ago (before publicly releasing it), it was neither alerted or prevented by any of them. In one specific case an vendor was injecting an DLL instead of loading it the regular way - if this is the case my PoC cannot block it. In another test, one specific DLL out of five different from that vendor did lead to process crashes when blocking it - luckily it was not the one placing hooks. So in any case, the PoC may need modifications per vendor or being adjusted depending on some special cases.

The final PoC can be found here:

- https://github.com/S3cur3Th1sSh1t/Ruy-Lopez

#### Is that OPSec Safe?

Well. Using Injection and hooking in general has well documented easy to spot Indicators of Compromise (IoCs) in any case. So if a Blue Team or Hunter/Analyst reviews the processes being involved, it will be easy to spot this IoCs and find out something malicious happened. However, till now I didnt face automated detections alerting on or preventing this technique. EDR vendors could for example also integrate checks, that if a suspended process was resumed and if some `ntdll.dll` function was hooked at that point kill the process. I guess, there are very few or maybe even no false positives at all for such a situation.

#### OPSec improvements

The first PoC was using Win32 APIs for Injection and for placing the hook. I changed this already to direct Syscalls after giving the Troopers talk. Whats left?

The Shellcoe itself needs to have `RWX` permissions in the published version, due to doing some self modifications. But this is not nessesarily needed and relatively easy to adjust, I placed some comments into the code on what to change for the ones interested. By modifying it, you can also use `RX` permissions.

Instead of using plain DLL-names to block (they appear as string in the shellcode itself), you could use Hashing and compare against that to avoid signature based detections for the Shellcode.

To get rid of hooking based IoCs, you could use Hardware Breakpoints instead. 

#### Alternative usage ideas

- Blocking `wldp.dll` to bypass Device Guard / trust checks
- Block custom AMSI Provider DLLs
- Inject/Execute shellcode ThreadlessInject style in the new process. The cool thing here is, our Shellcode is always automatically executed after resuming the process when any DLL is loaded but also for process initialization at least once. So we never need an **execute** primitive here or place a hook as ThreadlessInject does, but instead we could just decrypt embedded Shellcode and execute that.

## Credits

Some people helped me out on my way. Have to give some Credits.

- Ceri Coburn [@_EthicalChaos_](https://twitter.com/_EthicalChaos_) - Q/A all over the way
- Sven Rath [@eversinc33](https://twitter.com/eversinc33) - Inspired me to actually start writing the PoC instead of just having this in mind only
- Alejandro Pinna [@frodosobon](https://twitter.com/frodosobon) - The initial Blogpost my technique relies on
- Charles Hamilton [@MrUn1k0d3r](https://twitter.com/MrUn1k0d3r) - Q/A for writing PIC-Code
- Chetan Nayak [@NinjaParanoid](https://twitter.com/NinjaParanoid) - Q/A for writing PIC-Code

## Links & Resources

* Hellsgate [https://github.com/am0nsec/HellsGate](https://github.com/am0nsec/HellsGate)
* RecycledGate [https://github.com/thefLink/RecycledGate](https://github.com/thefLink/RecycledGate)
* DInvoke [https://github.com/TheWover/DInvoke](https://github.com/TheWover/DInvoke)
* Syswhispers [https://github.com/jthuraisamy/SysWhispers](https://github.com/jthuraisamy/SysWhispers)
* Syswhispers2 [https://github.com/jthuraisamy/SysWhispers2](https://github.com/jthuraisamy/SysWhispers2)
* Syswhispers3 [https://github.com/klezVirus/SysWhispers3](https://github.com/klezVirus/SysWhispers3)    
* TamperingSyscalls [https://github.com/rad9800/TamperingSyscalls](https://github.com/rad9800/TamperingSyscalls)
* SharpBlock Blog post [https://ethicalchaos.dev/2020/06/14/lets-create-an-edr-and-bypass-it-part-2/](https://ethicalchaos.dev/2020/06/14/lets-create-an-edr-and-bypass-it-part-2/)
* Chess White player idea [https://bruteratel.com/release/2022/08/18/Release-Scandinavian-Defense/](https://bruteratel.com/release/2022/08/18/Release-Scandinavian-Defense/)
* HandleKatz [https://github.com/codewhitesec/HandleKatz](https://github.com/codewhitesec/HandleKatz)
