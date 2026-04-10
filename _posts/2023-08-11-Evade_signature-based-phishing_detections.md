---
title: "Evade signature-based phishing detections"
layout: "post"
---

Phishing attacks are still the most used attack vector for initial access and credential stealing from our perspective. As phishing attempts become more frequent and sophisticated, so do vendors with detection/prevention features. Especially in recent years, it got much harder for us to simulate phishing attacks due to those protection mechanisms. One of these protection mechanisms is Google Safe Browsing. What is Safe Browsing?

<!--more-->

According [to google](https://developers.google.com/safe-browsing):

*"Safe Browsing is a Google service that lets client applications check URLs against Google's constantly updated lists of unsafe web resources. Examples of unsafe web resources are social engineering sites (phishing and deceptive sites) and sites that host malware or unwanted software. Come see what's possible.*

*With Safe Browsing you can:*

- *Check pages against our Safe Browsing lists based on platform and threat types.*
- *Warn users before they click links in your site that may lead to infected pages.*
- *Prevent users from posting links to known infected pages from your site."*

## 1. Safe Browsing in action

What could a typical phishing scenario from a Pentest/Red-Team look like nowadays? Especially after the pandemic and many more companies joined at least with a hybrid model into the Azure Cloud, the Microsoft Login Page seems to be a valuable target. There are plenty of publicly available Phishing templates for it already:

<p align="center">
          <img src="/assets/posts/PhishingDetections/Figure1_Github_search_Microsoft_Login_Phishing.png">
</p>

*Figure 1: Github search for Microsoft Login Phishing pages*

[Some](https://github.com/JoniRinta-Kahila/microsoft-login-spoof/tree/main/HTML%26JS-only) of these give the same feeling and look like the original login page:

<p align="center">
          <img src="/assets/posts/PhishingDetections/Figure2_Fake_Microsoft_Login_page.png">
</p>

*Figure 2: Fake Microsoft Login page*

However, if you embed this template code into your GoPhish instance and send E-Mails to a target organization, the website will be accessible only for a few minutes. Because as soon as the e-mails are sent, automatic scans of the phishing page might be triggered, causing a Google Safe Browsing categorization and leading to your page looking like this for victims:

<p align="center">
          <img src="/assets/posts/PhishingDetections/Figure3_Firefox_Google_Safe_Browsing_alert.png">
</p>

*Figure 3: Firefox Google Safe Browsing alert*

This is not only true for chrome, but also for all other browsers (as they all use Safe Browsing at the very end) and may look different on the end user side:

<p align="center">
          <img src="/assets/posts/PhishingDetections/Figure4_Edge_Defender_SmartScreen_alert.png">
</p>

*Figure 4: Edge Defender SmartScreen alert*

Users could potentially ignore the alert, but this is very unlikely due to several steps as well as the appearance of the warning. So in the very end, we can assume that this is an effective and good protection mechanism.

## 2. Evading Safe Browsing

To avoid this kind of detection from an attacker's perspective, we need to know exactly what gets us caught. However, the very first assumption here is simple: We used a publicly available phishing template and a scanner marked this as malicious content and classified it like that. This reminds us of the Antivirus/signature-based approaches, right?

If this is true, we could bypass this kind of detection by obfuscating the HTML code and decrypting it on runtime, just as a Packer would do to avoid signature-based detections against an Antivirus. When thinking about it, this process makes more sense in JavaScript, because this is typically only executed within a Browser when the victim visits the page. A scanner could only see some fake HTML content, plus random obfuscated strings and JavaScript content, but not the real page behind it.

Note: This is nothing new at all, Threat Actors use techniques like this [already for years](https://www.proofpoint.com/sites/default/files/proofpoint-obfuscation-techniques-phishing-attacks-threat-insight-en-v1.pdf).

And there are also different HTML Smuggling/Obfuscation PoC's public that could be used for our purpose, such as the following for example:

- [https://github.com/BinBashBanana/html-obfuscator](https://github.com/BinBashBanana/html-obfuscator)

<p align="center">
          <img src="/assets/posts/PhishingDetections/Figure5_Obfuscating_Microsoft_Phishing_page.png">
</p>

*Figure 5: Obfuscating our Microsoft Phishing page*

In this example, the tool combines URL + BASE64 encoding. First, it URL encodes the HTML body and then encodes it with BASE64 on top of it.

<p align="center">
          <img src="/assets/posts/PhishingDetections/Figure6_Decoding_payload_BASE64_BurpSuite.png">
</p>

*Figure 6: Decoding the payload from BASE64 with BurpSuite*

To verify, that Safe Browsing does not catch our newly generated page, we triggered different scans on it via VirusTotal plus other URL malware analysis services. Moreover, even on the next day, Safe Browsing did not categorize the site as malicious.

Therefore, we indeed verified, that at least some Safe Browsing checks are done on the HTML content itself and that this kind of scans can be bypassed by obfuscating the HTML code and de-obfuscating it on runtime. We however do not recommend using any public obfuscators here, as the resulting content – especially with BASE64 and URL-Encoding, can still easily get flagged with signatures. It is recommended to build your own obfuscator for Pentest or Red-Teaming projects.

If you are using encryption, you also might want to reduce the entropy of your HTML page by adding invisible HTML content or even more non-used JavaScript code.

## 3. Conclusion

Phishing protection features implemented by big vendors to increase the security for company organizations and individuals are really helpful to protect against known malicious pages or software and made life somehow harder for Pentest- or Red-Teams in the last years.

In the specific example of Google Safebrowsing however, it turned out that detections are somehow based on signatures and can therefore easily be bypassed by obfuscating the original code and de-obfuscating it on runtime in the Browser via JavaScript.

This technique is nothing new and has already been used for years by Threat Actors. Still, we did not need to use it until some months ago, we never faced Safe Browsing based detections before. This blog can therefore help other Pentest- or Red-Teams simulating phishing attacks to customize their campaigns and landing pages.

---
*This blog post was originally published on [r-tec.net](https://www.r-tec.net/r-tec-blog-evade-signature-based-phishing-detections.html).*