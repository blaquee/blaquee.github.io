---
title: A brief look at Compiler Optimization (Dead Code Elimination)
author: blaquee
layout: post
description: A practical look at Dead Code Elimination.
---

## Introduction

What I will be demonstrating here today is a practial look into compiler optimization, namely **dead code elimination. **This will not be a theoretical post or in-depth implementation of dead code elimination techniques, instead I will show you how a particular compiler utilizes this technique to change the disassembly of your high level code (C++). The compiler used will be Visual C++ from Visual Studio 2013 Compiler Suite. Although we&#8217;ll be looking at the disassembly of VC++, the same information here can also be applied to other mature C++ compilers (gcc, llvm, clang) with slight variations on the assembly output.

## What is Dead Code Elimination?

Dead Code Elimination is an optimization technique used by compilers of high level languages to systematically remove portions of the code that either aren&#8217;t used, or if used, the results are not used or affect the program operation. While its main purpose is to remove &#8216;dead code&#8217;, DCE can also result in better performant code. More can be read about this behavior in the references following this post.

## The Code

We will be looking at this piece of code. If you stare at this code for a second (or decide to compile it) you will notice some things, first of all, its pretty useless.

```C++
void doForLoop(int count)
{
	//lets mess around abit
	count = 3;
	if (count &lt; 5)
	{
		printf("I don't wanna loop da loop, and you can't make me!\n");
	}
	else{
		for (int i = 0; i &lt; count; i++)
			printf("Loop da loop! + %d \n", i);
	}
}

int main(int argc, char** argv)
{
	if (argc != 2){
		printf("You shouldnt be so mean, pass a number or i crash!\n");
	}
	//lets get this thing to loop da loop!
	doForLoop(atoi(argv[1]));

	return 0;
}
```

We'll compile this code both using optimization disabled, and then optimization enabled. Here are some of the commands for popular compilers. The ```-m32``` is only required if you&#8217;re running a 64bit operating system, else the compilers will default to compiling in x86_64 mode.

```bash
_clang -m32 -o unopt  filename.c_
_gcc -m32 -o unopt filename.c_
_cl /Od /FA filename.c_
```

Clang and gcc&#8217;s commands are interchangeable, while Visual C++ takes commands preceeded with forward slashes. ```/Od``` means optimization disabled. For the optimized equivalent of these commands, with gcc and clang, it is sufficient to add ```-O``` to the command, for VC++, you may choose ```/O2``` to optimize the code for speed. Let us quickly take a look at the assembly generated from this piece of code. This is our main function. I removed the prologue and argument sanitation and jumped to the relevant part, right before the call to our looping function.

```assembly
$LN1@main:
; Get second argument in argv
	mov	eax, 4
	shl	eax, 0
	mov	ecx, DWORD PTR _argv$[ebp]
	mov	edx, DWORD PTR [ecx+eax]
; Get our paramter, convert to int
	push	edx
	call	_atoi
	add	esp, 4
	push	eax
; Call our loop function
	call	_doForLoop
	add	esp, 4
```

We can also take a look at the ```_doForLoop()_``` function and notice that everything is also compiled, just as we typed in our C code.

```assembly
;doForloop Function
_i$1 = -4						; size = 4
_count$ = 8						; size = 4
_doForLoop PROC
; Function prologue
	push	ebp
	mov	ebp, esp
	push	ecx
; Move 3 into count variable
	mov	DWORD PTR _count$[ebp], 3
; Compare count to 5
	cmp	DWORD PTR _count$[ebp], 5
	jge	SHORT $LN5@doForLoop

	push	OFFSET stringData
	call	_printf
	add	esp, 4
; Exits the function
	jmp	SHORT $LN6@doForLoop
$LN5@doForLoop:
; Start of for Loop
	mov	DWORD PTR _i$1[ebp], 0
	jmp	SHORT $LN3@doForLoop
$LN2@doForLoop:
	mov	eax, DWORD PTR _i$1[ebp]
	add	eax, 1
	mov	DWORD PTR _i$1[ebp], eax
$LN3@doForLoop:
; is i greater than or equal to count?
	mov	ecx, DWORD PTR _i$1[ebp]
	cmp	ecx, DWORD PTR _count$[ebp]
; if it is, we're done
	jge	SHORT $LN6@doForLoop
; print line in else for loop
	mov	edx, DWORD PTR _i$1[ebp]
	push	edx
	push	OFFSET $SG4425
	call	_printf
	add	esp, 8
	jmp	SHORT $LN2@doForLoop
$LN6@doForLoop:
...
```

Everything looks normal here, if we look at our ```doForLoop()``` function, we&#8217;ll see all of our code is in tact, just the way the Gods intended. Now let us compile this code with optimization enabled. For VC++ replace /Od with /O2, for optimization for speed. In gcc and clang, just add -O to enable optimization for those compilers. Lets have another look at the generated assembly code.

```assembly
$LN1@main:
; Line 24
	mov	eax, DWORD PTR _argv$[esp-4]
	push	DWORD PTR [eax+4]
	call	_atoi
	push	OFFSET stringData
	call	_printf
	add	esp, 8
; Where did my CALL GO!?!?!?
	xor	eax, eax
	ret	0
```

Now if we take a look at our ```_doForLoop()_``` function, we find an even more shocking reveal!

```assembly
_count$ = 8						; size = 4
_doForLoop PROC						; COMDAT
; My Code has disappeared!
	mov	DWORD PTR _count$[esp-4], OFFSET stringData(I don't wanna loop...)
	jmp	_printf
_doForLoop ENDP
```

Oh my! We’ve just managed to make our code disappear without modifying the code at all!  What you just witnessed is your compiler doing it’s job. Making several passes on the code and applying its logic to make it faster, and more efficient. You’ll notice several things with the optimized code. First of all, in our ```_main()_``` function, there is absolutely no call to the ```doForLoop()``` function. Like I mentioned in the introduction to **dead code elimination, **the compiler will remove code who’s result isn’t used, or doesnot affect the program logic. Indeed we see that calling ```_doForLoop()_``` is not necessary overhead for our ```_main()_``` function, so instead the compiler decided it would optimize that function call out and place the print call of the ```_doForLoop()_``` function inside of``` _main()_```. This is actually not eliminating the code but it is moving code, sometimes referred to as **Code Motion Optimization.**

Furthermore, our ```_doForLoop()_``` function has been stripped naked literally. First of all, the for loop has been completely eliminated because it will never get executed due to the assignment of ```_count_``` to 3 why by pure mathematical reasoning we know will always be less than 5.  Using DCE and other methods the compiler determines that the function _doForLoop()_ only needs to print the results and nothing further.

## Conclusion

We&#8217;ve seen alittle bit of what happens when our code is compiled and what can be changed when certain switches are used. Compiler Optimizations are a complex subject and require alot of knowledge of some low level assembly optimization techniques. Dead Code Elimination is just one name to identify many variants of its kind. Hopefully you learned something and until next time. Keep Coding

### References

1. [Visual C++ Team Blog post on Dead Code Elimination](http://blogs.msdn.com/b/vcblog/archive/2013/08/09/optimizing-c-code-dead-code-elimination.aspx)
2. [University of California Riverside Slides on DCE](http://www.cs.ucr.edu/~gupta/teaching/201-12/My8.pdf)
3. [Collection of Compiler Optimization Techniques](http://www.compileroptimizations.com/index.html)