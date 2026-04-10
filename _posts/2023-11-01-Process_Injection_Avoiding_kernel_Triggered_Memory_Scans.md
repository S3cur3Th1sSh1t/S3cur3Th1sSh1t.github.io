---
title: "Process Injection - Avoiding Kernel Triggered Memory Scans"
layout: "post"
---

A very common technique used by threat actors as well as Red Teams is Process Injection. By using Process Injection, any position-independent code (shellcode) can be written into a remote process and executed within that process, so that it afterward runs in the context of it.

<!--more-->

This blog post was written and published on my employer's website, where it can be found here:

- [https://www.r-tec.net/r-tec-blog-process-injection-avoiding-kernel-triggered-memory-scans.html](https://www.r-tec.net/r-tec-blog-process-injection-avoiding-kernel-triggered-memory-scans.html)

This technique has for example the following typical use cases:

- Impersonate another user by injecting into his process
- Hide Command & Control (C2) traffic within a process, that does regular network connections
- Hide in general by running from within a legitimate system process instead of a non-trusted/new one

The most well-known and obvious approach to do this is via using the following Windows APIs:

<p align="center">
          <img src="/assets/posts/ProcessInjectionMemoryScans/Figure1_Exemplary_Process_Injection_APIs.png">
</p>

*Figure 1: Exemplary Process Injection APIs*

First, a handle to the remote process is retrieved (`OpenProcess`). Virtual Memory (RAM) is allocated within that remote process (`VirtualAllocEx`), to afterwards write the shellcode into that newly allocated memory region (`WriteProcessMemory`). In the last step, this code is executed by for example creating a new thread via `CreateRemoteThread`.

When using known malicious code such as Open Source C2-Framework implants, this combination of APIs however will get you caught very easily in a mature company environment. An EDR running on the endpoint can spot the known malicious code by utilizing Userland-Hooks or ETWti/Kernel Callbacks. Many EDR vendors still make heavy use of Userland-Hooks, but more and more vendors tend to use ETWti only as an alternative due to it being harder to bypass as it's running in Kernelland. This Blog won't go deep into the topic of how Userland-Hooks or ETWti/Kernel callbacks work, as this is already heavily documented in several blog posts. I wrote two personal Blog posts already about bypassing Userland-Hooks with summarizations of existing techniques:

- [https://s3cur3th1ssh1t.github.io/A-tale-of-EDR-bypass-methods/](https://s3cur3th1ssh1t.github.io/A-tale-of-EDR-bypass-methods/)
- [https://s3cur3th1ssh1t.github.io/Cat_Mouse_or_Chess/](https://s3cur3th1ssh1t.github.io/Cat_Mouse_or_Chess/)

The second Blog post also explains a new approach including a Proof of Concept tool release [Ruy Lopez](https://github.com/S3cur3Th1sSh1t/Ruy-Lopez).

Instead, this Blog will show a novel way to avoid detections for Process Injection triggered by ETWti from Kernel.

## 1. Avoiding detections coming from Kernelland

Kernel Callbacks can be used, to intercept/interact execution of processes on start or runtime. From my perspective, it is somehow imaginable like hooks with the difference that they are triggered after Syscall executions from within the kernel. There are different Kernel Callbacks that be used to check for potential malicious activity such as `PsSetCreateProcessNotifyRoutine()` to know about newly spawned processes and to check those before they get started or `PsSetCreateThreadNotifyRoutine()` to check newly created Threads and their entrypoint for known malicious code. If some malicious behavior is found/verified, the process can be killed for example and an alert is raised.

ETWti is an interface provided by Microsoft, where drivers can subscribe to receive special ETW events. These events are specifically meant to be used for detecting malicious activities and also include events such as Process creation, Allocation of memory, Thread creation, and much more.

So our shown Process Injection API example could easily get detected via Kernel Callbacks or ETWti, even if we were using (In)-Direct Syscalls, Stack Spoofing, and whatever else techniques to avoid detections. This is because, in the moment of creating a new Thread in a remote process, the EDR can trigger a Yara-rule memory scan of the remote process memory sections and or Thread entrypoint to spot the known malicious code.

In many cases, however, not only the execute primitive (Thread creation/continue, Queuing an APC, [...]) leads to a detection but the whole combination of events:

<p align="center">
          <img src="/assets/posts/ProcessInjectionMemoryScans/Figure2_Suspicious_combination_Process_Injection.png">
</p>

*Figure 2: Suspicious combination for Process Injection*

The memory scan is a very important piece of this puzzle, as vendors tend to **verify** malicious behavior before preventing execution to not have too many false positive events. There are applications doing code Injection or hooking for legitimate Use-Cases and only blocking execution due to a specific combination of APIs could interrupt production environments, which is not desired. Imagine a company buys an EDR software, which constantly alerts on legitimate processes or kills legitimate processes. This company would want to get rid of the software as fast as possible. So verification actually makes perfect sense and using a memory scan to check for known malicious code is a good way to do so.

How does it look like when you got flagged by behaviour or a memory scan? The detection nowadays typically also includes the name of the known malicious implant like that:

<p align="center">
          <img src="/assets/posts/ProcessInjectionMemoryScans/Figure3_Cobalt_Strike_specific_detection.png">
</p>

*Figure 3: Cobalt Strike specific detection*

<p align="center">
          <img src="/assets/posts/ProcessInjectionMemoryScans/Figure4_Havoc_specific_detection.png">
</p>

*Figure 4: Havoc specific detection*

One way to get rid of such detections would be to obfuscate/change the code base (Open Source only) or to use self-written C2 Frameworks. This way, the Yara memory scan would not find a matching malicious code anymore. But of course, this may include a lot of resources/effort.

## 2. Introducing Caro-Kann

After getting my hands dirty with self-written PIC-Code for the first time in developing Ruy-Lopez, a new world of ideas opened up in terms of malware development. You get full control of execution which led to another EDR bypass idea from my side. The idea for Caro-Kann was as follows:

<p align="center">
          <img src="/assets/posts/ProcessInjectionMemoryScans/Figure5_Caro-Kann_workflow.png">
</p>

*Figure 5: Caro-Kann workflow*

The Malware in this case injects two, instead of the one known malicious shellcode. The known malicious shellcode in this case is still encrypted and written into a `READ_WRITE` protected memory page. A second custom self-written shellcode is injected into a small `READ_EXECUTE` memory section and the execute primitive points to this code instead of the actual implant code. As we do not prevent ETWti events or Kernel Callbacks from occurring at all, the typical memory scan for verification is triggered. The custom shellcode Sleeps for an amount of x seconds and therefore does nothing suspicious at all. In this time slot, the memory scan will not find any known malicious code, as `READ_WRITE` protected pages may not have been scanned but even if so – the payload is still encrypted and therefore not found. The good thing here is, that this technique does not need any elevated privileges (such as the Driver exploitation tools, which were hyped in the last months), as it completely runs from Userland. Therefore it is also suitable for initial access payloads.

<p align="center">
          <img src="/assets/posts/ProcessInjectionMemoryScans/Figure6_Custom_shellcode_Workflow.png">
</p>

*Figure 6: Custom shellcode Workflow*

After Sleep, the `READ_WRITE` encrypted payload is decrypted and the memory protection is changed to `READ_EXECUTE` so that it can get executed. Execution takes place with a direct JMP instruction, as this does not lead to any ETWti events being generated and does not trigger any Kernel Callback.

You might wonder about how the custom shellcode knows about which memory address to decrypt and which shellcode length to use. In the published PoC I used an egg-hunter in the host process doing the injection. This process knows about the memory address and shellcode length and can therefore overwrite the eggs in the custom shellcode:

<p align="center">
          <img src="/assets/posts/ProcessInjectionMemoryScans/Figure7_Egg-hunter_output_from_injector.png">
</p>

*Figure 7: Egg-hunter output from the injector*

Caro-Kann can be found here:

- [https://github.com/S3cur3Th1sSh1t/Caro-Kann](https://github.com/S3cur3Th1sSh1t/Caro-Kann)

**Difference to existing tooling:**

This technique could somehow be compared to other long-time existing techniques. Shellcode encoders like [Shikata ga nai (SGN)](https://github.com/EgeBalci/sgn) also encrypt shellcode and decrypt it on runtime. If you encode your shellcode multiple hundreds or thousands of times via it, there is also a time delay which could prevent memory scans from finding the known malicious payload. But I made some tests against EDRs with it – leading to detections all of the time. So I'm **assuming** that at least some EDR vendors can **debug/unpack** SGN payloads already, which makes this alternative less effective.

## 3. OPSEC Improvements

Even though the public Proof of Concept is using the initially mentioned "unsafe" Win32 API combination, it bypasses both Userland-Hook detections and the memory scan trigger from Kernel due to encryption & Sleep. But this doesn't mean you won't pop an alert in a mature environment using this technique. As ETWti events are still logged and evaluated, the Process Injection behavior itself can generate Alerts by EDR vendors like this one:

<p align="center">
          <img src="/assets/posts/ProcessInjectionMemoryScans/Figure8_Suspicious_Process_Injection_Alert.png">
</p>

*Figure 8: Suspicious Process Injection Alert*

This is a Medium Alert, which is generated out of the following combination of events:

- Allocation of new memory in the remote process (`VirtualAllocEx` / `NtAllocateVirtualMemory`)
- Injection (`WriteProcessMemory` / `NtWriteVirtualMemory`) into the remote process
- Thread creation (`CreateRemoteThread` / `NtCreateThreadEx`)

As long, as you're using this combination of APIs – this Alert will get generated. This is by the way the same Alert you will see when using Direct Syscalls, as those also do not bypass ETWti events:

<p align="center">
          <img src="/assets/posts/ProcessInjectionMemoryScans/Figure9_Direct_Syscall_Injection_Alert.png">
</p>

*Figure 9: Direct Syscall Injection Alert*

To get rid of such detections, you will therefore need to use alternatives for memory allocation or shellcode execution. For memory allocation, a non-exhaustive list of alternatives could be the following:

<p align="center">
          <img src="/assets/posts/ProcessInjectionMemoryScans/Figure10_Find_Existing_RWX_Sections_SetWindowsHookEx.png">
</p>

*Figure 10: Find Existing RWX Sections & SetWindowsHookEx*

References for finding existing `RWX` sections or `SetWindowsHookEx` are e.g. the following:

- [https://www.ired.team/offensive-security/defense-evasion/finding-all-rwx-protected-memory-regions](https://www.ired.team/offensive-security/defense-evasion/finding-all-rwx-protected-memory-regions)
- [https://twitter.com/MrUn1k0d3r/status/1627678626584883200](https://twitter.com/MrUn1k0d3r/status/1627678626584883200)

By for example overwriting existing memory, you can get rid of memory allocation ETWti events. This could be the `.text` section of a DLL ([Module Stomping](https://blog.f-secure.com/hiding-malicious-code-with-module-stomping/)) or other existing `RWX` regions (for example [Mockingjay](https://www.securityjoes.com/post/process-mockingjay-echoing-rwx-in-userland-to-achieve-code-execution)).

For the execute primitive, two public non-ETWti event alternatives would be [ThreadlessInject](https://github.com/CCob/ThreadlessInject) or [DllNotificationInjection](https://github.com/ShorSec/DllNotificationInjection):

<p align="center">
          <img src="/assets/posts/ProcessInjectionMemoryScans/Figure11_Execute_primitive_alternatives.png">
</p>

*Figure 11: Execute primitive alternatives*

Some alternatives for the Inject/Write primitive were also outlined by [@x86matthew](https://twitter.com/x86matthew) in his blog posts:

- [https://www.x86matthew.com/view_post?id=writeprocessmemory_apc](https://www.x86matthew.com/view_post?id=writeprocessmemory_apc)
- [https://www.x86matthew.com/view_post?id=proc_env_injection](https://www.x86matthew.com/view_post?id=proc_env_injection)

Both alternatives however create Threads in the remote process with corresponding ETWti events and the second could "only" be implemented for Spawn/Inject loaders.

You could also get rid of generic Process Injection Alerts by changing `VirtualAllocEx` to Module Stomping and `CreateRemoteThread` to ThreadlessInject techniques. There is a public implementation of both techniques combined already by [@OtterHacker](https://twitter.com/OtterHacker):

- [https://github.com/OtterHacker/Conferences/tree/main/Defcon31](https://github.com/OtterHacker/Conferences/tree/main/Defcon31)

**C2-Considerations:**

Caro-Kann is there to bypass the initial Process Injection-related memory scan. But as EDRs might also do memory scans after observing specific behaviors or just regularly in an interval of x minutes for all processes you might still get detected later on. To avoid such detections you should prefer to use C2-Frameworks with built-in sleep encryption (Such as [Ekko](https://github.com/Cracked5pider/Ekko)) or as an alternative add your own Sleep encryption into them.

## 4. Closing words

More and more EDRs tend to trigger detections from Kernel which cannot easily get bypassed from Userland anymore. Especially Process Injection without getting detected got much harder so many Operators just don't use this anymore for not getting detected.

But most vendors (for good reasons) tend to **verify** known malicious before preventing execution via memory scans. By hiding the known malicious at the time of the execute primitive it's therefore enough to still get a known malicious implant to run via Process Injection. The Proof of Concept Caro-Kann demonstrates that by injecting a second custom shellcode, which sleeps, decrypts the known malicious on runtime and jumps to it.

The published PoC is not OPSec safe, as it will probably still generate generic Injection alerts. But by leaving out APIs that generate ETWti events (such as memory allocation and Thread creation) from the chain, these can also get bypassed. Exercise for the reader. ;-)

As only the initial memory scan gets bypassed via the published technique, you should consider using C2-Frameworks with Sleep encryption or implementing it into existing Frameworks yourself, as otherwise you might get caught by regular interval scans.

---
*This blog post was originally published on [r-tec.net](https://www.r-tec.net/r-tec-blog-process-injection-avoiding-kernel-triggered-memory-scans.html).*