# MTRX2700 Project 1

## Group Members:
Minh, Joshua, Pranav, Julian


## Roles and Responsibilities
Pranav: Exercise 1, Exercise 5

Joshua: Exercise 2, Exercise 5, Meeting Minutes

Minh: Exercise 3, Exercise 5

Julian: Exercise 4, Exercise 5


## Project Overview


## Exercise 1 - One Line Descriptor

### Summary
A string is defined in RAM, its length is measured, its case is converted, and it is packaged into a structured communication frame with a BCC XOR checksum A bonus CRC32 implementation uses the STM32F303's built-in hardware CRC peripheral as a more better alternative to BCC.

The program runs sequentially through all tasks on startup and then loops forever. Execution order in main.s:
string_length    → measures "Hello World"
string_case      → converts it to "HELLO WORLD" in RAM
build_buffer     → builds [STX][LEN][string][ETX][BCC] in RAM
verify_checksum  → confirms BCC is correct
crc32_init       → enables the STM32 CRC hardware clock
crc32_calculate  → computes CRC32 of "Hello World" in hardware
crc32_verify     → confirms CRC32 matches the stored value
forever_loop     → infinite loop, program stays alive

### Usage
Build and flash the project using the IDE. The program produces no visible output — correctness is verified by pausing execution in the debugger at breakpoints after each function call and inspecting register values and memory contents. No button presses or serial connection are required.

### Valid input
- All string inputs must be NUL-terminated (0x00 at the end). Without a NUL byte the length and copy loops will run indefinitely into adjacent     memory.
- string_case expects R2 to be exactly 0 (uppercase) or 1 (lowercase).
- build_buffer requires the destination buffer to be at least string_length + 4 bytes. Writing beyond the allocated .space 256 will silently corrupt adjacent memory.
- calc_checksum and crc32_calculate expect R2 to be greater than zero. A length of zero returns 0x00 immediately as an edge case.
- verify_checksum must receive the total buffer length including the BCC byte, not just the frame length. Passing the wrong length produces an incorrect result without any error.
- crc32_verify must receive the total buffer length including all 4 CRC bytes at the end.
- All pointer registers (R0, R1) must point to valid, accessible RAM addresses. Pointing into flash for a write operation will cause a hard fault.

### Functions and modularity
The code is split into four files, each with a single responsibility — no file knows the internals of another. main.s is the entry point only and contains no logic of its own. string_utils.s handles all string operations. checksum.s handles BCC checksum logic. crc.s handles hardware CRC32.
Functions are shared across files using .global in the defining file and .extern in the calling file. Every function has .type function_name, %function and .thumb_func above its label — required for the linker to correctly resolve cross-file BL calls in Thumb mode. Every function saves any register it uses (R4–R7, LR) with PUSH at the start and restores them with POP at the end so callers are never affected.
string_length (task a) — takes R1 as a string address, walks a copy of the pointer byte by byte with LDRB, increments a counter until 0x00 is found, returns the count in R2. R1 is never modified.
string_case (task b) — takes R1 as string address and R2 as a direction flag. Uses the ASCII bit-5 property: BIC R3, R3, #0x20 clears bit 5 making a letter uppercase, ORR R3, R3, #0x20 sets bit 5 making it lowercase. Only bytes in range 0x41–0x7A are modified. Each converted byte is written back with STRB.
build_buffer (task c) — takes R0 as source string and R1 as destination buffer. Calls string_length first to get the body length, then writes STX (0x02), LEN (body + 3), the string body, and ETX (0x03) in sequence. Then calls calc_checksum over all bytes from STX through ETX and appends the result. Returns total bytes written in R2.
calc_checksum (task d) — takes R1 as buffer address and R2 as byte count. Initialises accumulator R3 to 0x00 and loops through every byte using EOR R3, R3, byte. Uses SUBS to decrement the counter so the Z flag is set automatically when done. Returns BCC in R3.
verify_checksum (task e) — calls calc_checksum on the entire buffer including the appended BCC byte. Because X EOR X = 0, XORing all bytes including the checksum always produces 0x00 if the data is intact. Any corruption produces a non-zero result. Returns result in R3.

### Testing
Testing was done in the STM32CubeIDE debugger by placing breakpoints after each BL call in main.s and inspecting registers and memory.
Task a — after BL string_length, R2 should be 0x0000000B (11, the length of "Hello World").
Task b — after BL string_case, open Memory Browser at &CaseString. Bytes should read 48 45 4C 4C 4F 20 57 4F 52 4C 44 00 (HELLO WORLD\0). Digits and spaces must be unchanged.
Task c/d — after BL build_buffer, R2 should be 0x0000000F (15). Memory Browser at &OutputBuffer should show 02 0E 48 65 6C 6C 6F 20 57 6F 72 6C 64 03 XX where XX is the BCC byte.
Task e — after BL verify_checksum, R3 should be 0x00000000. To test corruption detection: after build_buffer, manually change one byte in OutputBuffer via Memory Browser, then re-run verify_checksum — R3 should become non-zero.
### Notes
- OutputBuffer and CrcBuffer are declared in .bss rather than .data because they start empty. .bss is zeroed automatically by the startup code and does not consume space in flash.
- The forever_loop at the end of main is required. Embedded programs never return — without it the program counter would advance into undefined memory and cause a hard fault.
- Build warnings about _close, _read, and _write are harmless. They come from the nano C library and are expected on bare-metal projects with no OS.

## Exercise 6 - BONUS Task, Developed from Part 1
## Summary
Extends checksum.s with a CRC32 implementation in a new file crc.s, using the STM32F303's built-in hardware CRC unit. CRC32 is order-sensitive and detects all single-bit errors and byte swaps — errors that BCC silently misses.

## Usage
Called at the end of main.s after tasks a–e. crc32_init must be called once first to enable the peripheral clock. No additional hardware required.

## Valid Input
Same rules as exercise 1. crc32_verify must receive the total buffer length including the 4 appended CRC bytes.

## Functions and Modularity
All functions live in crc.s, independent of checksum.s. crc32_init enables the CRC clock by setting bit 6 of RCC_AHBENR. crc32_calculate resets the unit via CRC_CR, feeds each byte into CRC_DR in a loop, then reads the 32-bit result. crc32_verify recalculates CRC32 over the data bytes, reads the stored CRC from the last 4 bytes with LDR, and returns 0 if they match.

## Testing
After BL crc32_calculate, R3 should be a consistent non-zero 32-bit value. After BL crc32_verify, R3 should be 0x00000000. Corrupt one byte in CrcBuffer via Memory Browser — R3 should become non-zero.

## Notes
The CRC unit must be reset via CRC_CR = 1 before each new calculation, otherwise the previous result corrupts the new one.
Uses IEEE 802.3 polynomial 0x04C11DB7 — the same standard as Ethernet and ZIP files.

## Exercise 2 - Digital I/O LED Counter with Button Control

### Summary
Exercise 2 uses digital input and output on the STM32 to create an 8-bit binary counter displayed on LEDs (PE8–PE15), which is controlled using a push button (PA0).

The system supports two modes:

Button-controlled mode: Each valid button press increments or decrements the counter.
Auto-timed mode: The counter updates automatically at a fixed time interval.

A simple software debounce mechanism is used to ensure reliable button input. The counter also changes direction (up/down) based on its current state.

### Usage
- In button mode, press the user button to update the counter value.
- In auto mode, the counter updates automatically with a fixed delay between steps.
- The mode is selected at compile time using the MODE_SELECT constant:

0 = Button-controlled mode

1 = Auto-timed mode

- The LED outputs display the current counter value in binary.

### Valid input
Button input (PA0):
- Digital input (pressed or not pressed).
- Debounced using a fixed delay (DEBOUNCE_MS).
Timing constants:
- DELAY_1MS must be a positive integer representing loop iterations for ~1 ms delay.
- AUTO_DELAY_MS and DEBOUNCE_MS should be non-negative integers (milliseconds).
Counter range:
- Valid values: 0 to 255 (8-bit range).

### Functions and modularity
definitions.s
- Stores all constants such as register addresses, pin mappings, and timing values. This keeps hardware details in one place.

Initialisation
- Sets up GPIO:
- PA0 as input (button)
- PE8–PE15 as outputs (LEDs)
- Enables required clocks

Main loop
- Handles:
- Reading the button
- Debouncing input
- Updating the counter (increment/decrement)
- Outputting the value to LEDs
- Switching behaviour based on mode


### Testing
- GPIO Setup:
Verified by confirming LEDs respond to written values on GPIOE.
- Button Input:
Tested by pressing the button and confirming a single increment/decrement per press (no bouncing effects).
- Debounce Logic:
Verified by ensuring rapid presses do not cause multiple unintended counts.
- Counter Functionality:
Observed LEDs to confirm correct binary counting from 0 to 255 and proper wrap-around.
- Mode Switching:
Tested by changing MODE_SELECT
- Button mode: counter updates only on press
- Auto mode: counter updates at fixed intervals

### Notes
- The counter operates as an 8-bit system, limiting values from 0–255.
- Mode selection is done at compile time, not dynamically during execution.

## Exercise 3 - Serial Communication via UART

### Summary
In this exercise, UART was used to send and receive data on the STM32 board.  
The program builds a packet containing a start byte, length byte, message data, end byte, and checksum.  
It can also receive a packet, check whether it is correct, send an ACK or NAK response.
### Usage
The program first sets up the clock and enables UART communication.  
In transmit mode, it builds a packet and sends it byte by byte through UART.  
In receive mode, it waits for incoming data, reads the packet, checks the format and checksum, and then decides whether the packet is valid.

### Functions and modularity
- `definitions.s`: contains register addresses, constants, protocol bytes, and baud-rate values  
- `initialise.s`: sets up clocks and UART  
- `assembly.s`: contains the main logic for packet building, transmission, reception, checksum checking, and ACK/NAK handling

### Testing
Testing was done in stages:


1. A single character was transmitted repeatedly to confirm UART was configured correctly  
2. A string message was transmitted and checked on the terminal  
3. The packet format was tested by sending structured data with STX, length, ETX, and checksum  
4. Clock speed was changed and the baud-rate register value was updated to keep the UART output correct  

The serial output was checked using a terminal on the laptop.


## Exercise 4 - Independent Hardware Timers with LEDs

### Summary
Exercise 4 focuses on using the STM32 TIM2 hardware timer. The code shows how the timer can be used for various timings speeds specifically microseconds and milliseconds. The STM32 also shows how a timer can be used in other tasks e.g flashing LEDs at desired speeds. The exercise is essentially dived into 4 parts (a, b, c and d).
- Part a: Sets up the TIM2 and uses the harware timer to create a delay function using microseconds.
- Part b: Shows how repeated 0.1ms delays add up to 1 second.
- Part c: Correctly reconfigures TIM2 for millisecond timing.
- Part d: Uses earlier parts to flashe two LEDs indepent of each other at different frequenices.

### Usage

### Valid input
A vaild input for the timer functions would be a non-negative integer value.
- deley_us function: R1 = delay in microseconds
- delay_ms function: R1 = delay in milliseconds

In part d the LED timing (frequency) is controlled by
• led1_half_period_ms
• led2_half_perios_ms
and should also have non-negtive integer values in milliseconds

### Functions and modularity
definitions.s: 
Contains all the hardware constants such as register addresses, offset, and LED/timer constants.

initialise.s:
Contains the following setup functions, gpio_init, enable_timer2_clock, setup_timer2_1us, and setup_timer2_1ms.

main.s:
Contains the following key functions set_leds, delay_us, delay_ms, and main.

### Testing
Part a:
Part a can be tested by stepping into delay_us and confirming that TIM2_CNT was reset and increased until it reached the value in R1.

Part b:
Thus was tested by repeating the 100 µs delay 10000 times and storing the result to show approximately 1 second.

Part c:
This was tested for millisecond timing by confirming delay_ms and waiting until TIM2_CNT reached the requested value.

Part d:
This was tested visually by checking the LEDs were flashing at there desired speeds and speeds changed when half periods were changed.

### Notes
- TIM2 is the timer used throughout this exercise.
- The LEDs are driven through PE8–PE15 on the STM32F3 discovery board.

## Exercise 5 - Final integratation of the earlier exercises into a two-board system.

### Summary
The first board (board 1) increments a counter every 1 second then builds a framed UART message which is sent to board 2. Board 2 receives this message, verifies it then displays the value on the LEDs and finally sends back ACK or NAK.
If Board 1 then receives NAK or no reply within 5 seconds, it flashes all LEDs three times and resets the counter.

### Usage
1. Load the sender program onto Board 1.
2. Load the receiver program onto Board 2.
3. Physically connect the two boards through UART.
4. Observe Board 2 displaying the counter value and sending ACK/NAK.
5. Observe Board 1 incrementing normally on ACK, or flashing/resetting on NAK/timeout.

### Valid input
The program expects a framed UART message in the form seen below:
[STX][LEN][COUNTER = XXX][ETX][BCC]
A vaild message should have body 
• COUNTER = XXX
where XXX is a 3-digit decimal value and the checksum must be correct.

### Functions and modularity
The code is split into two shared setup files (definitions and initialise) and two separate main programs for each board.

### Testing
We can start by testing that both boards work by connecting them both and see if correct number is displayed on LED and seeing that it updates every second. 
We can also test failure case by removing connection from board 1 and 2 and seeing if after 5 seconds board 1 starts flashing indicating failure to respond with ACK.
### Notes
• Exercise 5 reuses code from Exercises 1–4.
• A polling approach is used for UART and timer checks.
• Two separate main files are used because each STM32 board has a different role.
• TIM2 is used for the 1 second delay, 0.5 second flashing, and timeout handling.
