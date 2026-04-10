---
title: "WIFI Credential Dumping"
layout: "post"
---

The security of WPA2 PSK-protected networks depends mainly on the complexity of the chosen PSK itself. Most attacks with the goal of getting unauthorized access to a WPA2 PSK WIFI network somehow do Offline Wordlist or Brute-Force attacks to retrieve the key either from a Handshake or PMKID. We at r-tec recommend companies to not use WPA2 PSK - even when a strong/complex PSK is chosen. On the one hand side, a complex PSK can easily be phished over Evil-Twin attacks.

<!--more-->

This blog post was written and published on my employer's website, where it can be found here:

- [https://www.r-tec.net/r-tec-blog-wifi-credential-dumping.html](https://www.r-tec.net/r-tec-blog-wifi-credential-dumping.html)

On the other hand side, the PSK is only a single-factor authentication, which is rarely changed on a regular base. If an attacker retrieves this key at some point, access to the company network can be maintained through any physical device near the company building afterwards. This blog won't dive into any of the mentioned WIFI attacks, but will highlight techniques to retrieve the PSK from a workstation post-compromise instead.

## But Why?

Post compromise? Does it make sense for an attacker to retrieve a PSK when access to a domain system is already there? From our point of view - yes, of course. Even after getting initial access to some client system or server, a detection could always lead to the attacker being kicked out of the network again. For persistence purposes, a key could be used to regain access. We also often see company networks using WPA2 PSK for production networks, development networks, or other critical areas instead of the Office IT network. In those cases, retrieving the PSK can also lead to Lateral Movement into those protected networks. So yes, there are reasons for an attacker to dump these credentials post-compromise.

## 1. Finding WIFI profiles with credentials

Enumeration on how to find connected WIFI networks can be done via different ways. The builtin Windows binary `netsh.exe` can for example be used to get information about saved WIFI profiles:

```cmd
netsh wlan show profiles
```

This can also be done from a low privileged user's perspective.

<p align="center">
          <img src="/assets/posts/WIFICredentialDumping/Blog-image01-Displaying_saved_WIFI_profiles-with-netsh.png">
</p>

*Figure 1: Displaying the saved WIFI profiles with netsh, executed from C2 framework*

WIFI profiles are typically stored in `.xml` files in the folder `C:\ProgramData\Microsoft\Wlansvc\Profiles\Interfaces` on a Windows Operating System. The following PowerShell code could be used as an alternative to retrieve SSIDs, which have a PSK saved:

```powershell
function Get-Saved-Wifis{
    Get-ChildItem -Path "C:\ProgramData\Microsoft\Wlansvc\Profiles\Interfaces" -Filter "*.xml" -Recurse | ForEach-Object { 
        $xmlContent = [xml](Get-Content $_.FullName)
        $profileName = $xmlContent.WLANProfile.name
        $authType = $xmlContent.WLANProfile.MSM.security.authEncryption.authentication
        $password = $xmlContent.WLANProfile.MSM.security.sharedKey.keyMaterial

        if ($password -ne $null -and $password -ne "") {
            "$profileName - $authType"
        }
    }
}
```

<p align="center">
          <img src="/assets/posts/WIFICredentialDumping/Blog-image02-Retrieving_saved_WIFI_profiles-via-reflective_PowerShell_execution.png">
</p>

*Figure 2: Retrieving saved WIFI profiles via reflective PowerShell execution*

## 2. Extracting the cleartext PSK

To extract the PSK from saved WIFI networks you will need to be either in an elevated process (local administrator) or in the context of the user who initially configured this WIFI connection. So typically Privilege Escalation must already have taken place or some client administrator user account being obtained. The most common and documented way to get the cleartext PSK is by also using `netsh` again. You just need to specify the target WIFI SSID and `key=clear` arguments like this:

```cmd
netsh wlan show profile "TargetSSID" key=clear
```

<p align="center">
          <img src="/assets/posts/WIFICredentialDumping/Blog-image03-Extracting-cleartext-key-via-netsh.png">
</p>

*Figure 3: Extracting the cleartext key via netsh*

There are multiple public scripts already, which automate this process for all found SSIDs. However, I found that depending on the OS language (or other factors) the parsing of the SSID/PSK didn't work correctly. Therefore, my personal approach would be to output all information and manually look for the PSK myself:

```powershell
netsh wlan show profile | Where-Object { $_ -match ': (.+)$' } | ForEach-Object {
    $ssid = $matches[1].Trim()
    netsh wlan show profile name=$ssid key=clear
}
```

In heavily monitored environments, however, process creation and usage of native Windows binaries is considered as not OPsec safe, as it's super easy to build a detection rule for this technique and you might get flagged + the PSK gets changed in the worst case.

So we need to find out what's happening in the background for alternatives to retrieve the PSK. Little research on the topic shows, that DPAPI encryption is used to protect the WIFI PSK key material in the already mentioned `.xml` files.

<p align="center">
          <img src="/assets/posts/WIFICredentialDumping/Blog-image04-Key_material_in_XML-file.png">
</p>

*Figure 4: Key material in the XML file*

The encryption is not done in the context of the current user but instead with the `LocalSystem` secret. As I recently built a PoC to decrypt `Veeam` [MSSQL DPAPI saved credentials](https://github.com/S3cur3Th1sSh1t/SharpVeeamDecryptor), I knew, that decryption for `LocalSystem` DPAPI saved credentials can be done from an elevated context. I slightly modified the [WheresMyImplant](https://github.com/0xbadjuju/WheresMyImplant/blob/4503bf932dc5ea0f8cae1de7c9835edf236683f3/WheresMyImplant/Credentials/WirelessProfiles.cs#L50) code for a standalone `C#` assembly and tried to decrypt the DPAPI secrets, which failed:

<p align="center">
          <img src="/assets/posts/WIFICredentialDumping/Blog-image05-DPAPI_decryption_fails.png">
</p>

*Figure 5: DPAPI decryption fails*

Well, it looks like the `DataProtectionScope` of `LocalSystem` is not equal to `LocalSystem` or there is at least some missing piece of the puzzle. Doing the same thing after impersonating the SYSTEM account however, did lead to successful cleartext credential retrieval:

<p align="center">
          <img src="/assets/posts/WIFICredentialDumping/Blog-image06-DPAPI_decryption_succeeds-SYSTEM.png">
</p>

*Figure 6: DPAPI decryption succeeds as SYSTEM*

A modified version of the WIFI credential dumping code, which includes impersonation of the SYSTEM account before decryption can be found here:

[Link to Github (https://gist.github.com/S3cur3Th1sSh1t/ca315ccd46ae4b28c0abb4b450cbeedb)](https://gist.github.com/S3cur3Th1sSh1t/ca315ccd46ae4b28c0abb4b450cbeedb)

Impersonation of SYSTEM on the other hand, is also alerted or blocked by some EDR vendors via userland hooks, so consider doing proper unhooking before using this code. I was wondering at this point - are there alternatives to SYSTEM impersonation? [Asking the community](https://twitter.com/ShitSecure/status/1766048125242814630) resulted in a proper answer by [subat0mik](https://twitter.com/subat0mik) here. As already implemented in [SharpSCCM](https://github.com/Mayyhem/SharpSCCM/blob/f43eb0812a7c7109a62ac9062db4b94419bf2420/lib/LSADump.cs#L60) it's also possible to modify the LSA Secrets key DACLs, so that non-SYSTEM users can retrieve the masterkey.

What sounds easy here resulted in >1000 more lines of code in the PoC for WIFI credential extraction, as no Windows API can be used for DPAPI decryption with custom masterkeys but instead, the whole process needs to be done manually. Luckily for us, projects like [Mimikatz](https://github.com/gentilkiwi/mimikatz/), [SharpDPAPI](https://github.com/GhostPack/SharpDPAPI/) or [SharpSCCM](https://github.com/Mayyhem/SharpSCCM/) contain all the needed logic/code already. The modified PoC can be found here:

[Link to Github (https://gist.github.com/S3cur3Th1sSh1t/bc8a7b1b7972f25bda687a33bd0ebde5)](https://gist.github.com/S3cur3Th1sSh1t/bc8a7b1b7972f25bda687a33bd0ebde5)

<p align="center">
          <img src="/assets/posts/WIFICredentialDumping/Blog-image07-Elevated_DPAPI_decryption.png">
</p>

*Figure 7: Elevated DPAPI decryption*

## 3. Conclusion

At this point, it should be clear, that using WPA2 PSK for Office-IT network access or other sensitive production networks is not a very good idea. The best recommendation practice is to use `WPA2/3 Enterprise` with Multi-Factor-Authentication. Domain authentication with username and password is also not sufficient here due to potential Phishing attacks.

On the other hand side from the attacker's perspective, it's definitely worth taking a look at connected WIFI networks. Getting the PSK can be used for Persistence as well as Lateral Movement depending on the target network configuration.

DPAPI encryption with `LocalSystem` is not equal to DPAPI encryption with `LocalSystem`, as we saw in the manual decryption part. The key, which is used to encrypt the PSK is not the same as the one being used by the DPAPI Windows APIs. Or at least, it can only be used by the `SYSTEM` user itself **or** by manually modifying the LSA secret registry DACLs to get the masterkey for manual decryption.

---
*This blog post was originally published on [r-tec.net](https://www.r-tec.net/r-tec-blog-wifi-credential-dumping.html).*