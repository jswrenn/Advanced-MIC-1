# Advanced-MIC-1
A short guide I wrote on some advanced MIC-1 programming techniques. 

## Instruction Overview

| machine language | hex  | mnemonic | long name       | action
| ---------------- | ---- | -------- | ---------       | ------
| 0000xxxxxxxxxxxx | 0xxx | lodd     | load direct     | AC:=M[x]
| 0001xxxxxxxxxxxx | 1xxx | stod     | store direct    | M[x]:=AC
| 0010xxxxxxxxxxxx | 2xxx | addd     | add direct      | AC:=AC+M[x]
| 0011xxxxxxxxxxxx | 3xxx | subd     | subtract direct | AC:=AC-M[x]
| 0100xxxxxxxxxxxx | 4xxx | jpos     | jump positive   | if N = 0 and Z = 0 then PC:=x
| 0101xxxxxxxxxxxx | 5xxx | jzer     | jump zero       | if Z = 1 then PC:=x
| 0110xxxxxxxxxxxx | 6xxx | jump     | jump            | PC:=x
| 0111xxxxxxxxxxxx | 7xxx | loco     | load constant   | AC:=x, x nonnegative
| 1000xxxxxxxxxxxx | 8xxx | lodl     | load local      | AC:=M[SP+x]
| 1001xxxxxxxxxxxx | 9xxx | stol     | store local     | M[SP+x]:=AC
| 1010xxxxxxxxxxxx | Axxx | addl     | add local       | AC:=AC+M[SP+x]
| 1011xxxxxxxxxxxx | Bxxx | subl     | subtract local  | AC:=AC-M[SP+x]
| 1100xxxxxxxxxxxx | Cxxx | jneg     | jump negative   | if N = 1 then PC:=x
| 1101xxxxxxxxxxxx | Dxxx | jnzr     | jump nonzero    | if Z = 0 then PC:=x
| 1110xxxxxxxxxxxx | Exxx | call     | call procedure  | SP:=SP-1; M[SP]:=PC; PC:=x
| 1111000000000000 | F000 | pshi     | push indirect   | SP:=SP-1; M[SP]:=M[AC]
| 1111001000000000 | F200 | popi     | pop indirect    | M[AC]:=M[SP]; SP:=SP+1
| 1111010000000000 | F400 | push     | push            | SP:=SP-1; M[SP]:=AC
| 1111011000000000 | F600 | pop      | pop             | AC:=M[SP]; SP:=SP+1
| 1111100000000000 | F800 | retn     | return          | PC:=M[SP]; SP:=SP+1
| 1111101000000000 | FA00 | swap     | swap AC and SP  | tmp:=AC; AC:=SP; SP:=tmp
| 11111100yyyyyyyy | FCyy | insp     | increment SP    | SP:=SP+x, y nonnegative
| 11111110yyyyyyyy | FEyy | desp     | decrement SP    | SP:=SP-x, y nonnegative



## Basic Self Modifying Code - fibonacci

    0000    2007    MAIN    ADDD    0007    ; Add 0003 to the accumulator
    0001    1002            STOD    0002    ; Store the result in cell 0002
    0002    0000            LODD    ????    ; Were it becomes a LODD instruction
    0003    6003    STOP    JUMP    STOP    ; Terminate by looping infinitely.                
    0004    0001                            ; F1
    0005    0001                            ; F2
    0006    0002                            ; F3
    0007    0003                            ; F4
    0008    0005                            ; F5
    0009    0008                            ; F6
    000A    000D                            ; F7
    000B    0015                            ; F8
    000C    0022                            ; F9
    000D    0037                            ; F10
    000E    0059                            ; F11
    000F    0090                            ; F12
    0010    00E9                            ; F13
    0011    0179                            ; F14
    0012    0262                            ; F15
    0013    03DB                            ; F16
    0014    063D                            ; F17
    ; Specifications:
    ; Accumulator is input and output.
    ; Where i is the input: 0000 < i < 00A1
    ; Where o is the output: 0001 < o < 063D
    ; Input and output are signed integers.


## Advance Self Modifying Code - eval function

In the previous function, we were able to make some very convenient assumptions 
about the input, namely that it would be in the form 0000xxxxxxxxxxxx. We can 
make this assumption because our acceptable input falls within a very small 
range, 0000 to 00A1. We can write this accumulator value directly to a cell 
and have it become a load instruction; we just needed to add a small offset 
value to it. How do we handle cases where our input doesn't fit so neatly with
the computational constraints?
        
    ; eval (func pointer)
    0001    0006    EVAL    LODD    CALL    ; Load the memory address storing the base value for a CALL command
    0002    A000            ADDL            ; Add the base value for a CALL instr to the function pointer passed as an argument
    0003    1004            STOD    _SMC    ; Write our smc code value into the next cell, where it will be executed
    0004    0000    _SMC    ????    E???    ; This is our SMC cell whose value will be executed
    0005    F800            RETN            ; Exit the function
    0006    E000    CALL    CALL    ????    ; Memory cell for storing the 'base' call value

Alternative, we can write this function to use the accumulator as input and output:

    0001    2005    EVAL    ADDD    CALL    ; Load the memory address storing the base value for a CALL command
    0002    1003            STOD    _SMC    ; Write our smc code value into the next cell, where it will be executed
    0003    0000    _SMC    ????    E???    ; This is our SMC cell whose value will be executed
    0004    F800            RETN            ; Exit the function
    0005    E000    CALL    CALL    ????    ; Memory cell for storing the 'base' call value

## Basic Array Operations

We'll combine both styles to produce the basic array operations, get and set.
Arrays are nothing more than a contiguous area of memory storing data. We'll use
a C model of arrays and keep it data-only; metadata such as length will be the
job of the programmer to keep track of.

    ; get (addr, offset)
    000C    8002     GET    LODL    addr    ; load the array offset into memory
    000D    A001            ADDL    ofst    ; add the starting address to the index offset
    000E    00A1            STOD    _SMC    ; store the address into the next cell where it becomes a load direct instruction
    000F    0000    _SMC    LODD    ????    ; self modifying code cell for load-direct
    00A0    F800            RETN            ; end of get function 

    ; set (addr, offset, value)
    00A1    8003     GET    LODL    addr    ; load the array offset into memory
    00A2    A002            ADDL    ofst    ; add the starting address to the index offset
    00A3    20A7            ADDD    STOD    ; add to the base value for the 'stod' cell
    00A4    1004            STOD    _SMC    ; Write our smc code value into the next cell, where it will be executed
    00A5    1000    _SMC    STOD    ????    ; self modifying code cell for store-direct
    00A6    F800            RETN            ; end of get function 
    00A7    1000    STOD    STOD    CALL    ; memory address storing the base value for a STOD command


## Functional Programing in MAC-1

As you might imagine, constructing a function such as EVAL allows us to leverage advance
programming techniques in the vein of functional programming! Let's write a 
function using EVAL that takes a comparator function that places a 0 or a non-
zero in the accumulator (false and true, respectively) and executes one of two
respective function pointers. We'll do this using the second EVAL function we
designed for simplicity, since it will polute the stack less.

    ; if (comparator function pointer, true branch function, false branch function)
    0006    8003      IF    LODL    comp    ; Pop the comparator pointer off the stack
    0007    E001            CALL    EVAL    ; eval the comparator
    0008    8001            LODL    fn_t    ; load the false function pointer off the stack
    0009    500B            JZER    IF_F    ; jump to the false point if AC==0
    000A    8002    IF_T    LODL    fn_f    ; load the true function pointer off the stack
    000B    E001    IF_F    CALL    EVAL    ; evaluate the function pointer in the accumulator
    000C    F800            RETN            ; end of IF function

This is almost functional programming. A more functional implementation would be
to return the function pointer of the branch that SHOULD be executed. This fits
the 'everything is an expression' pattern better.

    EVAL(IF(COMP,TRUE,FALSE))
    ; if (comparator function pointer, true branch function, false branch function)
    0006    8003      IF    LODL    comp    ; Pop the comparator pointer off the stack
    0007    E001            CALL    EVAL    ; eval the comparator
    0008    8001            LODL    fn_t    ; load the false function pointer off the stack
    0009    500B            JZER    IF_F    ; jump to the false point if AC==0
    000A    8002    IF_T    LODL    fn_f    ; load the true function pointer off the stack
    000B    F800            RETN            ; end of IF function

For-Each
    TODO



The entire library so far:

    ; eval
    0001    2005    EVAL    ADDD    CALL    ; Load the memory address storing the base value for a CALL command
    0002    1003            STOD    _SMC    ; Write our smc code value into the next cell, where it will be executed
    0003    0000    _SMC    ????    E???    ; This is our SMC cell whose value will be executed
    0004    F800            RETN            ; Exit the function
    0005    E000    CALL    CALL    ????    ; Memory cell for storing the 'base' call value
    ; if (comparator function pointer, true branch function, false branch function)
    0006    8003      IF    LODL    comp    ; Pop the comparator pointer off the stack
    0007    E001            CALL    EVAL    ; eval the comparator
    0008    8001            LODL    fn_t    ; load the false function pointer off the stack
    0009    500B            JZER    IF_F    ; jump to the false point if AC==0
    000A    8002    IF_T    LODL    fn_f    ; load the true function pointer off the stack
    000B    F800            RETN            ; end of IF function
    ; get (addr, offset)
    000C    8002     GET    LODL    addr    ; load the array offset into memory
    000D    A001            ADDL    ofst    ; add the starting address to the index offset
    000E    00A1            STOD    _SMC    ; store the address into the next cell where it becomes a load direct instruction
    000F    0000    _SMC    LODD    ????    ; self modifying code cell for load-direct
    00A0    F800            RETN            ; end of get function 
    ; set (addr, offset, value)
    00A1    8003     GET    LODL    addr    ; load the array offset into memory
    00A2    A002            ADDL    ofst    ; add the starting address to the index offset
    00A3    20A7            ADDD    STOD    ; add to the base value for the 'stod' cell
    00A4    1004            STOD    _SMC    ; Write our smc code value into the next cell, where it will be executed
    00A5    1000    _SMC    STOD    ????    ; self modifying code cell for store-direct
    00A6    F800            RETN            ; end of get function 
    00A7    1000    STOD    STOD    CALL    ; memory address storing the base value for a STOD command
