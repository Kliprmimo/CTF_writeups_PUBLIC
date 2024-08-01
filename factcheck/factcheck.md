# Challenge description
Category: rev
Difficulty: Medium
This binary is putting together some important piece of information... Can you uncover that information? Examine this [file](https://artifacts.picoctf.net/c_titan/190/bin). Do you understand its inner workings?
# Writeup
![](attachments/Pasted%20image%2020240723135320.png)
when we first load challenge in ida we can see a lot of weird c++ alocators, but there is first part of our flag
```picoCTF{wELF_d0N3_mate_```
we can also see that this allocators so something on letters that could be part of the flag
![](attachments/Pasted%20image%2020240723135736.png)
later in the decompiled code we see some comparison operations, but what is interesting there is concat operation on ```}``` ![](attachments/Pasted%20image%2020240723135923.png)
and knowing that this is last char of flag we can try to set breakpoint after this operation to see if there is flag in memory

we launch program in gdb with pwndbg plugin
using  ```starti``` command we start programme and break on first instruction so that we can find out  base address using `info proc map` 
![](attachments/Pasted%20image%2020240723140749.png)
base address is `0x555555554000`
now we can rebase our programme in ida, we do that to find out address of instruction that does operation with `}` so that we can break on this address
![](attachments/Pasted%20image%2020240723141153.png)
the address we will break on will be after call of alloc function 
`br *0x0000555555555860`
![](attachments/Pasted%20image%2020240723141352.png)
and as we have guessed the value can be found on the stack
flag is:
`picoCTF{wELF_d0N3_mate_2394047a}`