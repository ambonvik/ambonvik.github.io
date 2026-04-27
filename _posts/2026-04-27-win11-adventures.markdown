---
title: "Adventures with Windows 11"
date: 2026-04-27  19:30:20 +0300
description: "Convincing Windows 11 that coroutines are not a stack-smashing exploit"
excerpt: "Windows 11 has far stronger stack security than Windows 10, supported by 
modern CPU hardware capabilities. Coroutines can easily trip up in that."
mathjax: true
header:
  overlay_image: /assets/images/windows11_wide.png
  teaser: /assets/images/windows11_teaser.png
---
My original development machine for Cimba was an old Intel Xeon E5 running Windows 10. 
However, it was not possible to switch to Windows 11 on it, not even with a TPM 2.0 
chip installed. Windows being what it is, I took that as the impetus for switching to an 
AMD Threadripper and Arch Linux for the main development rig. I still wanted to have 
Windows support for Cimba, though, so I checked from time to time that it still 
worked on the Xeon as well.

I then set up a GitHub Actions process to automate building and testing Cimba 
there. Effectively, it runs `meson build; meson test on the GitHub runners every time 
I push a change. The runners are specified as "latest" both for Linux Ubuntu and Windows.
That presumably implies new hardware and the newest versions of the OS.

It was an unpleasant surprise to see that the Windows test suite on GitHub failed all 
tests that involved coroutines, both directly as in `test_coroutine.c` and everything 
built on top of it like `test_process.c`. Windows 11 promptly killed the executable as 
soon as it was trying to start a coroutine. The only error message given was a rather 
terse error code '0xc0000005', access violation.

This was quite a debugging challenge. As always, the Windows internals are not 
particularly well-documented, but two things soon became clear:
1. Windows 11 has far stricter security measures to prevent stack-smashing exploits 
   than older versions and will summarily kill any suspect process.
2. This is supported by modern hardware such as the [Intel Control-flow Enforcement 
   Technology](https://www.intel.com/content/www/us/en/developer/articles/technical/technical-look-control-flow-enforcement-technology.html)
   (CET) that monitors program execution and alerts the OS to any suspicious activity.

This is, of course, a very good thing, but it was not able to distinguish the 
legit Cimba stackful coroutines from a hacker attack that tries to gain 
control by manipulating the stack and the return addresses there. As described in the 
[Cimba documentation](https://cimba.readthedocs.io/en/latest/background.html#cimba-processes-are-asymmetric-coroutines),
the coroutines work by creating individual stacks in heap memory. This fails when the 
CPU and OS are monitoring a "shadow stack" to continuously verify that the program's 
own stack still matches the secret copy. In effect, we have to explain to the OS and 
CPU both that our cactus stack is a valid stack and that our manipulations of return 
pointers are harmless.

Basically, the relevant parts of the Cimba context switching code for Windows followed 
the outline given by Malte Skarupke in his 2013 blog post
["Handmade Coroutines for Windows"](https://probablydance.com/2013/02/20/handmade-coroutines-for-windows/).
That still works for Windows 10, but Windows 11 on a modern AMD or Intel CPU, perhaps 
with a hypervisor inbetween, is a very different animal.

With significant assistance from both Google Gemini and Anthropic Claude together with
human trial-and-error testing, I was able to find a set of code clarifications that was
accepted. The complete code is [in the Cimba repo](https://github.com/ambonvik/cimba/tree/main/src/port/x86-64/windows),
so I will only review the main points here. 

1. Tell it explicitly to expect stack shenanigans. Even if not using any Windows fiber 
   library functions, call `ConvertThreadToFiber(NULL);` to state that here be dragons.
   For extra credit, wrap it in a test where the magic number '0x1e00' means "not a 
   fiber":
   ```C    
   if (GetCurrentFiber() == (void*)0x1e00) {
        ConvertThreadToFiber(NULL);
   }
   ```
2. Stack memory is different. Do not simply call `malloc()` or `calloc()` to allocate new 
   stack memory. Windows 11 expects page-aligned memory allocated by `VirtualAlloc()`
   as valid stack space. After use, it needs to be freed by the matching `VirtualFree()`. 
   Allocate and designate a guard page at the end of the stack.

```C
    /* Allocate memory suitable for a stack */
    unsigned char *cmi_coroutine_stack_alloc(const size_t size,  
                                             unsigned char **base_p,
                                             unsigned char **limit_p)
    {
        void *raw = VirtualAlloc(NULL, size, MEM_COMMIT | MEM_RESERVE, PAGE_READWRITE);
        cmb_assert_always(raw != NULL);
     
        DWORD old_protect;
        VirtualProtect(raw, 4096u, PAGE_READWRITE | PAGE_GUARD, &old_protect);
     
        /* The stack grows downwards, the base is at the top, less a few bytes for the OS */
        *base_p = raw + size - 16u;
     
        /* The bottom includes the guard page */
        *limit_p = raw + 4096u;
     
        return raw;
    }
     
    /* Free memory previously allocated for a stack */
    void cmi_coroutine_stack_free(unsigned char *stack)
    {
        int r = VirtualFree(stack, 0, MEM_RELEASE);
        cmb_assert_always(r != 0);
    }
```
3. There is a third value to worry about in the
   [Thread Information Block](https://en.wikipedia.org/wiki/Win32_Thread_Information_Block#Stack_information_stored_in_the_TIB)
   (TIB), the _DeallocationStack_ at `GS:1478`. It contains the raw 
   memory address returned by `VirtualAlloc()`. As 
   [the Wikipedia article](https://en.wikipedia.org/wiki/Win32_Thread_Information_Block#Stack_information_stored_in_the_TIB)
   states: 
   _"Setting stack limits without setting DeallocationStack will probably cause odd 
   behavior in SetThreadStackGuarantee. For example, it will overwrite the stack 
   limits to wrong values."_
4. In the actual context switch, the CPU becomes very afraid if it detects an 
   inconsistent state between the stack parameters in the TIB, its CET Shadow Stack, 
   and the actual memory accesses in progress. Wait until the very last moment before 
   changing the TIB values. Change them atomically by loading all three values from 
   the old stack to scratch registers before writing them to the TIB with no 
   interleaving stack access. Only then proceed with accessing the new stack.
5. Do not use `RET` to jump to the new coroutine. The oldest trick in the 
   hacker book is to overwrite the return address by abusing a buffer overflow on the 
   stack and then let the program return to a hacker-selected address, potentially 
   taking full control of the machine. Windows 11 and CET are understandably wary 
   of anything that looks strange there. Instead, spell it out explicitly by 
   popping the return address into a scratch register and then jumping to the address 
   in the register:

   ```ASM
    ;-------------------------------------------------------------------------------
    ; Push the TIB DeallocationStack, StackLimit, and StackBase entries
    mov r9, [gs:1478]      ; DeallocationStack
    push r9
    mov r9, [gs:16]        ; StackLimit
    push r9
    mov r9, [gs:8]         ; StackBase
    push r9
    ;
    ; Store old stack pointer to address given as first argument RCX
    mov [rcx], rsp
    ;
    ; Load the new RSP from the second argument RDX into a scratch register
    mov r9, [rdx]
    ;
    ; Load new stack info into scratch registers for atomic TIB change
    mov r10, [r9]           ; New StackBase
    mov r11, [r9 + 8]       ; New StackLimit
    mov rax, [r9 + 16]      ; New DeallocationStack
    ;
    ; Write the new stack info to Windows TIB without touching the stack
    mov [gs:8], r10         ; Update StackBase
    mov [gs:16], r11        ; Update StackLimit
    mov [gs:1478], rax      ; Update DeallocationStack
    ;
    ; Done, safe to switch to the new stack, advancing past the used TIB entries
    mov rsp, r9
    add rsp, 24
    ;
    ; We are now in the new context, restore other registers from the new stack
    load_context
    ;
    ; Load whatever was in the third argument R8 as return value in RAX
    mov rax, r8
    ;
    ; Return to wherever the new context was transferring from earlier
    ; Note that we spell out the 'ret' as 'pop, jmp' for Intel CET reasons.
    pop r9
    jmp r9
   ```
6. Tell `gcc` not to produce stack-checking code:
   
   ```
    if host_machine.system() == 'windows'
       desired_flags += ['-fno-plt',
                         '-fcf-protection=none',
                         '-fno-stack-protector']
    endif
   ```
   
Once all that was in place, Cimba started working again, also on the latest Windows 
runner. There may be steps in the above that could be omitted, or there could be 
additionaL steps needed under certain circumstances that I have not stumbled into yet. 
If so, please add a comment below.