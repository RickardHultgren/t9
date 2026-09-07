# T9 Ternary Forth Lab Emulator Manual

**Manual for the 9-trit balanced-ternary two-stack Forth computer emulator**  
Current emulator build: September 2026

---

## Contents

1. [What T9 is](#what-t9-is)
2. [Quick start](#quick-start)
3. [Balanced ternary in T9](#balanced-ternary-in-t9)
4. [The two-stack Forth model](#the-two-stack-forth-model)
5. [Understanding the main controls](#understanding-the-main-controls)
6. [Terminal tab](#terminal-tab)
7. [Forth language reference](#forth-language-reference)
8. [Definitions, variables, constants, and control flow](#definitions-variables-constants-and-control-flow)
9. [CPU tab](#cpu-tab)
10. [Memory / ASM tab](#memory-asm-tab)
11. [Machine instruction set](#machine-instruction-set)
12. [Assembler syntax](#assembler-syntax)
13. [I/O tab](#io-tab)
14. [Physical / Logic tab](#physical-logic-tab)
15. [Architecture / Tests tab](#architecture-tests-tab)
16. [Saving, loading, importing, and exporting](#saving-loading-importing-and-exporting)
17. [Worked examples](#worked-examples)
18. [Debugging and troubleshooting](#debugging-and-troubleshooting)
19. [What is emulated in hardware and what is handled by JavaScript](#what-is-emulated-in-hardware-and-what-is-handled-by-javascript)
20. [Using the emulator to design a physical T9](#using-the-emulator-to-design-a-physical-t9)
21. [Current limitations](#current-limitations)
22. [Quick reference](#quick-reference)

---

# 1. What T9 is

T9 is a browser-based emulator for a small computer architecture designed around three ideas:

- **balanced ternary arithmetic** rather than binary arithmetic;
- a **two-stack Forth-oriented CPU** rather than a conventional register-heavy CPU;
- an architecture simple enough to remain relevant to a future physical machine built from diode matrices, restoring elements, latches, memory, and simple I/O hardware.

The emulator is a single HTML file. It requires no server, external JavaScript library, CDN, or network connection. Open it in a modern browser and the machine is ready to use.

T9 is deliberately not just a JavaScript Forth calculator with ternary numbers painted on the screen. It contains a distinct emulated CPU with:

- 9-trit machine words;
- 27 possible 3-trit opcodes;
- program counter and instruction register;
- data and return stacks;
- memory;
- a ternary ALU;
- explicit FETCH and EXECUTE phases;
- assembler and disassembler;
- numbered I/O ports;
- a Forth layer that compiles colon definitions into emulated machine memory.

The default memory size is **2187 words**, or `3^7` words.

A normal 9-trit value has **19,683 possible patterns**:

```text
3^9 = 19683
```

T9 interprets those values as signed balanced-ternary integers from:

```text
-9841 through +9841
```

Arithmetic wraps around this range.

---

# 2. Quick start

## 2.1 Start the emulator

Open the HTML file in a modern desktop browser. The default tab is **TERMINAL**.

The terminal should show a banner similar to:

```text
TERNARY FORTH
T9 - 9-trit balanced-ternary computer
range -9841..+9841 - 27 ternary opcodes - two stacks
Type 3 4 + .   or   : SQUARE DUP * ;
ok
```

The Forth terminal is already active. **You do not need to press RUN to start Forth.**

This distinction is important:

- typing in the **interactive Forth terminal** executes Forth words;
- pressing **RUN** executes raw machine instructions beginning at the current PC.

If memory at the current PC does not contain a program you intentionally loaded, pressing RUN is not useful.

## 2.2 Your first calculation

Type:

```forth
3 4 + .
```

Press Enter.

Expected result:

```text
7 ok
```

Forth uses postfix notation. Instead of writing `3 + 4`, you place both numbers on the stack and then execute `+`.

## 2.3 Define a new word

Type:

```forth
: SQUARE DUP * ;
```

Then:

```forth
12 SQUARE .
```

Expected result:

```text
144 ok
```

`SQUARE` is not merely stored as a JavaScript function. Its body is compiled into emulated T9 machine memory and later executed by the emulated CPU.

## 2.4 Watch the stack

Type:

```forth
10 20 30
```

The **DATA STACK** panel should now show, from the top downward:

```text
T     30
N     20
+2    10
```

`T` means top of stack. `N` means the next stack item beneath T.

Try:

```forth
SWAP
```

The top two values exchange places.

## 2.5 Run the self-tests

Open **ARCHITECTURE / TESTS** and press **RUN SELF TESTS**.

This is a good first check after moving the HTML file to another computer or browser.

---

# 3. Balanced ternary in T9

## 3.1 Trit values

A binary bit has two states. A ternary **trit** has three.

T9 uses balanced ternary:

| Symbol | Numeric trit value | Meaning |
|---|---:|---|
| `-` | -1 | negative trit |
| `0` | 0 | zero trit |
| `+` | +1 | positive trit |

A 9-trit machine word might look like:

```text
00+-0-++0
```

Each position is a power of 3, just as each bit position in binary is a power of 2.

## 3.2 Decimal and ternary views

The emulator commonly shows the same word in two forms:

```text
DEC       TERNARY
7         000000+-+
```

On the TERMINAL tab, the **trit display** selector lets you switch between:

- symbols: `- 0 +`;
- numeric trits: `-1 0 +1`;
- Unicode minus display: `- 0 +`.

Changing this affects display only. It does not change the stored machine value.

## 3.3 Range and wraparound

T9 stores every arithmetic result as a 9-trit word.

Maximum:

```text
+9841
```

Minimum:

```text
-9841
```

Adding 1 to the maximum wraps to the minimum:

```text
9841 + 1 -> -9841
```

Likewise:

```text
-9841 - 1 -> +9841
```

This is normal machine arithmetic for T9, not an error.

## 3.4 Three-way condition information

Balanced ternary naturally distinguishes:

```text
NEGATIVE
ZERO
POSITIVE
```

T9 makes this explicit in both its CPU instruction set and debugger.

The machine contains separate branch instructions for:

- negative values;
- zero;
- positive values.

The `CMP` instruction also returns exactly one of:

```text
-1   if a < b
 0   if a = b
+1   if a > b
```

This is one of the architecture's most important ternary features.

## 3.5 Entering ternary literals

Decimal integer entry is the most reliable way to enter values in the current build:

```forth
-27
0
243
```

The emulator also contains parsing support for balanced-ternary symbol strings in several input fields. This input path is currently more restrictive than the display path. In particular, some strings made only from ASCII hyphens and zeroes may be interpreted ambiguously or rejected.

For reproducible work, use **decimal values for entry** and the ternary columns for inspection until this parser is expanded.

This is an emulator input limitation, not a limitation of balanced ternary itself.

---

# 4. The two-stack Forth model

T9 has two stacks:

1. **Data stack** - normal values, operands, addresses, flags.
2. **Return stack** - return addresses and values explicitly moved with `>R` and `R>`.

## 4.1 Data stack

If you enter:

```forth
1 2 3
```

conceptually the data stack is:

```text
TOP
  3   <- T
  2   <- N
  1
BOTTOM
```

Arithmetic consumes values from the top and pushes a result.

For example:

```forth
3 4 +
```

has the stack effect:

```text
( 3 4 -- 7 )
```

## 4.2 Stack-effect notation

This manual uses conventional Forth stack-effect comments:

```text
( before -- after )
```

For example:

```text
DUP   ( a -- a a )
DROP  ( a -- )
SWAP  ( a b -- b a )
```

The rightmost item is the top of the stack.

## 4.3 T and N

The CPU state exposes two useful views of the data stack:

- `T` - top item;
- `N` - next item.

They behave like top-of-stack registers from the programmer's perspective, while the emulator stores the complete stack internally.

## 4.4 Return stack

The return stack is automatically used by `CALL` and `EXIT`.

Forth code can also explicitly move values to and from it:

```forth
>R
R>
```

Do not casually mix temporary values and control-flow return addresses unless you understand the resulting stack discipline.

---

# 5. Understanding the main controls

The control bar appears at the top of the emulator.

| Control | Function |
|---|---|
| **RUN** | Run machine instructions continuously from the current PC. |
| **PAUSE** | Stop continuous running without setting the CPU's halted flag. |
| **STEP INSTR** | Execute one complete machine instruction. |
| **STEP CYCLE** | Execute one CPU phase: FETCH or EXECUTE. |
| **RUN 10** | Execute up to 10 instructions. |
| **RUN 100** | Execute up to 100 instructions. |
| **RUN 1000** | Execute up to 1000 instructions. |
| **RESET** | Reset CPU state and stacks while retaining memory and the Forth dictionary. |
| **HARD RESET** | Clear memory, reset dictionary, I/O and terminal, then rebuild the boot kernel. |
| **BREAK** | Stop execution, mark the CPU halted, and write `BREAK` to the terminal. |
| **speed** | Number of machine instructions executed per browser animation frame while RUN is active. |

## 5.1 RUN is not the Forth start button

The Forth outer interpreter is active immediately after page load.

Use **RUN** only when you intentionally want the CPU to execute a machine-code program from memory.

For ordinary Forth use, type into the terminal and press Enter.

## 5.2 RESET versus HARD RESET

Use **RESET** when you want to restart CPU execution without destroying your compiled Forth words or loaded memory.

RESET clears:

- PC back to 0;
- IR state;
- data stack;
- return stack;
- cycle count;
- halted state.

RESET retains:

- memory contents;
- Forth dictionary and definitions;
- variables stored in memory;
- terminal output;
- I/O state.

Use **HARD RESET** when you want to return to a clean boot image.

HARD RESET clears or rebuilds:

- all memory;
- CPU state;
- Forth dictionary;
- user definitions;
- variables;
- I/O;
- terminal log;
- built-in kernel and convenience I/O words.

The emulator asks for confirmation before a hard reset.

## 5.3 STEP INSTR versus STEP CYCLE

A normal machine instruction has two visible phases:

```text
FETCH -> EXECUTE
```

`STEP INSTR` performs the whole instruction.

`STEP CYCLE` performs only the current phase. This is useful for watching:

1. memory feed the instruction register;
2. the decoder select an operation;
3. the datapath execute it.

The displayed `CYCLE` value increments for each FETCH and EXECUTE phase. Therefore a normal instruction usually adds two to the cycle count.

---

# 6. Terminal tab

The TERMINAL tab is the main interactive Forth workspace.

It contains:

- CPU STATE;
- INTERACTIVE FORTH TERMINAL;
- DATA STACK and RETURN STACK.

## 6.1 CPU STATE

The CPU state table displays:

| Register/state | Meaning |
|---|---|
| `PC` | Program counter - address of the next instruction/operand flow. |
| `IR` | Instruction register - current opcode word. |
| `T` | Top of data stack. |
| `N` | Next data-stack item. |
| `DSP` | Data-stack depth. |
| `RSP` | Return-stack depth. |
| `CYCLE` | Number of FETCH/EXECUTE phases performed. |
| `PHASE` | Current microcycle phase, normally FETCH or EXECUTE. |
| `HALTED` | Whether raw CPU execution is halted. |

Each numeric register is shown in decimal and balanced ternary.

## 6.2 Interactive terminal

Type a Forth line into the input box and press Enter.

Examples:

```forth
3 4 + .
```

```forth
10 DUP * .
```

```forth
: DOUBLE DUP + ;
```

Successful completed input prints:

```text
ok
```

If a colon definition is still being compiled, the terminal reports that compilation is continuing.

## 6.3 Command history

Inside the terminal input:

- Up Arrow - previous command;
- Down Arrow - later command;
- Ctrl+C - BREAK.

## 6.4 CLEAR

**CLEAR** clears only the terminal text log. It does not reset CPU, stacks, memory, definitions, or I/O.

## 6.5 QUEUE TEXT AS KEY INPUT

This button places the current terminal input text into the character queue consumed by:

```forth
KEY
```

or machine instruction:

```text
IN 0
```

This is different from executing the text as Forth.

If the input queue is empty, `KEY`/port 0 currently returns `0` instead of blocking.

## 6.6 Built-in examples

The example selector contains demonstrations such as:

- arithmetic;
- stack manipulation;
- square definition;
- countdown;
- recursive factorial;
- variables;
- ternary sign comparisons;
- three-way sign output;
- virtual motor;
- LED output.

Press **LOAD EXAMPLE** to place the selected example in the input field. Then press Enter to execute it.

---

# 7. Forth language reference

T9 implements a deliberately compact Forth-like dialect, not full ANS Forth.

## 7.1 Stack words

| Word | Stack effect | Meaning |
|---|---|---|
| `DUP` | `( a -- a a )` | Duplicate T. |
| `DROP` | `( a -- )` | Remove T. |
| `SWAP` | `( a b -- b a )` | Exchange T and N. |
| `OVER` | `( a b -- a b a )` | Copy N to T. |
| `ROT` | `( a b c -- b c a )` | Rotate the top three values. |
| `DEPTH` | `( -- n )` | Push current data-stack depth. |

`ROT` is compiled into the emulated kernel rather than implemented as a dedicated CPU opcode.

## 7.2 Arithmetic words

| Word | Stack effect | Meaning |
|---|---|---|
| `+` | `( a b -- a+b )` | Add. |
| `-` | `( a b -- a-b )` | Subtract T from N. |
| `*` | `( a b -- a*b )` | Multiply. |
| `NEGATE` | `( a -- -a )` | Negate a value. |
| `NEG` | `( a -- -a )` | Alias of NEGATE at the Forth level. |

All results normalize to the 9-trit range.

Current build does **not** implement `/` or `MOD`.

## 7.3 Comparisons

| Word | Stack effect | True when |
|---|---|---|
| `=` | `( a b -- flag )` | `a = b` |
| `<` | `( a b -- flag )` | `a < b` |
| `>` | `( a b -- flag )` | `a > b` |
| `0=` | `( a -- flag )` | `a = 0` |
| `0<` | `( a -- flag )` | `a < 0` |
| `0>` | `( a -- flag )` | `a > 0` |

T9 Forth uses:

```text
false = 0
true  = -1
```

Example:

```forth
7 3 > .
```

prints:

```text
-1
```

because the comparison is true.

## 7.4 Memory words

| Word | Stack effect | Meaning |
|---|---|---|
| `@` | `( addr -- value )` | Fetch one 9-trit word from memory. |
| `!` | `( value addr -- )` | Store one 9-trit word in memory. |

Memory addresses must be in the valid range:

```text
0..2186
```

Example:

```forth
VARIABLE X
42 X !
X @ .
```

prints `42`.

## 7.5 Return-stack words

| Word | Stack effect | Meaning |
|---|---|---|
| `>R` | `( x -- )` | Move x from data stack to return stack. |
| `R>` | `( -- x )` | Move x from return stack to data stack. |

These share the same return stack used by calls.

## 7.6 Character and number output

| Word | Stack effect | Meaning |
|---|---|---|
| `EMIT` | `( char -- )` | Emit a character to terminal output. |
| `KEY` | `( -- char )` | Read next queued input character code; returns 0 if queue empty. |
| `.` | `( n -- )` | Print signed decimal value followed by a space. |
| `CR` | `( -- )` | Emit newline character 10. |
| `SPACE` | `( -- )` | Emit space character 32. |

Example:

```forth
65 EMIT 66 EMIT 67 EMIT CR
```

prints:

```text
ABC
```

## 7.7 Dictionary and system words

| Word | Meaning |
|---|---|
| `:` | Begin a colon definition. |
| `;` | End a colon definition. |
| `VARIABLE` | Define a variable whose runtime behavior pushes its memory address. |
| `CONSTANT` | Pop a value now and define a word that pushes it later. |
| `WORDS` | Print known dictionary words in sorted order. |
| `DEPTH` | Push current data-stack depth. |
| `HALT` | Invoke the CPU HALT primitive. |
| `BYE` | Alias of HALT. |

---

# 8. Definitions, variables, constants, and control flow

## 8.1 Colon definitions

Syntax:

```forth
: NAME body ;
```

Example:

```forth
: DOUBLE DUP + ;
```

Then:

```forth
7 DOUBLE .
```

prints:

```text
14
```

Colon definitions are compiled into emulated memory beginning in the Forth dictionary region.

## 8.2 Recursive definitions

The compiler permits the word currently being defined to call itself.

Example:

```forth
: FACT
  DUP 1 >
  IF
    DUP 1 - FACT *
  ELSE
    DROP 1
  THEN
;
```

Then:

```forth
6 FACT .
```

prints:

```text
720
```

Be aware that recursion consumes return-stack space.

## 8.3 IF / ELSE / THEN

Syntax:

```forth
flag IF
  ...
THEN
```

or:

```forth
flag IF
  ...true path...
ELSE
  ...false path...
THEN
```

`IF` consumes the flag.

Zero is false. Any non-zero value takes the true path, although comparison words specifically produce `-1` for true.

Example:

```forth
: SIGNWORD
  DUP 0<
  IF
    DROP 78 EMIT
  ELSE
    DUP 0=
    IF
      DROP 90 EMIT
    ELSE
      DROP 80 EMIT
    THEN
  THEN
;
```

ASCII codes 78, 90 and 80 emit `N`, `Z` and `P`.

## 8.4 BEGIN / UNTIL

`BEGIN` marks a loop start.

`UNTIL` consumes a flag and branches back while that flag is zero.

Pattern:

```forth
BEGIN
  ...
  flag
UNTIL
```

Example:

```forth
: COUNTDOWN
  BEGIN
    DUP .
    1 -
    DUP 0=
  UNTIL
  DROP
;
```

Then:

```forth
5 COUNTDOWN
```

prints the countdown.

## 8.5 BEGIN / AGAIN

`AGAIN` always branches back to the matching `BEGIN`:

```forth
BEGIN
  ...
AGAIN
```

This is an infinite loop unless execution is interrupted by some other mechanism. Use **BREAK** or Ctrl+C if necessary.

## 8.6 Variables

Syntax:

```forth
VARIABLE NAME
```

A variable word pushes its address.

Example:

```forth
VARIABLE COUNT
10 COUNT !
COUNT @ .
```

## 8.7 Constants

`CONSTANT` consumes the current T value when the constant is created.

Example:

```forth
42 CONSTANT ANSWER
ANSWER .
```

The constant's value is stored in the JavaScript-managed dictionary metadata, while its runtime behavior is equivalent to pushing the value.

---

# 9. CPU tab

The CPU tab is for understanding what the emulated processor is doing beneath Forth.

## 9.1 Microarchitecture diagram

The diagram contains:

```text
PROGRAM MEMORY
      |
      v
     IR
      |
   DECODER
   /     \
  T       N
   \     /
  TERNARY ALU
      |
      v
 DATA STACK

RETURN STACK -> PC
```

The emulator highlights relevant nodes and paths according to the most recent operation.

Examples:

- `ADD` highlights T, N, ALU, and data-stack path;
- a memory operation highlights memory;
- `CALL`/`EXIT` can highlight the return stack and PC;
- FETCH highlights memory to IR.

The small status text beneath the diagram describes the last logical path, for example:

```text
T + N -> ALU -> T
```

## 9.2 ALU / CONDITION UNIT

This panel displays the ALU's most recent:

- A input;
- B input;
- result;
- operation.

It also shows the condition of T:

```text
NEGATIVE
ZERO
POSITIVE
```

## 9.3 Trit-by-trit carry trace

After an `ADD` or subtraction operation that uses addition internally, the emulator records a per-trit trace.

Columns include:

- trit position;
- A trit;
- B trit;
- carry in;
- sum trit;
- carry out.

Balanced-ternary addition constrains each sum and carry to `-1`, `0`, or `+1`.

Example local rule:

```text
+1 + +1 = -1 with carry +1
```

because:

```text
2 = (-1) + 3*(+1)
```

This panel is especially useful when translating the emulator's ALU into physical ternary logic.

---

# 10. Memory / ASM tab

This tab contains:

- memory inspector;
- assembler editor;
- disassembly preview;
- raw memory import/export.

## 10.1 Memory map

The default convention is:

| Address range | Intended use |
|---:|---|
| `0..255` | Assembly/program area |
| `256..383` | Interactive scratch area |
| `384..1791` | Compiled Forth dictionary |
| `1792..2186` | Variables/data |

This map is a convention rather than hard memory protection.

Advanced users should remember that the current compiler does not perform sophisticated collision detection if enormous definitions grow into the variable area.

## 10.2 Memory inspector columns

The memory table shows:

- address in decimal;
- address in balanced ternary;
- stored value in decimal and editable form;
- stored value in balanced ternary;
- possible opcode decode.

The current PC is highlighted when visible.

Use **Jump** and **GO** to inspect another address range.

The table renders a window of memory rather than all 2187 cells simultaneously.

## 10.3 Editing memory

Change the value in the DEC/input field and leave the field to commit the change.

Decimal entry is recommended for reliability.

Direct edits are immediate and can alter:

- programs;
- operands;
- Forth definitions;
- variables;
- kernel code.

There is no protection against editing live program structures.

## 10.4 Decode-column caveat

The memory inspector can label a cell as an opcode whenever its numeric value matches an opcode code.

An operand can coincidentally have the same numeric value as an opcode. Therefore the DECODE column is a convenience, not a full control-flow-aware disassembly of arbitrary memory.

Use the assembler's disassembly preview for an intentionally assembled program.

---

# 11. Machine instruction set

The instruction opcode is exactly 3 trits. There are therefore exactly:

```text
3^3 = 27
```

possible opcode patterns.

T9 assigns every pattern, with one pattern currently reserved.

`000` is deliberately `NOP`, so zero-filled memory is inert rather than automatically becoming a branch or destructive instruction.

## 11.1 Opcode table

| Trits | Code | Instruction | Operand | Effect / meaning |
|---|---:|---|---|---|
| `000` | 0 | `NOP` | no | Do nothing. |
| `00+` | 1 | `LIT` | next word | Push operand. |
| `0+-` | 2 | `DUP` | no | `( a -- a a )` |
| `0+0` | 3 | `DROP` | no | `( a -- )` |
| `0++` | 4 | `SWAP` | no | `( a b -- b a )` |
| `+--` | 5 | `OVER` | no | `( a b -- a b a )` |
| `+-0` | 6 | `ADD` | no | `( a b -- a+b )` |
| `+-+` | 7 | `SUB` | no | `( a b -- a-b )` |
| `+0-` | 8 | `NEG` | no | `( a -- -a )` |
| `+00` | 9 | `MUL` | no | `( a b -- a*b )` |
| `+0+` | 10 | `FETCH` | no | `( addr -- value )` |
| `++-` | 11 | `STORE` | no | `( value addr -- )` |
| `++0` | 12 | `BRANCH` | next word | PC = target. |
| `+++` | 13 | `ZBRANCH` | next word | Pop x; branch if x = 0. |
| `---` | -13 | `NBRANCH` | next word | Pop x; branch if x < 0. |
| `--0` | -12 | `PBRANCH` | next word | Pop x; branch if x > 0. |
| `--+` | -11 | `CALL` | next word | Push return PC; jump to target. |
| `-0-` | -10 | `EXIT` | no | Pop return address into PC. |
| `-00` | -9 | `TOR` | no | Move T to return stack (`>R`). |
| `-0+` | -8 | `RFROM` | no | Move return-stack top to data stack (`R>`). |
| `-+-` | -7 | `EMIT` | no | Pop character code to terminal. |
| `-+0` | -6 | `KEY` | no | Push next input character code. |
| `-++` | -5 | `IN` | next word | Push value from numbered input port. |
| `0--` | -4 | `OUT` | next word | Pop value to numbered output port. |
| `0-0` | -3 | `HALT` | no | Halt raw CPU execution. |
| `0-+` | -2 | `CMP` | no | `( a b -- -1/0/+1 )` according to a vs b. |
| `00-` | -1 | `RESERVED` | no | Deliberately invalid for execution. |

## 11.2 Instructions with operand cells

These instructions consume the following memory word as an operand:

```text
LIT
BRANCH
ZBRANCH
NBRANCH
PBRANCH
CALL
IN
OUT
```

Example:

```text
address 0: LIT
address 1: 42
address 2: OUT
address 3: 8
address 4: HALT
```

The operand is data, not another instruction.

## 11.3 Conditional branches consume the tested value

`ZBRANCH`, `NBRANCH`, and `PBRANCH` pop the value they test.

This behavior is used by the Forth compiler for `IF`, `UNTIL`, and comparison macros.

## 11.4 CALL and EXIT

`CALL target`:

1. reads its target operand;
2. pushes the address after the operand onto the return stack;
3. loads target into PC.

`EXIT`:

1. pops the return stack;
2. loads the popped value into PC.

A raw `EXIT` with an empty return stack raises an error.

The interactive Forth layer uses a special internal return sentinel when it calls compiled words from JavaScript. That implementation detail lets a compiled definition return cleanly to the outer interpreter.

---

# 12. Assembler syntax

The assembler is intentionally small.

## 12.1 Basic program

```asm
LIT 3
LIT 4
ADD
OUT 8
HALT
```

This pushes 3 and 4, adds them, sends the result to decimal-number output port 8, and halts.

## 12.2 Comments

A semicolon starts a comment:

```asm
LIT 42     ; put 42 on the stack
OUT 8      ; print it as a decimal number
HALT
```

## 12.3 Labels

Labels use a colon:

```asm
start:
    LIT 5
    BRANCH start
```

Labels can be used where an address operand is expected.

## 12.4 Aliases

The assembler accepts several Forth-like aliases:

| Alias | Machine instruction |
|---|---|
| `@` | `FETCH` |
| `!` | `STORE` |
| `0BRANCH` | `ZBRANCH` |
| `>R` | `TOR` |
| `R>` | `RFROM` |
| `NEGATE` | `NEG` |
| `+` | `ADD` |
| `-` | `SUB` |
| `*` | `MUL` |

## 12.5 ASSEMBLE

**ASSEMBLE** parses the editor and updates the disassembly preview but does not intentionally replace the running program in memory.

Assembler errors are shown with line information when possible.

## 12.6 ASSEMBLE + LOAD @ 0

This assembles the source and writes the resulting words starting at memory address 0.

It also positions the CPU for execution at address 0 without automatically running continuously.

## 12.7 LOAD + RUN

This:

1. assembles the source;
2. loads it at address 0;
3. resets the CPU;
4. starts execution from address 0.

Use this for self-contained assembly programs.

## 12.8 Disassembly preview

The preview displays lines such as:

```text
   0  00000000+  LIT 3
   2  00000000+  LIT 4
   4  000000+-0  ADD
```

The raw field is the 9-trit stored opcode word, while the mnemonic is decoded from its 3-trit opcode value.

---

# 13. I/O tab

T9 uses numbered abstract I/O ports.

## 13.1 Port map

| Port | Direction | Emulator peripheral |
|---:|---|---|
| 0 | input | Keyboard/key queue |
| 1 | output | Terminal character output |
| 2 | output | LED bank |
| 3 | output | Beeper |
| 4 | input | Three-state ternary switch |
| 5 | input | Generic sensor slider |
| 6 | output | Motor direction |
| 7 | output | Motor power |
| 8 | output | Signed decimal number output |

The machine instructions `IN` and `OUT` use a port number in the following memory cell.

## 13.2 Port 0 - key queue

Characters can be queued from:

- the terminal's **QUEUE TEXT AS KEY INPUT** button;
- the I/O tab's key-queue text area and **QUEUE** button.

`KEY` or `IN 0` consumes one queued character code.

If the queue is empty, the current emulator returns `0`.

## 13.3 Port 1 - terminal character output

The CPU instruction `EMIT` is the normal Forth interface to this output.

Example:

```forth
84 EMIT 57 EMIT CR
```

prints:

```text
T9
```

## 13.4 Port 2 - LED bank

The emulator renders nine LEDs.

The current visualization intentionally treats the absolute output value as a **bit-like lamp pattern for convenience**. This LED widget is therefore not a pure ternary display.

In Forth, the convenience word is:

```forth
OUT2
```

Example:

```forth
17 OUT2
```

## 13.5 Port 3 - beeper

Output to port 3 triggers the virtual beeper.

Forth convenience word:

```forth
OUT3
```

The I/O panel also has a manual **BEEP** button.

## 13.6 Port 4 - ternary switch

The switch has three states:

```text
-1   0   +1
```

Forth convenience word:

```forth
IN4
```

Example:

```forth
IN4 .
```

## 13.7 Port 5 - generic sensor

The sensor slider ranges from:

```text
-100 to +100
```

Forth convenience word:

```forth
IN5
```

Example:

```forth
IN5 0< .
```

## 13.8 Port 6 - motor direction

The sign of the output controls the virtual motor:

```text
negative -> REVERSE
zero     -> STOP
positive -> FORWARD
```

Forth convenience word:

```forth
OUT6
```

Example:

```forth
-1 OUT6
0 OUT6
1 OUT6
```

This is a particularly natural use of balanced ternary.

## 13.9 Port 7 - motor power

Port 7 uses the absolute magnitude of the output and clamps it to 0-100 percent.

Forth convenience word:

```forth
OUT7
```

Example:

```forth
75 OUT7
```

## 13.10 Port 8 - decimal number output

Port 8 prints a signed machine word in decimal followed by a space.

The Forth word:

```forth
.
```

is compiled to this port.

In assembly:

```asm
OUT 8
```

---

# 14. Physical / Logic tab

This tab is intended to connect the logical emulator to a future physical computer design.

It does **not** simulate real analog electronics.

## 14.1 Simulated voltage rails

By default, logical trits are mapped conceptually to:

```text
-1 -> -5 V
 0 ->  0 V
+1 -> +5 V
```

The voltage values are editable.

The panel shows the nine trits of:

- T;
- N;
- PC;
- ALU result;
- IR.

This helps answer questions such as:

- Which physical rails would be active for this machine word?
- How many ternary signal lines are required?
- Which signal must a future level-restorer reproduce?

Do not use this panel to predict transistor currents, noise margins, diode drops, or real propagation delay.

## 14.2 Ternary logic lab

Choose one trit for A and one for B.

The panel displays:

- `NEG A`;
- `MIN(A,B)`;
- `MAX(A,B)`;
- comparison;
- balanced-ternary sum trit;
- carry trit.

This is useful for deriving truth tables for physical logic.

## 14.3 Retro front panel

The front panel displays 9-trit lamp rows for:

- T;
- N;
- PC.

It also duplicates basic controls:

- RUN;
- STOP;
- STEP;
- RESET.

## 14.4 Manual 9-trit word input

Nine three-state buttons let you construct a machine word.

Each button cycles through trit states. The displayed decimal and ternary value updates as you change the word.

**ENTER -> DATA STACK** pushes the completed word directly onto the data stack.

This is a convenience front-panel action. It does not emulate a memory-deposit sequence or a specific future hardware bus protocol.

## 14.5 Calculator / Forth keypad

The virtual keypad models a possible future repurposed calculator keyboard.

It contains direct keys for:

```text
DUP  DROP  SWAP  OVER
7    8     9     +
4    5     6     -
1    2     3     *
0    .     :     ENTER
```

The keys insert text into the same interactive terminal input used by the computer keyboard.

## 14.6 Telephone keypad

The virtual telephone keypad uses:

```text
1 2 3
4 5 6
7 8 9
* 0 #
```

`#` acts as Enter.

`*` is a command modifier. The current map is:

| Sequence | Forth text inserted |
|---|---|
| `*1` | `DUP` |
| `*2` | `DROP` |
| `*3` | `SWAP` |
| `*4` | `+` |
| `*5` | `-` |
| `*6` | `*` |
| `*7` | `@` |
| `*8` | `!` |
| `*9` | `.` |

This is an input experiment, not yet a definitive physical keyboard protocol.

---

# 15. Architecture / Tests tab

## 15.1 Architecture section

The built-in architecture section summarizes:

- word size;
- opcode design;
- memory map;
- division of labor between CPU and Forth outer interpreter;
- three-way conditions;
- implemented Forth subset;
- physical intent.

Use it as the emulator's compact specification. Use this manual for operating detail.

## 15.2 Self-tests

Press **RUN SELF TESTS**.

The current suite checks representative behaviors including:

- integer -> ternary -> integer round trip;
- minimum and maximum word values;
- zero;
- addition;
- subtraction;
- negation;
- balanced-ternary carry propagation;
- wraparound at both numeric limits;
- DUP and SWAP;
- memory read/write;
- CALL/EXIT;
- zero, negative, and positive branches;
- assembler/disassembler behavior;
- basic Forth arithmetic;
- colon-definition execution.

A green all-pass result is a useful smoke test before debugging your own program.

---

# 16. Saving, loading, importing, and exporting

State tools are under **ARCHITECTURE / TESTS -> STATE / PERSISTENCE**.

## 16.1 SAVE LOCAL

Stores a machine snapshot in the browser's `localStorage` under the T9 state key.

The snapshot includes:

- memory;
- CPU state;
- data and return stacks;
- Forth dictionary metadata;
- I/O state;
- terminal log.

This save belongs to the browser/profile in which the emulator is running.

## 16.2 LOAD LOCAL

Restores the state previously stored by SAVE LOCAL.

## 16.3 EXPORT STATE

Downloads a JSON representation of the complete emulator state.

Use this when you want a portable checkpoint independent of browser local storage.

## 16.4 IMPORT STATE

Loads a previously exported state JSON file.

The current state format expects version 1.

## 16.5 EXPORT TERMINAL LOG

Downloads the terminal transcript as a plain-text file.

This is useful for:

- documenting experiments;
- preserving Forth sessions;
- comparing behavior before and after emulator changes.

## 16.6 EXPORT MEMORY

Under **MEMORY / ASM**, this downloads only the memory image as a JSON array of signed integer words.

This does not include dictionary metadata, CPU registers, stack state, or terminal history.

## 16.7 IMPORT MEMORY

Loads a JSON memory array into the emulator's 2187-word memory.

Use complete state export/import when you need Forth dictionary metadata to remain synchronized with compiled definitions.

A raw memory image alone does not reconstruct JavaScript-side dictionary names.

---

# 17. Worked examples

## 17.1 Example A - use T9 as an RPN calculator

Enter:

```forth
20 5 - .
```

Stack behavior:

```text
20 5
SUB
15
```

Output:

```text
15
```

Now try:

```forth
7 8 * .
```

Result:

```text
56
```

## 17.2 Example B - define a reusable word

```forth
: CUBE DUP DUP * * ;
```

Then:

```forth
4 CUBE .
```

Result:

```text
64
```

Watch the DATA STACK while single-stepping the compiled definition from the CPU side if you want to see the sequence of stack operations.

## 17.3 Example C - variables

```forth
VARIABLE SCORE
100 SCORE !
SCORE @ .
```

Result:

```text
100
```

Increment it:

```forth
SCORE @ 1 + SCORE !
SCORE @ .
```

Result:

```text
101
```

## 17.4 Example D - ternary sign decision

Use the virtual switch on I/O port 4.

Set it to negative, zero, or positive, then execute:

```forth
IN4 .
```

You will receive `-1`, `0`, or `1`.

A control word can map this directly to motor direction:

```forth
: SWITCH>MOTOR IN4 OUT6 ;
```

Now change the switch and run:

```forth
SWITCH>MOTOR
```

The motor display becomes REVERSE, STOP, or FORWARD.

This demonstrates a clean ternary control path:

```text
input sign -> ternary machine value -> motor sign
```

## 17.5 Example E - sensor threshold

Move the sensor slider on port 5.

Then:

```forth
IN5 0< .
```

The result is:

```text
-1
```

when the sensor is negative and `0` otherwise.

You can define:

```forth
: SENSOR-SIGN
  IN5
  DUP 0< IF DROP -1
  ELSE
    DUP 0= IF DROP 0
    ELSE DROP 1
    THEN
  THEN
;
```

Then:

```forth
SENSOR-SIGN .
```

## 17.6 Example F - character output

```forth
72 EMIT 69 EMIT 76 EMIT 76 EMIT 79 EMIT CR
```

Output:

```text
HELLO
```

## 17.7 Example G - queued keyboard input

Type:

```text
ABC
```

into the I/O key-queue text area and press QUEUE.

Then in Forth:

```forth
KEY EMIT KEY EMIT KEY EMIT CR
```

This consumes and re-emits the queued characters.

## 17.8 Example H - raw assembly

Open **MEMORY / ASM** and enter:

```asm
; Calculate 12 * 12 and print the result
LIT 12
DUP
MUL
OUT 8
HALT
```

Press **LOAD + RUN**.

The terminal should receive:

```text
144
```

The machine then enters HALTED state.

## 17.9 Example I - inspect an assembly instruction cycle by cycle

Load this program without running it:

```asm
LIT 3
LIT 4
ADD
HALT
```

Use **ASSEMBLE + LOAD @ 0**.

Then use **STEP CYCLE** repeatedly.

You should see a pattern like:

```text
FETCH LIT opcode
EXECUTE LIT, reading operand 3
FETCH LIT opcode
EXECUTE LIT, reading operand 4
FETCH ADD opcode
EXECUTE ADD
```

Watch:

- PC;
- IR;
- PHASE;
- T and N;
- microarchitecture highlighting;
- ALU result;
- carry trace.

## 17.10 Example J - observe wraparound

In the Forth terminal:

```forth
9841 1 + .
```

Expected:

```text
-9841
```

This demonstrates native 9-trit normalization.

---

# 18. Debugging and troubleshooting

## 18.1 `Data stack underflow`

A word tried to consume more data values than were available.

Example:

```forth
+
```

requires two values but the stack may be empty.

Fix: inspect DATA STACK and provide the expected inputs.

## 18.2 `Return stack underflow`

Typical causes:

- executing `R>` with nothing on the return stack;
- executing a raw `EXIT` without a corresponding CALL;
- corrupting return-stack discipline in a definition.

## 18.3 `Data stack overflow` or `Return stack overflow`

Each software stack currently permits a maximum of 512 entries.

Likely causes include:

- a loop that keeps pushing without dropping;
- runaway recursion;
- unbalanced `>R` use.

## 18.4 `Invalid memory address`

Valid addresses are:

```text
0..2186
```

Check the address before `@`, `!`, branch, or call operations.

## 18.5 `Invalid opcode`

The CPU fetched a word that is not a valid executable opcode, including the reserved opcode.

Possible causes:

- PC jumped into operand/data memory;
- program was edited incorrectly;
- a branch target is wrong;
- memory and dictionary metadata are no longer synchronized.

## 18.6 `Unknown Forth word`

The outer interpreter could not find the token in its current dictionary.

Use:

```forth
WORDS
```

Check spelling and whether a definition completed successfully.

## 18.7 `Unknown Forth word while compiling`

The compiler encountered an undefined token inside a colon definition.

The incomplete definition remains a compilation problem. Correct the source or hard reset if you want to return to a known-clean state.

## 18.8 `ELSE without IF`, `THEN without IF/ELSE`, or unresolved control structure

The compile-time control-flow stack is unbalanced.

Check the nesting of:

```forth
IF ELSE THEN
BEGIN UNTIL
BEGIN AGAIN
```

## 18.9 Program appears to run forever

Press:

- **BREAK**;
- or Ctrl+C.

Continuous machine RUN is intentionally capable of executing loops indefinitely, but it is batched through browser animation frames to reduce the chance of freezing the UI.

Interactive CPU calls from the Forth layer also have a runaway execution limit.

## 18.10 STEP does nothing after a halt

`STEP INSTR` and `STEP CYCLE` do not automatically clear a halted CPU.

Options:

- use **RESET** to restart at PC 0 with clean stacks;
- use **RUN** if you intentionally want to clear the halted state and continue from the current PC.

For deterministic debugging after HALT, RESET and then single-step from the desired program start.

## 18.11 RUN produced nonsense while I was using Forth

The terminal is already an active interactive environment. RUN executes raw memory from PC.

If you had no program loaded at PC, the CPU may have walked through blank memory, scratch space, compiled dictionary cells, or operands.

Use RESET, then continue using the Forth terminal normally.

## 18.12 A memory cell is decoded as an instruction even though it is data

The memory table's DECODE column is local and value-based. An operand such as `6` has the same numeric value as the `ADD` opcode.

This does not mean that the CPU will execute it unless PC reaches that cell as an instruction.

## 18.13 Raw memory import lost my word names

The Forth dictionary contains JavaScript-side metadata mapping names to compiled addresses and word types.

A memory-only JSON file does not contain that metadata.

Use **EXPORT STATE / IMPORT STATE** when preserving a full interactive Forth system.

---

# 19. What is emulated in hardware and what is handled by JavaScript

This distinction is central to T9.

## 19.1 CPU-level behavior

The following are represented as operations of the emulated machine:

- stack arithmetic;
- `DUP`, `DROP`, `SWAP`, `OVER`;
- `ADD`, `SUB`, `NEG`, `MUL`, `CMP`;
- memory fetch/store;
- machine branches;
- CALL/EXIT;
- return-stack transfer;
- EMIT/KEY;
- IN/OUT;
- HALT;
- execution of compiled colon definitions;
- compiled control-flow branches.

## 19.2 Outer-interpreter/compiler behavior

JavaScript currently performs tasks that would need a monitor, interpreter, compiler, or operating environment on a fully self-hosting physical machine:

- tokenizing typed Forth source;
- dictionary name lookup;
- compilation directives;
- allocating `VARIABLE` names;
- creating `CONSTANT` dictionary entries;
- `WORDS` listing;
- compile-time control-flow patching for `IF`, `ELSE`, `THEN`, `BEGIN`, `UNTIL`, `AGAIN`.

The important design boundary is that a colon definition's executable body becomes actual T9 machine code in emulated memory.

## 19.3 Why this boundary exists

The emulator is intended first to validate:

- the ternary datapath;
- instruction encoding;
- stack architecture;
- branching;
- memory behavior;
- I/O model.

A later version could move more of the outer interpreter onto the emulated machine itself, eventually approaching a self-hosted Forth system.

---

# 20. Using the emulator to design a physical T9

The emulator is most valuable when treated as an executable specification for future hardware.

## 20.1 Start from the opcode decoder

There are exactly 27 3-trit opcode patterns.

A physical decoder must distinguish patterns such as:

```text
000 -> NOP
00+ -> LIT
+-0 -> ADD
--- -> NBRANCH
```

A diode matrix is a natural candidate for portions of this decoding/control logic, provided signal restoration is added where required.

## 20.2 Use the logic lab to derive ternary truth tables

For each proposed physical gate or ALU cell, compare its expected output with the emulator.

In particular, verify:

- trit negation;
- sum trit;
- carry trit;
- comparison behavior.

## 20.3 Use STEP CYCLE as the timing specification

The current logical CPU exposes two major phases:

```text
FETCH
EXECUTE
```

A first physical implementation can preserve this simplicity even if a later design divides operations into more hardware clock phases.

## 20.4 Treat the voltage panel as notation, not circuit proof

The -5/0/+5 V mapping is a conceptual signal representation.

A real machine still requires engineering for:

- threshold detection;
- voltage drops;
- restoration/gain;
- fan-out;
- noise margins;
- clocking;
- storage;
- power supply stability;
- I/O protection.

Do not infer that three ideal rails alone constitute a complete ternary logic family.

## 20.5 Prototype peripherals independently

The port model gives a clean boundary for external hardware.

A physical system could implement separate modules for:

```text
port 0  keyboard interface
port 1  character terminal
port 2  indicators
port 3  audio
port 4  ternary switch input
port 5  ADC/sensor adapter
port 6  H-bridge direction control
port 7  power/PWM controller
```

The physical electrical interface does not need to expose raw ternary rails to every external device. Binary or analog peripheral circuitry can be translated at the I/O boundary while the CPU remains logically ternary.

## 20.6 Preserve the emulator as the reference model

When a physical module is built, test it against the same vectors used in T9.

For example, for a 1-trit full adder, enumerate all:

```text
A     = -1,0,+1
B     = -1,0,+1
Cin   = -1,0,+1
```

That is only:

```text
3^3 = 27
```

input combinations.

Compare physical sum/carry outputs against the emulator's balanced-ternary rules before scaling to 9 trits.

---

# 21. Current limitations

The current build intentionally leaves several features out.

## 21.1 Forth limitations

Not implemented:

- `/`;
- `MOD`;
- `DO` / `LOOP`;
- `WHILE` / `REPEAT`;
- strings;
- `CREATE` / `DOES>`;
- pictured numeric output;
- full ANS Forth compatibility;
- persistent block/file words.

## 21.2 CPU/architecture limitations

- `MUL` is currently a direct hardware primitive in the logical ISA rather than a Forth-only synthesized operation.
- Memory addresses are ordinary non-negative JavaScript indices even though they are displayed as machine words.
- The logical microarchitecture has only FETCH and EXECUTE phases; it is not a transistor-level timing model.
- The physical voltage view is conceptual only.
- The LED display on port 2 uses a bit-like magnitude visualization rather than nine independent ternary lamps.

## 21.3 Forth implementation limitations

- `VARIABLE`, `CONSTANT`, `WORDS`, and `DEPTH` rely partly or wholly on the JavaScript outer interpreter.
- Dictionary name metadata is outside raw emulated memory.
- Very large dictionary growth is not protected by a full memory manager.
- The current ternary text-literal parser has input edge cases; decimal input is safer.

## 21.4 Terminal limitations

- `KEY` does not block; it returns 0 if no character is queued.
- No full-screen terminal control language is implemented.
- No direct serial/USB hardware is connected from the browser version.

These limitations are useful boundaries for future versions rather than reasons to complicate the first physical CPU.

---

# 22. Quick reference

## 22.1 Essential Forth session

```forth
3 4 + .
: DOUBLE DUP + ;
12 DOUBLE .
VARIABLE X
42 X !
X @ .
WORDS
```

## 22.2 Essential stack words

```text
DUP   ( a -- a a )
DROP  ( a -- )
SWAP  ( a b -- b a )
OVER  ( a b -- a b a )
ROT   ( a b c -- b c a )
```

## 22.3 Essential arithmetic

```text
+       add
-       subtract
*       multiply
NEGATE  negate
```

All arithmetic wraps to `-9841..+9841`.

## 22.4 Essential comparisons

```text
=   <   >   0=   0<   0>
```

```text
false = 0
true  = -1
```

## 22.5 Memory

```text
@   ( addr -- value )
!   ( value addr -- )
```

## 22.6 Control flow

```forth
flag IF ... THEN
flag IF ... ELSE ... THEN
BEGIN ... flag UNTIL
BEGIN ... AGAIN
```

## 22.7 I/O convenience words

```text
EMIT  character output
KEY   queued character input
.     decimal numeric output
IN4   ternary switch input
IN5   sensor input
OUT2  LED bank
OUT3  beeper
OUT6  motor direction
OUT7  motor power
```

## 22.8 CPU operation

```text
RUN         raw machine execution from PC
PAUSE       stop continuous execution
STEP INSTR  one complete instruction
STEP CYCLE  one FETCH or EXECUTE phase
RESET       reset CPU/stacks; keep memory/dictionary
HARD RESET  rebuild clean machine
BREAK       stop and mark HALTED
```

## 22.9 Memory map

```text
0..255      assembly/program area
256..383    scratch
384..1791   compiled Forth dictionary
1792..2186  variables/data
```

## 22.10 Word format

```text
9 trits
3^9 = 19683 patterns
range = -9841..+9841
```

## 22.11 Opcode format

```text
3 trits
3^3 = 27 opcode patterns
000 = NOP
00- = RESERVED
```

---

# Appendix A - Suggested learning sequence

For a first serious session, use this order:

1. Run the self-tests.
2. Enter `3 4 + .`.
3. Enter `1 2 3` and watch the DATA STACK.
4. Try `DUP`, `DROP`, `SWAP`, `OVER`, and `ROT` one at a time.
5. Define `: SQUARE DUP * ;`.
6. Inspect the compiled dictionary around memory address 384.
7. Load a five-instruction assembly program at address 0.
8. RESET and STEP CYCLE through it.
9. Watch T, N, PC, IR, PHASE, and the microarchitecture diagram.
10. Perform an ADD and inspect the trit carry trace.
11. Change the ternary switch and read it with `IN4`.
12. Connect it logically to the motor with `: SWITCH>MOTOR IN4 OUT6 ;`.
13. Open the physical voltage panel and inspect the nine rail states.
14. Export the complete state before making destructive memory edits.

This sequence moves from **using Forth**, through **understanding the CPU**, to **thinking about physical implementation**.

# Appendix B - Minimal assembly debugging checklist

Before pressing RUN on a hand-written assembly program, verify:

- Does execution begin at the PC you expect?
- Does every operand-bearing instruction have a following word?
- Are branch and call targets valid addresses?
- Does every CALL path eventually execute EXIT?
- Is EXIT guaranteed to have a return address?
- Do conditional branches receive a value to pop?
- Do stack operations have enough operands?
- Are memory addresses within `0..2186`?
- Is there a HALT or deliberate loop at the end?
- Have you exported state if the current Forth dictionary matters?

For a new program, prefer **ASSEMBLE + LOAD @ 0**, then use **STEP INSTR** or **STEP CYCLE** before continuous RUN.

# Appendix C - Minimal physical-hardware interpretation

A future physical T9 can be thought of as six blocks:

```text
+-------------------+
| PROGRAM / MEMORY  |
+---------+---------+
          |
          v
+-------------------+
| IR + 3-TRIT       |
| OPCODE DECODER    |
+---------+---------+
          |
          v
+---------+---------+      +----------------+
| T / N + DATA      |<---->| TERNARY ALU    |
| STACK STORAGE     |      +----------------+
+---------+---------+
          |
          +--------------------+
                               |
+-------------------+          |
| RETURN STACK + PC |<---------+
+-------------------+
          |
          v
+-------------------+
| NUMBERED I/O      |
+-------------------+
```

The emulator can therefore serve as a reference for progressively replacing simulated blocks with real ones without changing the whole machine at once.

