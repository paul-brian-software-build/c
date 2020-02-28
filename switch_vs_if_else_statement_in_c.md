##Switch vs if-else conditionals in C edit_wiki_cmd

[]{#topic1}
From [ansi c 3.6.4.2 "The switch statement"]{.caps}

> ["The controlling expression of a switch statement shall have integral type.  The expression of each case label shall be an [integral constant expression]{.bold}.  No two of the case constant expressions in the same switch statement shall have the same value after conversion.  There may be at most one default label in a switch statement.  (Any enclosed switch statement may have a default label or case constant expressions with values that duplicate case constant expressions in the enclosing switch statement.)"]{.caps}


While very limiting, the restriction on constant type expression in `case`{.c} labels for the `switch`{.c} statements in C, guarantees a huge performance boost for the compiled program.


[]{#topic2}
The following if-else statement:

```` {.c .numberLines}
void func(unsigned long a) {
	if (a == 1) {
		a1();
	} else if (a == 2) {
		a2();
	} else if (a == 3) {
		a3();
	} else if (a == 4) {
		a4();
	} else if (a == 5) {
		a5();
	} else {
		a_default();
	}
}
````



When compiled, we have:
```` {.S .numberLines}
0000000000000620 <func>:
 620:	48 83 ec 08          	sub    $0x8,%rsp
 624:	31 c0                	xor    %eax,%eax
 626:	48 83 ff 01          	cmp    $0x1,%rdi
 62a:	74 34                	je     660 <func+0x40>
 62c:	48 83 ff 02          	cmp    $0x2,%rdi
 630:	74 3e                	je     670 <func+0x50>
 632:	48 83 ff 03          	cmp    $0x3,%rdi
 636:	74 58                	je     690 <func+0x70>
 638:	48 83 ff 04          	cmp    $0x4,%rdi
 63c:	74 12                	je     650 <func+0x30>
 63e:	48 83 ff 05          	cmp    $0x5,%rdi
 642:	74 3c                	je     680 <func+0x60>
 644:	e8 b7 00 00 00       	callq  700 <a_default>
 649:	31 c0                	xor    %eax,%eax
 64b:	48 83 c4 08          	add    $0x8,%rsp
 64f:	c3                   	retq   
````

Quite a tedious branching decision where the cmp [opcode]{.italic} had to be repeated for every single case:

```` {.S .numberLines}
 626:	48 83 ff 01          	cmp    $0x1,%rdi
...
 62c:	48 83 ff 02          	cmp    $0x2,%rdi
...
 632:	48 83 ff 03          	cmp    $0x3,%rdi
...
 638:	48 83 ff 04          	cmp    $0x4,%rdi
...
 63e:	48 83 ff 05          	cmp    $0x5,%rdi
````

--------------------
[]{#topic3}

Enter the elegance and efficiency of the `switch`{.c} statement:

```` {.c .numberLines}
void func(unsigned long a) {
	switch(a) {
		case 1:
			a1();
			break;
		case 2:
			a2();
			break;
		case 3:
			a3();
			break;
		case 4:
			a4();
			break;
		case 5:
			a5();
			break;
		default:
			break;
	}
}
````

```` {.S .numberLines}
0000000000000620 <func>:
 620:	48 83 ff 05          	cmp    $0x5,%rdi
 620:	48 83 ff 05          	cmp    $0x5,%rdi
 624:	77 6a                	ja     690 <func+0x70>
 626:	48 8d 15 67 01 00 00 	lea    0x167(%rip),%rdx
 62d:	48 83 ec 08          	sub    $0x8,%rsp
 631:	48 63 04 ba          	movslq (%rdx,%rdi,4),%rax
 635:	48 01 d0             	add    %rdx,%rax
 638:	ff e0                	jmpq   *%rax
````

There is one and only [instruction]{.italic} that is shared by [all]{.bold} case labels:

```` {.c .numberLines}
 638:	ff e0                	jmpq   *%rax
````

Meaning, that no matter how convoluted and how many conditionals checking your `switch`{.c} statements accounts for, it will all compile down to a shared, [one single]{.bold} [instruction]{.italic} that accounts for all the different `case`{.c} labels.

[]{#topic4}
So, we investigate:

We have:
```` {.c .numberLines}
 626:	48 8d 15 67 01 00 00 	lea    0x167(%rip),%rdx
````

```` {.c .numberLines}
#62d is start of the next instruction
$ printf '%x\n' $((0x62d + 0x167))
794
````

```` {.c .numberLines}
0000000000000790 <_IO_stdin_used>:
 790:	01 00                	add    %eax,(%rax)
 792:	02 00                	add    (%rax),%al
 794:	b1 fe                	mov    $0xfe,%cl
 796:	ff                   	(bad)  
 797:	ff cc                	dec    %esp
 799:	fe                   	(bad)  
 79a:	ff                   	(bad)  
 79b:	ff                   	(bad)  
 79c:	dc fe                	fdivr  %st,%st(6)
 79e:	ff                   	(bad)  
 79f:	ff                   	(bad)  
 7a0:	ec                   	in     (%dx),%al
 7a1:	fe                   	(bad)  
 7a2:	ff                   	(bad)  
 7a3:	ff ac fe ff ff bc fe 	ljmp   *-0x1430001(%rsi,%rdi,8)
 7aa:	ff                   	(bad)  
 7ab:	ff                   	.byte 0xff
````

Calculating "`631: 48 63 04 ba   movslq (%rdx,%rdi,4),%rax`{.s}" - when `%rdi`{.c} == [0]{.bold}

```` {.sh .numberLines}
# case '0'
printf '%x\n' $((0xfffffffffffffeb1 + 0x794))
645
````

```` {.s .numberLines}
 645:	31 c0                	xor    %eax,%eax
 647:	48 83 c4 08          	add    $0x8,%rsp
 64b:	c3                   	retq   
````

Calculating "`631: 48 63 04 ba   movslq (%rdx,%rdi,4),%rax`{.s}" - when `%rdi`{.c} == [1]{.bold}

```` {.sh .numberLines}
# case '1'
printf '%x\n' $((0xfffffffffffffecc + 0x794))
660
````

```` {.s .numberLines}
 660:	e8 3b 00 00 00       	callq  6a0 <a1>
 665:	31 c0                	xor    %eax,%eax
 667:	48 83 c4 08          	add    $0x8,%rsp
 66b:	c3                   	retq   
````

Calculating "`631: 48 63 04 ba   movslq (%rdx,%rdi,4),%rax`{.s}" - when `%rdi`{.c} == [2]{.bold}

```` {.sh .numberLines}
# case '2'
printf '%x\n' $((0xfffffffffffffedc + 0x794))
670
````

```` {.s .numberLines}
 670:	e8 3b 00 00 00       	callq  6b0 <a2>
 675:	31 c0                	xor    %eax,%eax
 677:	48 83 c4 08          	add    $0x8,%rsp
 67b:	c3                   	retq   
````

Calculating "`631: 48 63 04 ba   movslq (%rdx,%rdi,4),%rax`{.s}" - when `%rdi`{.c} == [3]{.bold}

```` {.sh .numberLines}
# case '3'
printf '%x\n' $((0xfffffffffffffeec + 0x794))
680
````

```` {.s .numberLines}
 680:	e8 3b 00 00 00       	callq  6c0 <a3>
 685:	31 c0                	xor    %eax,%eax
 687:	48 83 c4 08          	add    $0x8,%rsp
 68b:	c3                   	retq   
````

Calculating "`631: 48 63 04 ba   movslq (%rdx,%rdi,4),%rax`{.s}" - when `%rdi`{.c} == [4]{.bold}

```` {.sh .numberLines}
# case '4'
printf '%x\n' $((0xfffffffffffffeac + 0x794))
640
````

```` {.s .numberLines}
 640:	e8 8b 00 00 00       	callq  6d0 <a4>
 645:	31 c0                	xor    %eax,%eax
 647:	48 83 c4 08          	add    $0x8,%rsp
 64b:	c3                   	retq   
````

Calculating "`631: 48 63 04 ba   movslq (%rdx,%rdi,4),%rax`{.s}" - when `%rdi`{.c} == [5]{.bold}

```` {.sh .numberLines}
# case '5'
printf '%x\n' $((0xfffffffffffffebc + 0x794))
650
````

```` {.s .numberLines}
 650:	e8 8b 00 00 00       	callq  6e0 <a5>
 655:	31 c0                	xor    %eax,%eax
 657:	48 83 c4 08          	add    $0x8,%rsp
 65b:	c3                   	retq   
````
