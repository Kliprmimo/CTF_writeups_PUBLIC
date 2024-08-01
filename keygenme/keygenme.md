# Description
Category: rev
Difficulty: Medium 
Can you get the flag? Reverse engineer this [binary](https://artifacts.picoctf.net/c/51/keygenme).
# Writeup
when we load file into ida we can find first part of flag being copied into some buffor `picoCTF{br1ng_y0ur_0wn_k3y_` 
![](attachments/Pasted%20image%2020240723142812.png)
later this flag is hashed using md5 
result is saved in flag_buff at indexes from 32 to 63
then this hash is hashed again and saved to
check_buff at indexes 0 to 31
after that first 27 indexes of flag_buff are copied to check buff at indexes 32 to 58
this indexes are actually a first part of flag we found earlier, after that ![](keygenme/attachments/Pasted%20image%2020240723144129.png)
letters are copied from random looking indexes from flag buff to check buff, there is also v13 which is not assigned anywhere so im not quite sure what happens here, but that should not be an issue. check_buff[67] is `}` After that letters of check buffer are checked against our input, if those are equal we have a flag.
now as we understand what is happening we can solve challenge.
To sum things up:
there is check_buff created that consists of first 27 letters of flag and some random letters from md5 hash of flag, all we need to do is compare state of memory before letters from flag buff are copied to check buff and we can find out a flag
we set our first breakpoint at `0x00005555555553C8`
and second one at `0x000055555555540A`
![](attachments/Pasted%20image%2020240723145243.png)we dump 100x from top of stack at first address
![](attachments/Pasted%20image%2020240723145411.png)
and the same amount from second
then we save those values to a file and run git compare on them so that we get nice colorfull output
![](attachments/Pasted%20image%2020240723145858.png)
we see some changes, so we can load those into hex to char converter
 ```
hex2ascii 31363338397d386463
>> 16389}8dc
```
due to small endian vs big endian differences we can tell that last letters of flag are:
`19836cd8}`
if we run program with flag that we found we can validate that flag is indeed correct
![](attachments/Pasted%20image%2020240723150601.png)
flag is:
`picoCTF{br1ng_y0ur_0wn_k3y_19836cd8}`