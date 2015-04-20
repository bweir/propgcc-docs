---
layout: doc
title: Compressed Memory Model
---

Compressed Memory Model (CMM) is a mode of operation wherein the
normal Propeller instructions are compressed into smaller forms, and
then decompressed by the kernel as they are read from HUB memory. This
results in a significant saving in the space required for programs,
but at the cost of execution speed.

CMM mode only works for programs running out of HUB memory; it will
not work for COG programs or from external memory.

Using CMM
---------

To use CMM, specify `-mcmm` on the PropGCC command line (or select the
corresponding option in the IDE). Make sure that all object files that
you intend to link together are compiled with the same memory model;
`-mcmm` objects cannot be linked with `-mlmm` objects.


The CMM Instruction Set
-----------------------

The compressed forms of instructions are byte oriented, and can be
read from any byte boundary. They are of variable length. The first
byte specifies the instruction type and (usually) the destination
register; following bytes fill in the rest of the instruction.

The upper 4 bits of the first byte select the type, and the lower 4
bits typically have the destination register (r0-r15) or a condition code.

MACRO INSTRUCTIONS (see below)

    $0m   "macro" instruction; m selects which one

COMMON OPERATIONS (see below)

    $1y   register/register instruction; y is destination
    $2y   register/4 bit immediate: y is destination
    $3y   register/12 bit immediate: y is destination

MISCELLANEOUS

    $4y   brw: y is the condition for the branch, next 2 bytes are signed offset to the PC
    $5y   mvil; y is destination, next 4 bytes are immediate value
    $6y   mviw; y is destination, next 2 bytes are unsigned immediate

    $7y   brs: y is condition, next byte is a signed offset to the PC
    $8y   skip2: y is condition, no more bytes; if condition, pc = pc + 2
    $9y   skip3: y is condition, no more bytes; if condition, pc = pc + 3

    $Ay   mvib: y is destination, next byte is 8 bit immediate
    $By   mvi0: y is destination, sets register y to 0

    $Cy   leasp: y is destination, next 2 bytes are offset;
          sets y = sp + offset

XMOV: MOV plus COMMON OPERATION

    $Dy   xmov: move instruction followed by register/register operation
    $Ey   xmov: move followed by register/4 bit immediate

PACKED NATIVE

    $Fy   packed unconditional instruction; next 3 bytes are the remaining
          parts of the instruction. The whole long word looks like:

          1111_eeeI	   eee = effects bits(wz,wc,wr) I = immediate bit
          ssss_ssss    sss = source bits (9 in all)
          sddd_dddd    ddd = destination bits (9 in all)
          ddii_iiii    iii = instruction bits (6 in all)

This format was chosen so that the source and destination bits are
laid out in memory the same as for normal PASM instructions, and so
the linker can operate on them the same way. Note that the condition
bits are all 1 for this instruction, which is the same as the $F
selector at the start, so unpacking is a matter of re-arranging the
bits.

If condition bits need to be set on an instruction, use the $0F
"native" macro instruction.

Macro Instructions
------------------

The 16 available macro operations and their expansions are given below:

    $00   no-op
         nop
    $01   reserved for break 
          (currently a no-op)
    $02   ret: return from subroutine
             mov pc,lr
    $03   pushm <byte>: push registers on stack
             mov __TMP0, #<byte>
         call #__LMM_PUSHM
    $04   popm <byte>: pop registers from stack
             mov __TMP0, #<byte>
         call #__LMM_PUSHM
    $05   popret <byte>: pop registers and return
             mov __TMP0, #<byte>
         call #__LMM_PUSHRET
    $06   lcall <long>: call subroutine
             mov __TMP0, #<long>
             call #_LMM_CALL_INDIRECT
    $07   mul: multiply
             call #__MULSI
    $08   udiv: unsigned divide
             call #_UDIVSI
    $09   div: signed divide
             call #_DIVSI
    $0A   mvreg <byte>: register to register move
             mov x, y  where byte is $xy
    $0B   xmov <byte1> <byte2>: two register to register moves
             mov x, y  where byte1 is $xy
         mov z, w  where byte2 is $zw
    $0C   addsp <byte>: add signed 8 bit offset to sp
             add sp, <byte> (note that byte is signed)
    $0D   ljmp <addr>: jump to long address
             call #__LMM_JMP
         long <addr>
    $0E   fcache <word>: load code to fcache
             this is approximately the same as call #__LMM_FCACHE_LOAD,
         except that 16 bytes instead of a long word follow, and the
         pc is then forced to 32 byte alignment. The next <word> bytes
         after the alignment are loaded into the FCACHE and run from
         there.
    $0F    native <long>
             the 32 bits after the $0F byte are executed directly as a
         PASM instruction

Common Operations
-----------------

The most common 16 instruction forms are compressed specially. Here is
a table of those 16 forms:

    add	0-0,0-0             '' common op 0
    sub	0-0,0-0             '' common op 1
    cmps	0-0,0-0 wz,wc   '' common op 2
    cmp	0-0,0-0 wz,wc       '' common op 3
    and	0-0,0-0             '' common op 4
    andn	0-0,0-0         '' common op 5
    neg	0-0,0-0             '' common op 6
    or	0-0,0-0             '' common op 7
    xor	0-0,0-0             '' common op 8
    shl	0-0,0-0             '' common op 9
    shr	0-0,0-0             '' common op A
    sar	0-0,0-0             '' common op B
    rdbyte	0-0,0-0         '' common op C
    rdlong	0-0,0-0         '' common op D
    wrbyte	0-0,0-0         '' common op E
    wrlong	0-0,0-0         '' common op F

Note that these forms are always executed (condition is if_always) and
the effects flags are the defaults for the instruction, unless
otherwise specified.

The compressed encoding for these is as follows:

    $1y $sw: register/register operation:
              opw y, s, where "opw" is the operation selected by w from the
              table above
    $2y $sw:
          opw y, #s, where "opw" is selected by w and s is an unsigned 4 bit
              immediate
    $3y $ss $ws:
              opw y, #sss, where "opw" is selected by w, and s is a signed 12
              bit constant

NOTE that the encoding of which operation to use is in an unusual place
for the 12 bit case!

The first two of these can also be encoded along with a register/register move:

    $Dy $uv $sw: xmov mov u, v  opw y,s
              execute mov u,v, then opw y, s
    $Ey $uv $sw: xmov mov u,v   opw y, #s
              execute mov u,v, then opw y, #s

