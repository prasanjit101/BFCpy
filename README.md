# BFK program - A brainf*ck compiler implemented in python

The bfk program is a compiler for brainf*ck program, implemented in python. It takes a brainfuck source
file and outputs native code in the form of a 32-bit ELF executable. The
executable only depends on Linux syscalls, not the C library or anything else.
This is basically as fast as unoptimized brainfuck code is going to get.

The output program allocates 1 MB (1048576 cells) of memory on the stack for the
tape. Because the stack grows downwards, the increase and decrease
pointer operations are reversed in the output code.


## Details

The input source goes through the

- Lexing
- Parsing
- Optimizing
- Compiling
- Linking

After all this processes the result is passed in an output executable. 
The lexer simply discards all characters that are not part of grammar and turns the characters
into tokens, implemented as child classes of "Token".

The parser is a recursive descent parser and basically only exists for loops.

The parser is implemented according to the following LL(1) grammar in extended Backus–Naur form (EBNF) form:

    
    program  = { command | loop }
    command  = ">" | "<" | "+" | "-" | "." | ","
    loop     = "[", { command | loop }, "]
    

The optimizer merges repeated increment and decrement instructions, so that the
code generator can easily emit add and subtract instructions instead.

The code generator produces one or two instructions for each node in the AST.
Jumps are written with the correct relative addresses corresponding to loop
beginnings and endings.

The final pass creates a executable and linker (ELF) file that instructs the loader to load
the entire file into memory and run the program
positioned in the file after the ELF and program header.

There are no checks at runtime, so it can write or read memory where it shouldn't.

The code generated for each token is listed below. Input and output is handled
by the read and write syscalls on Linux. They are called using interrupt `0x80`.
The exit syscall is used to end the program.

**>**
```nasm
dec esp
```

**<**
```nasm
inc esp
```

**+**
```nasm
inc [esp]
```

**-**
```nasm
dec [esp]
```

**.**
```nasm
mov eax, 0x4 ; write
mov ebx, 0x1 ; stdout
mov ecx, esp
mov edx, 0x1 ; write 1 byte at pointer
int 0x80
```

**,**
```nasm
mov eax, 0x3 ; read
mov ebx, 0x0 ; stdin
mov ecx, esp
mov edx, 0x1 ; read 1 byte to pointer
int 0x80
```

**[**
```nasm
loop:
    cmp [esp], 0
    je loop_end
```

**]**
```nasm
    jmp loop
loop_end:
```

**EOF**
```nasm
mov eax, 1 ; exit
mov ebx, 0 ; EXIT_SUCCESS (= 0)
int 0x80
```

It may be an interesting experiment to mark the code section itself as writable
and allow for self-modifying code that way.

## Usage

Run the python program with the brainfuck program filename as argument :

    $ ./bfc.py tests/hello.bf

If the file could be read and correctly parsed, then an executable named equivalent of filename
will be produced in the same directory.

    $ input
    Hello tests/World!

## TODOs 

#
- add bounds checks to every read and write