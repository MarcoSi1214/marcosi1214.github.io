So yeah, I haven't posted here in a while. Stuff got in the way, but hey, here's another post ¯\_(ツ)_/¯. I'll probably be uploading io.netgarage writeups every 1-2 weeks, or as time permits.


# io.netgarage.org: level1
We start by logging into the challenge server
```
ssh level1@io.netgarage.org
password: level1
```
```
 ______   _____
/\__  _\ /\  __`\       Levels are in /levels
\/_/\ \/ \ \ \/\ \      Passes are in ~/.pass
   \ \ \  \ \ \ \ \     Readmes in /home/level1
    \_\ \__\ \ \_\ \
    /\_____\\ \_____\   Server admin: bla (blapost@gmail.com)
    \/_____/ \/_____/

level1@io:~$
```

We change directory to `/levels`, as stated in the MOTD and have a look at 
what's in there:
```
level1@io:/levels$ ls
beta           level06_alt       level11        level17.c      level25.c
level01        level06_alt.c     level11.c      level17_alt    level26
level02        level06_alt.pass  level12        level17_alt.c  level26.l
level02.c      level07           level12.c      level18        level26.y
level02_alt    level07.c         level12.pass   level18.c      level27
...
```
We run `./level01` to have a look at what we're dealing with:
```
level1@io:/levels$ ./level01
Enter the 3 digit passcode to enter:
```
So it's asking for a passcode, let's try a random value:
```
Enter the 3 digit passcode to enter: 000
level1@io:/levels$
```
So clearly we can't just supply a random value, let's fire up `GDB` and 
figure this out:
```
level1@io:/levels$ gdb ./level01
(gdb)
```
Now let's have a look at the disassembly of it:
```
(gdb) disas main
Dump of assembler code for function main:
   0x08048080 <+0>:	    push   $0x8049128
   0x08048085 <+5>:	    call   0x804810f
   0x0804808a <+10>:	call   0x804809f
   0x0804808f <+15>:	cmp    $0x10f,%eax
   0x08048094 <+20>:	je     0x80480dc
   0x0804809a <+26>:	call   0x8048103
End of assembler dump.
(gdb)
```
Ok, so it pushes a value onto the stack, then calls the function at 
`0x804810f`, then the function at `0x804809f`, compares the `eax` register 
to hex `0x10f`, and then jumps to the function at `0x80480dc` if they're equal.

Let's try get a better sense as to what those functions are:
```
(gdb) x/35c 0x8049128
(1)
0x8049128:	69 'E'	110 'n'	116 't'	101 'e'	114 'r'	32 ' '	116 't'	104 'h'
0x8049130:	101 'e'	32 ' '	51 '3'	32 ' '	100 'd'	105 'i'	103 'g'	105 'i'
0x8049138:	116 't'	32 ' '	112 'p'	97 'a'	115 's'	115 's'	99 'c'	111 'o'
0x8049140:	100 'd'	101 'e'	32 ' '	116 't'	111 'o'	32 ' '	101 'e'	110 'n'
0x8049148:	116 't'	101 'e'	114 'r'
(gdb) disas 0x804810f
(2)
Dump of assembler code for function puts:
   0x0804810f:	mov    0x4(%esp),%ecx
...
End of assembler dump.
(gdb) disas 0x804809f
(3)
Dump of assembler code for function fscanf:
   0x0804809f:	sub    $0x1000,%esp
...
End of assembler dump.
(gdb) disas 0x80480dc
Dump of assembler code for function YouWin:
   0x080480dc:	mov    $0x4,%eax
...
End of assembler dump.
(gdb)
```
Lucky for us, the functions are all named. Revisiting what we said before, the 
program will push the address of the "Enter the 3 digit passcode to enter:" 
string (1), then it will call `puts()` (2), then calls `fscanf()` (3), finally 
if `eax` equals hex `0x10f`, it will jump to the `YouWin()` function.

Considering that `eax` needs to be `0x10f` to go to `YouWin()`, let's try that 
as the password (271 in decimal):
```
level1@io:/levels$ ./level01
Enter the 3 digit passcode to enter: 271
Congrats you found it, now read the password for level2 from /home/level2/.pass
sh-4.3$
```
Great, we have a shell! We can now read the pass from `/home/level2/.pass`:
```
sh-4.3$ cat /home/level2/.pass
<REDACTED>
```
And so, level01 is solved.

Happy Hacking!

~MarcoSi
