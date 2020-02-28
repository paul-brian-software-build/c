##Function pointers in c edit_wiki_cmd

Function pointers are very useful for [generic programming]{.italic} in c.

A typical example is the qsort() prototype in libc:

```` {.c .numberLines}
extern void qsort(void *__base, size_t __nmemb, size_t __size, __compar_fn_t __compar) __nonnull((1, 4));
````

Where `__compar_fn_t`{.c} is defined as:

```` {.c .numberLines}
typedef int (*__compar_fn_t) (const void *, const void *);
````

By providing a callback function `__compar_fn_t`{.c} to the `qsort()`{.c} function, you can pass in any data structure you can imagine, be it a [scalar]{.italic} or an [aggregate]{.italic} type, specifying its size and the number of elements and have it sorted.

Another example is the revolutionary unix convention that [everything is a file]{.italic}, that allows you to `read()`{.c} , `write()`{.c} etc. to and from a [regular]{.italic} file as well as a [pipe]{.italic} or a [socket]{.italic} and so on, by allowing any file-system that implements that following (and more) file-pointers to assume the functionality of a "normal" file-system.

From [include/linux/fs.h]{.bold}:

```` {.c .numberLines}
struct file_operations {
	struct module *owner;
	loff_t (*llseek) (struct file *, loff_t, int);
	ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);
	ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);
	ssize_t (*read_iter) (struct kiocb *, struct iov_iter *);
	ssize_t (*write_iter) (struct kiocb *, struct iov_iter *);
	int (*iterate) (struct file *, struct dir_context *);
	int (*iterate_shared) (struct file *, struct dir_context *);
	__poll_t (*poll) (struct file *, struct poll_table_struct *);
	long (*unlocked_ioctl) (struct file *, unsigned int, unsigned long);
	long (*compat_ioctl) (struct file *, unsigned int, unsigned long);
	int (*mmap) (struct file *, struct vm_area_struct *);
	unsigned long mmap_supported_flags;
	int (*open) (struct inode *, struct file *);
	int (*flush) (struct file *, fl_owner_t id);
	int (*release) (struct inode *, struct file *);
	int (*fsync) (struct file *, loff_t, loff_t, int datasync);
	int (*fasync) (int, struct file *, int);
	int (*lock) (struct file *, int, struct file_lock *);
	ssize_t (*sendpage) (struct file *, struct page *, int, size_t, loff_t *, int);
	unsigned long (*get_unmapped_area)(struct file *, unsigned long, unsigned long, unsigned long, unsigned long);
	int (*check_flags)(int);
	int (*flock) (struct file *, int, struct file_lock *);
	ssize_t (*splice_write)(struct pipe_inode_info *, struct file *, loff_t *, size_t, unsigned int);
	ssize_t (*splice_read)(struct file *, loff_t *, struct pipe_inode_info *, size_t, unsigned int);
	int (*setlease)(struct file *, long, struct file_lock **, void **);
	long (*fallocate)(struct file *file, int mode, loff_t offset,
			  loff_t len);
	void (*show_fdinfo)(struct seq_file *m, struct file *f);
#ifndef CONFIG_MMU
	unsigned (*mmap_capabilities)(struct file *);
#endif
	ssize_t (*copy_file_range)(struct file *, loff_t, struct file *,
			loff_t, size_t, unsigned int);
	int (*clone_file_range)(struct file *, loff_t, struct file *, loff_t,
			u64);
	ssize_t (*dedupe_file_range)(struct file *, u64, u64, struct file *,
			u64);
} __randomize_layout;
````
One of the pitfalls of a function-pointers is the very important distinction between the [fucntion]{.italic} itself and its [pointer]{.italic}

There is a major difference between:
```` {.c .numberLines}
// Not a valid pointer to a main() function
typedef int *main_p(int argc, char **argv);
````

and:
```` {.c .numberLines}
// Valid pointer to a main() fucntion
typedef int (*main_p)(int argc, char **argv);
````

Let's take for example, the following 2 files.

We will compile [main.c]{.italic} and [func.c]{.italic}, where `main()`{.c} will call `g()`{.c} defined in [func.c]{.italic} and declared in [main.c]{.italic}.

We will first use the wrong declaration for a pointer to `g()`{.c}.

[main.c]{.bold}:

```` {.c .numberLines}
// Wrong declation of @fp, the function pointer.
typedef int *fp_type(void);
fp_type g;
int main()
{
	g();
	return 0;
}
````

[func.c]{.bold}:

```` {.c .numberLines}
int func()
{
	return 0;
}
__typeof__(func) *g = &func;
````
We get:

```` {.sh .numberLines}
$ gcc -Wall main.c func.c -o a.out && ./a.out
Segmentation fault      (core dumped)
````

So, we have to inspect the executable to see what went wrong.

Running:  `objdump -d a.out | grep -A5 'main>:' `{.sh}, we have:

```` {.s .numberLines}
0000000000000570 <main>:
 570:	48 83 ec 08          	sub    $0x8,%rsp
 574:	e8 97 0a 20 00       	callq  201010 <g>
 579:	31 c0                	xor    %eax,%eax
 57b:	48 83 c4 08          	add    $0x8,%rsp
 57f:	c3                   	retq   

````

We have in line 8:

|                  `574:	e8 97 0a 20 00       	callq  201010 <g>`{.c}

The [e8]{.italic} opcode is defined in [Intel's Software Documentation Manual]{.bold} (Vol. 2A 3-123):


|       `E8 cd 	CALL rel32 	Call near, relative, displacement relative to next instruction`{.s}

This [rel32]{.italic} is defined as:

> ["A relative offset (rel16 or rel32) is generally specified as a label in assembly code. But at the machine code level, it is encoded as a signed, 16- or 32-bit immediate value. This value is added to the value in the EIP(RIP) register. In 64-bit mode the relative offset is always a 32-bit immediate value which is sign extended to 64-bits before it is added to the value in the RIP register for the target calculation."]{.caps}

On this logic, we can obtain from the following line:

|                  `574:	e8 97 0a 20 00       	callq  201010 <g>`{.c}

and with the proper arithmetic, we reach the desired call target:

```` {.sh .numberLines}
$ printf '%x\n' $((0x579 + 0x200a97))
201010
````

We grep now for `0x201010`{.c} in the executable:

```` {.sh .numberLines}
    $ readelf -aW a.out | grep -A1 '201010'
    58: 0000000000201010     8 OBJECT  GLOBAL DEFAULT   23 g
    59: 0000000000201018     0 NOTYPE  GLOBAL DEFAULT   24 __bss_start
````

So, we see that target call, where the executable will jump to is an object of size 8, where immediately follows the [bss]{.italic} section.


Certainly, not a valid section to load the next instructions, so what is going on.

Let's fix the bug above in [main.c]{.bold}, as the following:

```` {.c .numberLines}
typedef int (*fp_type)(void);
fp_type g;
int main()
{
	g();
	return 0;
}
````
Now, inspect the call target:

```` {.sh .numberLines}
$ gcc -Wall main.c func.c -o a.out
$ objdump -d a.out | grep -A5 'main>:'
````

and we get a [very different]{.bold} call opcode:

```` {.s .numberLines}
0000000000000570 <main>:
 570:	48 83 ec 08          	sub    $0x8,%rsp
 574:	ff 15 96 0a 20 00    	callq  *0x200a96(%rip)        # 201010 <g>
 57a:	31 c0                	xor    %eax,%eax
 57c:	48 83 c4 08          	add    $0x8,%rsp
 580:	c3                   	retq   
````

No longer is it a relative offset to g, but it is an indirect absolute address at g.


Which is exactly the definition a pointer function.

The same logic applies to array pointers, in the following concepts

+ array pointers, i.e. a pointer to an array
+ array of pointers

Which should be treated as 2 [different]{.bold} species.

Implementing them in source code, we do:

```` {.c .numberLines}
typedef char table_t[2][4];
typedef char (*line_t)[4];
typedef char *array[4];
````

On the 1st line we have a definition of a multi-dimensional array.

In this case, we have a table of 2 lines and 4 columns.

To get a pointer to each individual line - an array of 4 bytes - we do:

|        `typedef char (*line_t)[4];`{.c}

And then we can assign it a value, by:

```` {.c .numberLines}
table_t table;
line_t line;
# Get the pointer to the first slot of the array.
line = table;
````

If we will try erroneously, to assign:

|          `array_t array = table;`{.c}

and compile, we will get:

```` {.c .numberLines}
$ gcc -Wall -c main.c
main.c: error: invalid initializer
  array_t line = table[0];
                 ^~~~~
````

A more intricate example from [ansi c (3.5.4.3 "Function declarators")]{.caps}:

|         `int (*fpfi(int (*)(long), int))(int, ...);`{.c}

Here we have a function type definition [fpfi]{.italic}, which:

* has the following parameters:
     + `fpfi(int (*)(long), int)`{.c}, which are 2 parameters:
          -  Parameter 1: `int (*)(long)`{.c}, a function pointer returning `int`{.c}.
          -  Parameter 2: an `int`{.c}
* Returns a pointer to a function, that:
     +   Has the following parameters:
          -  Parameter 1: `int`{.c}
          -  Unspecified number of additional parameters
     +   Returns an `int`{.c}.
