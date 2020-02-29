
edit_wiki_cmd
#Stack Trace

[]{#topic1}
From [gcc Version 7.3.0 manual]{.caps}:
```` {.man .numberLines}
'-fomit-frame-pointer'
     Don't keep the frame pointer in a register for functions that don't
     need one.  This avoids the instructions to save, set up and restore
     frame pointers; it also makes an extra register available in many
     functions.  *It also makes debugging impossible on some machines.*

     ...

     The default setting (when not optimizing for size) for 32-bit
     GNU/Linux x86 and 32-bit Darwin x86 targets is
     '-fomit-frame-pointer'.  You can configure GCC with the
     '--enable-frame-pointer' configure option to change the default.

     Enabled at levels '-O', '-O2', '-O3', '-Os'.
````

The [stack-frame-pointer]{.italic} is a life-saver when chasing after elusive bugs, espeically when the execution path is not straight-forward.

Be it, recursive functions or unexpected jumps, dumping the stack frame, helps to visualize, as well as profile the weakest links in your program.

[]{#topic2}
A very simple example, is tracing file redirection in `sh`{.sh}.

```` {.sh .numberLines}
$ sh -c 'exec >/tmp/output'
````

We get the following call-trace, where the stack grows downward.

```` {.c .numberLines}
[7]  main          +441
[6]  cmdloop       +371
[5]  evaltree      +135
[4]  evalcommand   +453
[3]  redirectsafe  +198
[2]  redirect      +113
[1]  openredirect  +124
````
Chasing after `redirect +113`{.S}, we have:

```` {.s .numberLines}
000000000040ebf0 <redirect>:
````

We then calculate the frame-pointer:

```` {.sh .numberLines}
$ printf '%x\n' $((0x40ebf0 + 113))
40ec61
````
And we can confirm, that [40ec61]{.italic} is the next instruction on the `redirect()`{.c} [stack-frame]{.italic}, right after the `openredirect()`{.c} function-call.

```` {.s .numberLines}
  40ec5c:	e8 bf fa ff ff       	callq  40e720 <openredirect>
  40ec61:	83 f8 ff             	cmp    $0xffffffff,%eax
````

[]{#topic3}
For a more useful example, we demonstrate one of the many pitfalls in [posix]{.italic} shells:

```` {.sh .numberLines}
$ { var=value; true; } | true
$ echo 'var is: '"$var"
var is: 

$ { var=value; true; }
$ echo 'var is: '"$var"
var is: value
````

Enter the call-trace, but not before we examine yet another example:
```` {.sh .numberLines}
$ ( var=value; true ) | true
$ echo 'var is: '"$var"
var is:
````

The striking similiarity between sub-shells and pipes, is evident.

[]{#topic4}
Let see the respective dumps of the processes envolved, where each dump is headed by each process id, [pid]{.italic}.

We shell explore the similarity between sub-shells and pipes:

Stack trace for "`{ var=value; true; } | true`{.sh}":

```` {.S .numberLines}

stack dump for pid [9436]:

evaltree                     +60	[1]
cmdloop                      +355	[2]
main                         +441	[3]

stack dump for pid [9436]:
evaltree                     +60	[1]
cmdloop                      +355	[2]
main                         +441	[3]

stack dump for pid [9439]:

evaltree                     +60	[1]
evalpipe                     +468	[2]
evaltree                     +167	[3]
cmdloop                      +355	[4]
main                         +441	[5]

stack dump for pid [9436]:

evaltree                     +60	[1]
cmdloop                      +355	[2]
main                         +441	[3]
var is: 

stack dump for pid [9436]:

evaltree                     +60	[1]
cmdloop                      +355	[2]
main                         +441	[3]

stack dump for pid [9436]:

evaltree                     +60	[1]
cmdloop                      +355	[2]
main                         +441	[3]
````

Stack trace for "`{ var=value; true; }`{.sh}":

```` {.S .numberLines}
stack dump for pid [9436]:

evaltree                     +60	[1]
cmdloop                      +355	[2]
main                         +441	[3]

stack dump for pid [9436]:

evaltree                     +60	[1]
evaltree                     +299	[2]
cmdloop                      +355	[3]
main                         +441	[4]

stack dump for pid [9436]:

evaltree                     +60	[1]
evaltree                     +167	[2]
cmdloop                      +355	[3]
main                         +441	[4]

stack dump for pid [9436]:

evaltree                     +60	[1]
cmdloop                      +355	[2]
main                         +441	[3]
var is: value

stack dump for pid [9436]:

evaltree                     +60	[1]
cmdloop                      +355	[2]
main                         +441	[3]

stack dump for pid [9436]:

evaltree                     +60	[1]
cmdloop                      +355	[2]
main                         +441	[3]
````
Stack trace for "`( var=value; true ) | true`{.sh}":

```` {.S .numberLines}
stack dump for pid [9436]:

evaltree                     +60	[1]
cmdloop                      +355	[2]
main                         +441	[3]

stack dump for pid [9441]:

evaltree                     +60	[1]
evalpipe                     +468	[2]
evaltree                     +167	[3]
cmdloop                      +355	[4]
main                         +441	[5]

stack dump for pid [9436]:

evaltree                     +60	[1]
cmdloop                      +355	[2]
main                         +441	[3]
````

[]{#tmp}
