# Stack Virtual Machine Specification

This document specifies a stack-based virtual machine (VM) designed for executing bytecode instructions. The primary application of this VM is to perform operations on an array, with the ultimate goal of "annihilating" it (e.g., reducing it to an empty state).

## 1. State

The virtual machine's state is composed of the following components:

*   **Data Stack:** A LIFO (Last-In, First-Out) stack used for storing temporary values and passing arguments to and from instructions. Most operations will pop operands from this stack and push results back onto it. If an operation requires more operands than are available on the stack, the machine will halt with an error.

*   **Return Stack:** A separate stack used to store return addresses for function calls. This allows for nested function calls and proper execution flow.

*   **Instruction Pointer (IP):** A register that holds the memory address of the next bytecode instruction to be executed. The IP is automatically incremented after each instruction is fetched, unless a control flow instruction (like a jump or call) modifies it directly.

*   **Main Memory (Array Space):** A region of memory specifically allocated for storing the array that the VM will operate on. Instructions for array manipulation will interact with this memory space. For the purpose of this specification, we can assume this is a contiguous block of memory that can hold a single array of integers.
## 2. Instruction Set (Bytecode)

The following is a list of bytecode instructions that the virtual machine can execute. Each instruction is represented by a single byte.

### Stack Manipulation
- `0x01` **PUSH** `value`: Pushes a 32-bit integer `value` onto the data stack. The `value` is the next 4 bytes in the bytecode stream.
- `0x02` **POP**: Pops the top value from the data stack and discards it.
- `0x03` **DUP**: Duplicates the top value on the data stack.

### Arithmetic Operations
- `0x10` **ADD**: Pops the top two values from the data stack, adds them, and pushes the result back onto the stack.
- `0x11` **SUB**: Pops the top two values from the data stack, subtracts the top value from the second-to-top value, and pushes the result back onto the stack.

### Control Flow
- `0x20` **JMP** `addr`: Sets the Instruction Pointer (IP) to the specified `addr`.
- `0x21` **JZ** `addr`: Pops the top value from the data stack. If the value is zero, sets the IP to `addr`.
- `0x22` **CALL** `addr`: Pushes the current IP onto the return stack, then sets the IP to `addr`.
- `0x23` **RET**: Pops the return address from the return stack and sets the IP to it.

### Array Operations
- `0x30` **NEW_ARRAY** `size`: Creates a new array of the given `size` in the main memory. The `size` is popped from the data stack.
- `0x31` **PUSH_ARRAY** `index`: Pushes the value at the specified `index` of the array onto the data stack. The `index` is popped from the data stack.
- `0x32` **POP_ARRAY** `index`: Pops a value from the data stack and stores it at the specified `index` in the array. The `index` is also popped from the data stack.
- `0x33` **LEN_ARRAY**: Pushes the current size of the array onto the data stack.
- `0x34` **RESIZE_ARRAY**: Resizes the array to a new size. The new size is popped from the data stack.

### Halt
- `0xFF` **HALT**: Stops the execution of the virtual machine.
## 3. Execution Cycle

The virtual machine operates on a continuous fetch-decode-execute cycle. The cycle is as follows:

1.  **Fetch**: The VM fetches the bytecode instruction located at the address specified by the Instruction Pointer (IP).
2.  **Decode**: The VM decodes the instruction to determine the operation to be performed and any required operands.
3.  **Increment IP**: The IP is incremented to point to the next instruction in the bytecode stream. If the current instruction has operands that are part of the bytecode stream (like the `value` for `PUSH`), the IP is incremented past these operands as well.
4.  **Execute**: The VM executes the decoded instruction. This may involve manipulating the data stack, accessing memory, or changing the IP (in the case of control flow instructions).
5.  **Repeat**: The cycle repeats by returning to the fetch step, unless a `HALT` instruction is executed, in which case the VM stops.
## 4. Example Program: Array Annihilation

This example program demonstrates how to "annihilate" an array by repeatedly shrinking it until it is empty. The program starts with an array of size 5, and then enters a loop that reduces the size of the array by 1 in each iteration until the array is empty.

**Bytecode:**
`0x01 0x05 0x00 0x00 0x00` // PUSH 5
`0x30`                   // NEW_ARRAY
`0x33`                   // LEN_ARRAY (loop start at 0x06)
`0x21 0x19 0x00 0x00 0x00` // JZ 0x19 (if len == 0, jump to HALT)
`0x33`                   // LEN_ARRAY
`0x01 0x01 0x00 0x00 0x00` // PUSH 1
`0x11`                   // SUB
`0x34`                   // RESIZE_ARRAY
`0x20 0x06 0x00 0x00 0x00` // JMP 0x06
`0xFF`                   // HALT (at address 0x19)

**Assembly-like representation:**

```assembly
  ; Address 0x00
  PUSH 5       ; Push the initial size of the array onto the stack
  ; Address 0x05
  NEW_ARRAY    ; Create a new array of size 5

; Address 0x06
LOOP_START:
  LEN_ARRAY    ; Get the current length of the array
  ; Address 0x07
  JZ HALT      ; If the length is 0, jump to HALT (address 0x19)
  ; Address 0x0C
  LEN_ARRAY    ; Get the length again for the subtraction
  ; Address 0x0D
  PUSH 1       ; Push 1 to subtract from the length
  ; Address 0x12
  SUB          ; Subtract 1 from the length
  ; Address 0x13
  RESIZE_ARRAY ; Resize the array to the new length
  ; Address 0x14
  JMP LOOP_START ; Jump back to the start of the loop (address 0x06)

; Address 0x19
HALT:
  HALT         ; Stop the machine
```
