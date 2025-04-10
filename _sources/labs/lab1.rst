========================
Lab 1: Hello World
========================

*************
Introduction
*************

In this lab, you will begin practicing bare-metal programming on the VF2 board by implementing a minimal interactive shell.

This lab focuses on configuring the Universal Asynchronous Receiver-Transmitter (UART) interface for serial communication,
which serves as the primary I/O channel between your host computer and the VF2 during development.
You will also gain experience in low-level system initialization, peripheral access, and basic input/output handling.

*****************
Goals of this lab
*****************

- Practice bare-metal programming.
- Understand how to access VF2's peripherals.
- Set up UART for serial communication.

*****************
Basic Exercises
*****************

Basic Exercise 1 - Basic Initialization - 25%
#############################################

When a bare-metal program is loaded onto the VF2, several conditions must be satisfied 
before it can execute correctly:

- All data sections must be placed at the expected memory addresses.
- The program counter must point to the correct entry address.
- The ``.bss`` segment (which holds uninitialized global variables) must be zero-initialized.
- The stack pointer must be set to a valid memory location.

During boot, VF2's bootloader loads the kernel image into a designated physical address 
and begins execution from there. If your linker script is correct, the first two conditions 
are already handled.

However, the ``.bss`` segment and the stack pointer are not initialized automatically.
Failing to configure them properly can lead to undefined behavior, such as corrupted memory or crashes.

.. admonition:: Todo

    Initialize the system state immediately after the bootloader hands control to your program.
    This includes setting up the stack pointer and zeroing out the ``.bss`` segment manually.

Basic Exercise 2 - UART Setup - 25%
####################################

For all labs in this course, UART will be the primary interface for communication between the VF2 and your host computer.
You will use it to read input, print output, and interact with the system while debugging.

In this exercise, you will configure the UART peripheral by directly writing to its memory-mapped registers.
This includes setting the baud rate, data format, and enabling both the transmitter and receiver.

Refer to the VF2's hardware documentation for details on the base address and layout of the UART registers.

.. admonition:: Todo

    Set up the UART interface on VF2.
    This includes configuring the UART registers to initialize the baud rate, data bits, stop bits,
    and enabling both the transmitter and receiver.

Basic Exercise 3 - Simple Shell - 25%
######################################

Once UART is configured correctly, you can build a simple shell interface 
to enable basic interaction between the VF2 and your host computer.

The shell should process user input received via UART and respond with predefined messages.
At minimum, it must support the following commands:

- ``help``: Display a list of available commands.
- ``hello``: Display the message "Hello World!"

You may implement a simple command parser that reads input character by character,
identifies complete commands, and prints the corresponding output.

.. important::

    Be mindful of character alignment issues when handling screen I/O.
    Consider translating newline characters (`\n`) into carriage return + newline (`\r\n`)
    to ensure proper display across different serial terminals.

.. admonition:: Todo

    Implement a basic shell that reads input from UART and displays output accordingly.
    The shell should recognize the listed commands and print appropriate responses.

Basic Exercise 4 - System Information - 25%
############################################

Interacting with low-level hardware is a fundamental part of bare-metal programming.

In this exercise, you will retrieve and display basic system information from the VF2 board,
such as board revision and memory layout. This helps you understand how to read from
hardware-defined memory locations or control registers, and how to format the output for human-readable display.

At minimum, your shell should be able to display the following information:

- Board revision
- Base address and size of system memory

.. admonition:: Todo

    Implement functionality to retrieve and display the VF2's hardware information.
    You should at least print the board revision, and the base address and size of system memory.
