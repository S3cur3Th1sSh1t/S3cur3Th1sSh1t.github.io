---
title: "Named Pipe Pass-the-Hash"
layout: "post"
---

This post will cover a little project I did last week and is about Named pipe Impersonation in combination with Pass-the-Hash (PTH) to execute binaries as another user. Both techniques used are not new and often used, the `only` thing I did here is combination and modification of existing tools. The current public tools all use PTH for network authentication only. The difference to this "new" technique is therefore, that you can also spawn a new shell or C2-Stager as the PTH user for local actions `and` network authentication. 

<!--more-->

## Introduction - why another PTH tool?

I faced certain Offensive Security project situations in the past, where I already had the NTLM-Hash of a `low privileged` user account and needed a shell for that user on the current compromised system - but that was not possible with the current public tools. Imagine two more facts for a situation like that - the NTLM Hash could not be cracked *and* there is no process of the victim user to execute shellcode in it or to migrate into that process. This may sound like an absurd edge-case for some of you. I still experienced that multiple times. Not only in one engagement I spend a lot of time searching for the right tool/technique in that specific situation. Last week, <a href="https://twitter.com/n00py1">@n00py1</a> tweeted exactly the question I had in mind in those projects: 

<p align="center">
          <img src="/assets/posts/NamedPipePTH/N00py.JPG">
</p>

So I thought: Other people in the field obviously have the same limitations in existing tools.

My personal goals for a tool/technique were:

* Fully featured shell or C2-connection as the victim user-account
* It must to able to also Impersonate `low privileged` accounts - depending on engagement goals it might be needed to access a system with a specific user such as the CEO, HR-accounts, SAP-administrators or others
* The tool has to be used on a fully compromised system without another for example linux box under control in the network, so that it can be used as C2-module for example

The Tweet above therefore inspired me, to again search for existing tools/techniques. There are plenty of tools for network authentication via Pass-the-Hash. Most of them have the primary goal of code execution on remote systems - which needs a privileged users Hash. Some of those are:

* SMB (<a href="https://github.com/byt3bl33d3r/CrackMapExec">CrackMapExec</a>, <a href="https://github.com/SecureAuthCorp/impacket/blob/master/examples/smbexec.py">smbexec.py</a>, <a href="https://github.com/Kevin-Robertson/Invoke-TheHash/blob/master/Invoke-SMBExec.ps1">Invoke-SMBExec.ps1</a>)
* WMI (<a href="https://github.com/Kevin-Robertson/Invoke-TheHash/blob/master/Invoke-WMIExec.ps1">Invoke-WMIExec.ps1</a>, <a href="https://github.com/SecureAuthCorp/impacket/blob/master/examples/wmiexec.py">wmiexec.py</a>)
* DCOM (<a href="https://github.com/SecureAuthCorp/impacket/blob/master/examples/dcomexec.py">dcomexec.py</a>)
* WinRM (<a href="https://github.com/Hackplayers/evil-winrm">evil-winrm</a>)

If we want to have access to an administrative account and a shell for that account, we can easily use the WMI, DCOM and WinRM PTH-tools, as commands are executed in the users context. The python tools could be executed over a SOCKS tunnel via C2 for example, the Powershell scripts work out-of-the-box locally. SMB PTH tools execute commands as `nt-authority\system`, so user impersonation is not possible here. One of my personal goals was not fulfilled - the impersonation of `low privileged` accounts. So I had to search for more possibilities. 

The best results for *local* PTH actions are in my opinion indeed <a href="https://github.com/gentilkiwi/mimikatz">Mimikatz</a>'s `sekurlsa::pth` and <a href="https://github.com/GhostPack/Rubeus">Rubeus</a>'s `PTT` features. I tested them again to start software via PTH or inject a Kerberos ticket into existing processes and realized, that they `only` provide network authentication for the PTH-user. Network authentication *Only*? Ok, I have to admit, in the most cases network authentication is enough. You can read/write the Active Directory via LDAP, access network shares via SMB, execute code on remote systems with a privileged user (SMB, WMI, DCOM, WinRM) and so on. But still - the `edge case` to start an application as the other user via Pass-the-Hash is not possible. I thought to myself, that it might be possible to modify one of those tools to archieve the specific goal of an interactive shell. To do that, I had to first dig into the code to understand it. Modifying Rubeus was no opion for me, because `PTT` uses a Kerberos ticket, which is as far as I know only used for network authentication. That won't help us authenticating on the localhost for a shell. So I took a look at the Mimikatz feature in the next step. 

## Mimikatz's sekurlsa::pth feature

This part will only give some background information to the `sekurlsa::pth` Mimikatz module. If you already know about it feel free to skip. Searching for `sekurlsa::pth` internals resulted in two good blog posts for me, which I recommend reading for a deeper look into the topic, as I will only explain the high-level process:

* <a href="https://www.praetorian.com/blog/inside-mimikatz-part1/">https://www.praetorian.com/blog/inside-mimikatz-part1/</a>
* <a href="https://www.praetorian.com/blog/inside-mimikatz-part2/">https://www.praetorian.com/blog/inside-mimikatz-part2/</a>

A really short high-level overview of the process is as follows:

* MSV1_0 and Kerberos are Windows two Authentication providers, which handle authentication using provided credential material 
* The LSASS process on a Windows Operating System contains a structure with MSV1_0 and Kerberos credential material
* Mimikatz `sekurlsa::pth` creates a new process with a dummy password for the PTH user. The process is first created in the SUSPENDED state
* Afterwards it creates a new MSV and Kerberos structure with the user provided NTLM hash and overwrites the original structure for the given user
* The newly created process is RESUMED, so that the specified binary like for example `cmd.exe` is executed

---
**This part is copy & paste from the part II blog:**
Overwriting these structures does not change the security information or user information for the local user account. The credentials stored in LSASS are associated with the logon session used for network authentication and not for identifying the local user account associated with a process.

---

Those of you, who read my other blog posts know, that C/C++ is not my favorite language. Therefore I decided to work with <a href="https://twitter.com/b4rtik">@b4rtik's</a> <a href="https://github.com/b4rtik/SharpKatz/">SharpKatz</a> code, which is a C# port of the in my opinion most important and most used Mimikatz functions. Normally, I don't like blog posts explaining a topic with code. Don't ask me why, but this time I did it myself here. The <a href="https://github.com/b4rtik/SharpKatz/blob/master/SharpKatz/Module/Pth.cs">PTH module</a> first creates a structure for the credential material called `data` from the class `SEKURLSA_PTH_DATA`:

<p align="center">
          <img src="/assets/posts/NamedPipePTH/SEKURLSA_PTH_DATA.JPG">
</p>

The NtlmHash of this new structure is filled with our given Hash:

```batch
                if (!string.IsNullOrEmpty(rc4))
                    ntlmHashbytes = Utility.StringToByteArray(rc4);

                if (!string.IsNullOrEmpty(ntlmHash))
                    ntlmHashbytes = Utility.StringToByteArray(ntlmHash);

                if (ntlmHashbytes.Length != Msv1.LM_NTLM_HASH_LENGTH)
                    throw new System.ArgumentException();

                data.NtlmHash = ntlmHashbytes;
```

A new process in the `SUSPENDED` state is opened. Note, that our PTH username is chosen with an empty password:

```batch
                    PROCESS_INFORMATION pi = new PROCESS_INFORMATION();
                    if(CreateProcessWithLogonW(user, "", domain, @"C:\Windows\System32\", binary, arguments, CreationFlags.CREATE_SUSPENDED, ref pi))
```

In the next step, the process is opened and the `LogonID` of the new process is copied into our credential material object, which is related to our PTH username.

<p align="center">
          <img src="/assets/posts/NamedPipePTH/LogonID.JPG">
</p>

Afterwards, the function `Pth_luid` is called. This function first searches for and afterwards overwrites the MSV1.0 and Kerberos credential material with our newly created structure:

<p align="center">
          <img src="/assets/posts/NamedPipePTH/Pth_Luid.JPG">
</p>

If that resulted in success, the process is resumed via `NtResumeProcess`.

## Named Pipe Impersonation 

Thinking about alternative ways for PTH user Impersonation I asked <a href="https://twitter.com/_EthicalChaos_">@_EthicalChaos_</a> about my approach/ideas and the use-case. Brainstorming with you is always a pleasure, thanks for that! Some ideas for the use-case were:

* NTLM challenge response locally via InitializeSecurityContext / AcceptSecurityContext
* Impersonation via process token
* Impersonation via named pipe identity
* Impersonation via RPC Identity

I excluded the first one, because I simply had no idea about that and never worked with it before. Impersonation via process token or RPC Identity required an existing process for the target user to steal the token from. A process for the target user doesn't exist in my szenario, so only Named Pipe Impersonation was left. And I thought cool, I already worked with that to build a script to get a `SYSTEM` shell - <a href="https://github.com/S3cur3Th1sSh1t/Get-System-Techniques/blob/master/NamedPipe/NamedPipeSystem.ps1">NamedPipeSystem.ps1</a>. So I'm not completely lost in the topic and know what it is about.

For everyone out there, who doesn't know about Named Pipe Impersonation I can recommend the following blog post by <a href="https://twitter.com/decoder_it">@decoder_it</a>:
* <a href="https://decoder.cloud/2019/03/06/windows-named-pipes-impersonation/">https://decoder.cloud/2019/03/06/windows-named-pipes-impersonation/</a>

Again, I will give a short high-level overview for it. Named Pipes are ment to be used for asynchronous or synchronous communication between processes. It's possible to send or receive data via Named Pipes locally or over the network. Named Pipes on a Windows Operating System are accessible over the `IPC$` network share. One Windows API call, namely `ImpersonateNamedPipeClient()` allows the server to impersonate any client connecting to it. The `only` thing you need for that is the `SeImpersonatePrivilege` privilege. Local administrators and many service-accounts have this privilege by default. So opening up a Named Pipe with this privileges enables us to Impersonate any user connecting to that Pipe via `ImpersonateNamedPipeClient()` and open a new process with the token of that user-account.

My first thought about Named Pipe Impersonation in combination with PTH was, that I could spawn a new `cmd.exe` process via `Mimikatz` or `SharpKatz` Pass-the-Hash and connect to the Named Pipe over `IPC$` in the new process. If the network credentials are used for that, we would be able to fulfill all our goals for a new tool. So I opened up a new Powershell process via PTH and SharpKatz with the following command:

```batch
.\SharpKatz.exe --Command pth --User testing --Domain iPad --NtlmHash 7C53CFA5EA7D0F9B3B968AA0FB51A3F5 --Binary "\WindowsPowerShell\v1.0\powershell.exe"
```

What happens in the background? That is explained above. To test, that we are really using the credentials for the user `testing` we can connect to a linux boxes SMBServer:

```batch
smbserver.py -ip 192.168.126.131 -smb2support testshare /mnt/share
```

After opening up the server we can connect to it via simply echoing into the share:

<p align="center">
          <img src="/assets/posts/NamedPipePTH/TestShare.JPG">
</p>

And voila, the authentication as `testing` came in, so this definitely works:

<p align="center">
          <img src="/assets/posts/NamedPipePTH/Connection.JPG">
</p>

<a href="https://twitter.com/decoder_it">@decoder_it's</a> wrote a Powershell script - <a href="https://github.com/decoder-it/pipeserverimpersonate/blob/master/pipeserverimpersonate.ps1">pipeserverimpersonate.ps1</a> - which let's us easily open up a Named Pipe Server for user Impersonation and to open `cmd.exe` afterwards with the token of the connecting user. The next step for me was to test, if connections from this new process connect to the Named Pipe Server with the network credentials. It turned out, that this unfortunately is not the case:

<p align="center">
          <img src="/assets/posts/NamedPipePTH/Fail.JPG">
</p>

I tried to access the Pipe via `127.0.0.1`, `Hostname`, `External IP`, but the same result in every case:

<p align="center">
          <img src="/assets/posts/NamedPipePTH/MoreFails.JPG">
</p>

I also tried using a NamedPipeClient via Powershell - maybe this would result in network authentication with the user `testing` - still no success:

<p align="center">
          <img src="/assets/posts/NamedPipePTH/Fail3.JPG">
</p>

At this point I had no clue on how I could trigger network authentication to localhost for the Named Pipe access. So I gave up on Mimikatz and SharpKatz - but still learned something by doing that. And maybe some of you also learned something in this section. This was a dead end for me.

But what happens exactly when network authentication is triggered? To check that, I monitored the network interface for SMB access from one Windows System to another one:

<p align="center">
          <img src="/assets/posts/NamedPipePTH/Wireshark.JPG">
</p>

1. The TCP/IP Three-way-handshake is done (SYN,SYN/ACK,ACK)
2. Two Negotiate Protocol Requests and Responses
3. Session Setup Request, `NTLMSSP_NEGOTIATE` + `NTLMSSP_AUTH`
4. Tree Connect Request to `IPC$`
5. Create Request File `testpipe`

During my tool research I took a look at <a href="https://twitter.com/kevin_robertson">@kevin_robertson's</a> <a href="https://github.com/Kevin-Robertson/Invoke-TheHash/blob/master/Invoke-SMBExec.ps1">Invoke-SMBExec.ps1</a> code and found, that this script contains exactly the same packets and sends them manually. So by modifying this script, it could be possible to skip the Windows default behaviour and just send exactly those packets manually. This would simulate a remote system authenticating to our Pipe with the user `testing`.

I went through the <a href="https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-smb2/5606ad47-5ee0-437a-817e-70c366052962">SMB documentation</a> for some hours, but that did not help me much to be honest. But than I had the idea to just monitor the default `Invoke-SMBExec.ps1` traffic for the testing user. Here is the result:

<p align="center">
          <img src="/assets/posts/NamedPipePTH/Loopback.JPG">
</p>

Comparing those two packet captures results in only one very small difference. `Invoke-SMBExec.ps1` tries to access the Named Pipe `svcctl`. We can easily change that in <a href="https://github.com/Kevin-Robertson/Invoke-TheHash/blob/master/Invoke-SMBExec.ps1#L1562">line 1562</a> and <a href="https://github.com/Kevin-Robertson/Invoke-TheHash/blob/master/Invoke-SMBExec.ps1#L2248">2248</a> for the `CreateRequest` and `CreateAndXRequest` stage, by using different hex values for another Pipe name. So if we only change those bytes to the following, a `CreateRequest` request is send to our attacker controlled Named Pipe:

```batch
$SMB_named_pipe_bytes = 0x74,0x00,0x65,0x00,0x73,0x00,0x74,0x00,0x70,0x00,0x69,0x00,0x70,0x00,0x65,0x00 # \testpipe
```

The result is an local authentication to the Named Pipe as the user `testing`:

<p align="center">
          <img src="/assets/posts/NamedPipePTH/FirstSuccess.JPG">
</p>

To get rid of the error message and the resulting timeout we have to do some further changes to the `Invoke-SMBExec` code. I therefore modified the script, so that after the `CreateRequest` a `CloseRequest`, `TreeDisconnect` and `Logoff` packet is send instead of the default code execution stuff for Service creation and so on. I also removed all Inveigh Session stuff, parameters and so on.

But there still was one more thing to fix. I got the following error from `cmd.exe` when impersonating the user `testing` via network authentication:

<p align="center">
          <img src="/assets/posts/NamedPipePTH/WindowStationError.png">
</p>

This error *didn't pop up*, when a `cmd.exe` was opened with the password, accessing the Pipe afterwards.

Googling this error results in many many crap answers ranging from `corrupted filesystem, try to repair it` to `install DirectX 11` or `Disable Antivirus`. I decided to <a href="https://twitter.com/ShitSecure/status/1382711809208115203">ask the community</a> via Twitter and got a fast response from <a href="https://twitter.com/tiraniddo">@tiraniddo</a>, that the error code is likely due to not being able to open the Window Station. A solution for that is changing the `WinSta/Desktop` DACL to grant everyone access. I would have never figured this out, so thank you for that! :-) <a href="https://twitter.com/decoder_it">@decoder_it</a> also send a link to RoguePotato, especially the code for setting correct WinSta/Desktop permissions <a href="https://github.com/antonioCoco/RoguePotato/blob/master/RoguePotato/Desktop.cpp">is included there</a>.

## Modifying RoguePotato & building one script as PoC

Taking a look at the `Desktop.cpp` code from RoguePotato I decided pretty fast, that porting this code to Powershell or C# is no good idea for me as I would need way too much time for that. So my idea was, to modify the <a href="https://github.com/antonioCoco/RoguePotato">RoguePotato</a> code to get a PipeServer which sets correct permissions for `WinSta/Desktop`. Doing this was straight forward as I mostly had to remove code. So I removed the RogueOxidResolver components, the IStorageTrigger and so on. The result is the <a href="https://github.com/S3cur3Th1sSh1t/NamedPipePTH/tree/main/Resources/PipeServerImpersonate">PipeServerImpersonate</a> code.

Testing the server in combination with our modified Invoke-SMBExec script resulted in no shell at first. The `CreateProcessAsUserW` function did not open up the desired binary even though the token had `SE_ASSIGN_PRIMARY_NAME` privileges. I ended up using `CreateProcessWithTokenW` with `CREATE_NEW_CONSOLE` as dwCreationFlags, which worked perfectly fine. Opening up the Named Pipe via modified RoguePotato and connecting to it via <a href="https://github.com/S3cur3Th1sSh1t/NamedPipePTH/blob/main/Resources/Invoke-NamedPipePTH.ps1">Invoke-NamedPipePTH.ps1</a> resulted in successfull Pass-the-Hash to a Named Pipe for Impersonation and binary execution with the new token:

<p align="center">
          <img src="/assets/posts/NamedPipePTH/Success.JPG">
</p>

Still - this is not a perfect solution. Dropping PipeServerImpersonate to disk and executing the script in another session is one option, but a single script doing everything is much better in my opinion. Therefore I build a single script, which leverages <a href="https://github.com/PowerShellMafia/PowerSploit/blob/master/CodeExecution/Invoke-ReflectivePEInjection.ps1">Invoke-ReflectivePEInjection.ps1</a> to execute PipeServerImpersonate from memory. This is done in the background via `Start-Job`, so that `Invoke-NamedPipePTH` can connect to the Pipe afterwards. It's possible to specify a custom Pipe Name and binary for execution:

<p align="center">
          <img src="/assets/posts/NamedPipePTH/PoC.JPG">
</p>

This enables us to use it from a C2-Server as module. You could also specify a C2-Stager as binary, so that you will get a new agent with the credentials of the PTH user.

## Further ideas & improvements

I see my code still as PoC, because it is far away from being OPSEC safe and I didn't test that much possible use-cases. Using Syscalls for PipeServerImpersonate and PE-Injection instead of Windows API functions would further improve this for example.

For those of you looking for a C# solution: <a href="https://github.com/checkymander/Sharp-SMBExec">Sharp-SMBExec</a> is a C# port of `Invoke-SMBExec` which can be modified the same way I did here to get a C# version for the PTH to the Named Pipe part. However, the PipeServerImpersonate part should also be ported, which in my opinion is more work todo.

The whole project gave me the idea, that it would be really cool to also add an option to <a href="https://github.com/SecureAuthCorp/impacket">impacket</a>'s  `ntlmrelayx.py` to relay connections to a Named Pipe. Imagine you compromised a single host in a customer environment and this single host didn't gave any valuable credentials but has `SMB Signing disabled`. Modifying PipeServerImpersonate, so that the Named Pipe is not closed but re-opened again after executing a binary would make it possible to get a C2-Stager for every single incoming NetNTLMV2 connection. This means raining shells. The connections only need to be relayed to `\\targetserver\IPC$\pipename` to get a shell or C2-connection. 

## Conclusion

This is the first time, that I created somehow a new technique. At least I didn't see anyone else using a combination of PTH and Named Pipe Impersonation with the same goal. For me, this was a pretty exciting experience and I learned a lot again.

I hope, that you also learned something from that or at least can use the resulting tool in some engagements whenever you are stuck in a situation described above. The script/tool is released with this post, and feedback is as always very welcome!

<a href="https://github.com/S3cur3Th1sSh1t/NamedPipePTH">https://github.com/S3cur3Th1sSh1t/NamedPipePTH</a>

## Links & Resources

* Crackmapexec - <a href="https://github.com/byt3bl33d3r/CrackMapExec">https://github.com/byt3bl33d3r/CrackMapExec</a>
* Impacket - <a href="https://github.com/SecureAuthCorp/impacket/">https://github.com/SecureAuthCorp/impacket/</a>
* Invoke-TheHash - <a href="https://github.com/Kevin-Robertson/Invoke-TheHash/">https://github.com/Kevin-Robertson/Invoke-TheHash/</a>
* evil-winrm - <a href="https://github.com/Hackplayers/evil-winrm">https://github.com/Hackplayers/evil-winrm</a>
* mimikatz - <a href="https://github.com/gentilkiwi/mimikatz">https://github.com/gentilkiwi/mimikatz</a>
* Rubeus - <a href="https://github.com/GhostPack/Rubeus">https://github.com/GhostPack/Rubeus</a>
* Inside Mimikatz part I - <a href="https://www.praetorian.com/blog/inside-mimikatz-part1/">https://www.praetorian.com/blog/inside-mimikatz-part1/</a>
* Inside Mimikatz part II - <a href="https://www.praetorian.com/blog/inside-mimikatz-part2/">https://www.praetorian.com/blog/inside-mimikatz-part2/</a>
* SharpKatz - <a href="https://github.com/b4rtik/SharpKatz/">https://github.com/b4rtik/SharpKatz/</a>
* NamedPipeSystem - <a href="https://github.com/S3cur3Th1sSh1t/Get-System-Techniques/blob/master/NamedPipe/NamedPipeSystem.ps1">https://github.com/S3cur3Th1sSh1t/Get-System-Techniques/blob/master/NamedPipe/NamedPipeSystem.ps1</a>
* Windows Named Pipes Impersonation - <a href="https://decoder.cloud/2019/03/06/windows-named-pipes-impersonation/">https://decoder.cloud/2019/03/06/windows-named-pipes-impersonation/</a>
* PipeServerImpersonate.ps1 - <a href="https://github.com/decoder-it/pipeserverimpersonate/blob/master/pipeserverimpersonate.ps1">https://github.com/decoder-it/pipeserverimpersonate/blob/master/pipeserverimpersonate.ps1</a>
* SMB Documentation - <a href="https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-smb2/5606ad47-5ee0-437a-817e-70c366052962">https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-smb2/5606ad47-5ee0-437a-817e-70c366052962</a>
* RoguePotato - <a href="https://github.com/antonioCoco/RoguePotato">https://github.com/antonioCoco/RoguePotato</a>
* Invoke-ReflectivePEInjection - <a href="https://github.com/PowerShellMafia/PowerSploit/blob/master/CodeExecution/Invoke-ReflectivePEInjection.ps1">https://github.com/PowerShellMafia/PowerSploit/blob/master/CodeExecution/Invoke-ReflectivePEInjection.ps1</a>
* Sharp-SMBExec - <a href="https://github.com/checkymander/Sharp-SMBExec">https://github.com/checkymander/Sharp-SMBExec</a>
* NamedPipePTH - <a href="https://github.com/S3cur3Th1sSh1t/NamedPipePTH">https://github.com/S3cur3Th1sSh1t/NamedPipePTH</a>
