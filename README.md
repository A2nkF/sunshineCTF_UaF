# sunshineCTF_UaF
My Solution for the sunshineCTF's UaF Challenge. I didn't really solve it through the heap though.

## Writeup 
Probably coming soon ;).

## Short Writeup
The first bug is an Info leak, because the `ID`, which the create string/array functions return are pointers to the actual array/string. 
The second bug is a Use after Free() in the function at `0x08048a0c`. This allows you to perform operations (e.g printing, editing) on an array or string. One issue is, that freeing an array results in the size field of the array being set to zero.
This means that even though you could theoretically edit it, it won't work 'cause the index that you want to edit has to be smaller than the size of the array, which means smaller than zero. And that's not possible since we are dealing with unsigned numbers. But using a few tricks, we can still achieve an Out of bounds read/write in the arrays

An int array consists of two chunks:
This is the Layout
```
Chunk1:     Heap Metadata      Array size      index0 ptr                NULL
0x9d4d4b4:    0x00000011       0x00000004      0x09d4d4c8             0x00000000

Chunk2:                     index0            index1         index2
0x9d4d4c4:  0x00000019  0x00000001          0x00000001      0x00000001
              index3     NULL
0x9d4d4d4:  0x00000001  0x00000000
```
This is how we can get an OOB read/write and escalate that to an arbitrary R/W:
```
1. Alloc arr1,arr2,arr3: Every thing is normal...
Arr1
0x9d4d4b4:  0x00000011  0x00000004  0x09d4d4c8  0x00000000
0x9d4d4c4:  0x00000019  0x00000001  0x00000001  0x00000001
0x9d4d4d4:  0x00000001  0x00000000
Arr2
0x9d4d4dc:  0x00000011  0x00000004  0x09d4d4f0  0x00000000
0x9d4d4ec:  0x00000019  0x00000002  0x00000002  0x00000002
0x9d4d4fc:  0x00000002  0x00000000
Arr3
0x9d4d504:  0x00000011  0x00000004  0x09d4d518  0x00000000
0x9d4d514:  0x00000019  0x00000003  0x00000003  0x00000003
0x9d4d524:  0x00000003  0x00000000
gdb-peda$

2. Free arr1:
Arr1                                +---------------------------- size field of arr1 has been set to zero
0x9d4d4b4:  0x00000011  0x00000000<-| 0x09d4d4c8  0x00000000
0x9d4d4c4:  0x00000019  0x00000000    0x00000001  0x00000001
0x9d4d4d4:  0x00000001  0x00000000
Arr2
0x9d4d4dc:  0x00000011  0x00000004  0x09d4d4f0  0x00000000
0x9d4d4ec:  0x00000019  0x00000002  0x00000002  0x00000002
0x9d4d4fc:  0x00000002  0x00000000
Arr3
0x9d4d504:  0x00000011  0x00000004  0x09d4d518  0x00000000
0x9d4d514:  0x00000019  0x00000003  0x00000003  0x00000003
0x9d4d524:  0x00000003  0x00000000
gdb-peda$

3. Free arr2:
Arr1                                +---------------------------- size field of arr1 is still zero
0x9d4d4b4:  0x00000011  0x00000000<-| 0x09d4d4c8  0x00000000
0x9d4d4c4:  0x00000019  0x00000000    0x00000001  0x00000001
0x9d4d4d4:  0x00000001  0x00000000
Arr2                                +---------------------------- however some data has been written into the size field
0x9d4d4dc:  0x00000011  0x09d4d4b0<-| 0x09d4d4f0  0x00000000      of our second array. This means we have an out of bounds
0x9d4d4ec:  0x00000019  0x09d4d4c0    0x00000002  0x00000002      read/write because the programm thinks that our array now 
0x9d4d4fc:  0x00000002  0x00000000                                has 0x09d4d4b0 elements

Arr3                                            +--------------- we can use that oob write to overwrite the index0 pointer
0x9d4d504:  0x00000011  0x00000004  0x09d4d518<-| 0x00000000     of our third array to point to some location in memory that
0x9d4d514:  0x00000019  0x00000003  0x00000003    0x00000003     we want to read from/write to. After doing so, we can use the
0x9d4d524:  0x00000003  0x00000000                               print_array function(4) to read arbitrary values or the 
gdb-peda$                                                        edit_array function(3) to write arbitrary values :D.
```

Since I don't want to mess araund with the heap, I'll use ROP to get Code execution. Rop requires stack control, which we don't
have yet but we can use our write primitive to get there. First we leak the address of some libc function and calculate the
base. Then we can use the write to overwrite the GOT entry for `fgets()` with a pointer to the `gets()` function. `gets()` is prone to a stack buffer overflow. This gives us full stack control, however the binary has been compiled with stack cannary's on.
This will cause the process to crash, if a buffer overflow is detected. To bypass that, we could use our read primitive
to leak the cannary before overflowing but the only way for doing that, that I could think of involved messing around with the heap, which I still don't want to. A much easier way to bypass that is to just let the process detect the overflow. After 
detecting a buffer overflow, the program will attept to call the `__stack_chk_fail` function. This function lies within libc 
and thus has an entry in the Global Offset Table. This also means that we can overwrite the pointer to that function with a
pointer to let's say a `ret` instruction, thereby bypass the crash and get full stack control. At this point, getting Code
Execution is just a matter of crafting a short `system('/bin/sh\0')` ROP-chain and we get a shell!
