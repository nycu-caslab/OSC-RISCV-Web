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

Interacting with low-level system firmware is a key aspect of bare-metal development on RISC-V platforms.

In this exercise, you will implement a general-purpose SBI (Supervisor Binary Interface) call wrapper and use it to query basic system information from OpenSBI. This includes retrieving the current hardware thread ID (hart ID) and the OpenSBI version specification.

You will:
	•	Implement the ``sbi_ecall(...)`` function in C using inline assembly.
	•	Use the RISC-V standard extension SBI_EXT_BASE to obtain:
	•	Current hart ID
	•	OpenSBI spec version
	•	Integrate the results into your shell (e.g., via info command).

SBI Call Design
========================

.. #TODO add https://elixir.bootlin.com/linux/v6.6/source/arch/riscv/kernel/sbi.c#L25

OpenSBI exposes system services to supervisor-mode software through a standardized calling convention using the ecall instruction.

You will implement a generic function:

.. code-block:: sbi_ecall

    struct sbiret sbi_ecall(int ext, int fid,
                            unsigned long arg0,
                            unsigned long arg1,
                            unsigned long arg2,
                            unsigned long arg3,
                            unsigned long arg4,
                            unsigned long arg5);


This function uses inline assembly to load arguments into the appropriate RISC-V registers (a0-a7), executes an ecall, 
and retrieves the result from registers a0 (error code) and a1 (value).

.. admonition:: Todo

    Implement ``sbi_ecall(...)`` using inline assembly.  
    Test it by calling the following SBI functions with extension ID ``0x10`` (SBI_EXT_BASE):

    - Function ID ``0x0``: ``sbi_get_spec_version()``: returns OpenSBI version
    - Function ID ``0x1``: ``sbi_get_impl_id()``: returns implementation ID
    - Function ID ``0x2``: ``sbi_get_impl_version()``: returns implementation version
    .. - sbi_system_reset #TODO

.. #TODO suggest implementation of printf-like function is suggested

.. Use the ``sbi_ecall(...)`` function to implement a new shell command ``info`` that displays the following information:

.. - OpenSBI version
.. - Implementation ID
.. - Implementation version


********************
Advanced Exercise
********************

Advanced Exercise: Device Tree-Based Info (10%)
###############################################

On most RISC-V platforms, including the VF2 board, a pre-installed firmware-level bootloader
(such as OpenSBI or U-Boot) is responsible for early hardware initialization and for passing 
a Flattened Device Tree (DTB) to the kernel or to a custom bare-metal program.

The DTB provides a structured description of the platform's hardware components. 
It is assumed that the DTB pointer is passed in register ``a1`` during boot.
You should store this value and use it as the starting point for your access functions.

.. admonition:: Todo

    Implement two simple accessors:

    - Extract the ``model`` string from the root node.
    - Search for a node whose ``compatible`` field contains ``"snps,dw-apb-uart"``,
      and extract its ``reg`` property as the UART base address.

    Integrate these outputs into your ``info`` shell command alongside the SBI results,
    and use the obtained base address to initialize the serial interface.

The device tree source file ``jh7110-starfive-visionfive-2-v1.3b.dts`` can be found
in the official `StarFive Linux GitHub repository <https://github.com/starfive-tech/linux>`_.

.. #TODO alternative code sample repository
.. model:
.. https://elixir.bootlin.com/linux/v6.6/source/arch/riscv/boot/dts/starfive/jh7110-starfive-visionfive-2-v1.3b.dts#L11
.. uart0:
.. https://elixir.bootlin.com/linux/v6.6/source/arch/riscv/boot/dts/starfive/jh7110.dtsi#L374

Although full device tree parsing is complex and will be covered in detail in the next lab,
in this exercise you will retrieve a few key fields using fixed property names and simple node matching.
This allows you to begin using the device tree as a source of system configuration data
without needing to understand its full hierarchical structure yet.

The ``model`` property under the root node provides a human-readable description of the target board.
It is commonly used in logs, diagnostics, and shell outputs to identify the hardware platform. 
Even though this lab explicitly targets the VF2 board, retrieving the ``model`` field helps illustrate how
hardware information can be extracted from the DTB for display and verification purposes. 

This also provides a good opportunity to replace hardcoded board-specific configuration
values from Lab 0 with values extracted from the DTB. 
Specifically, the UART base address used to initialize the serial interface in Lab 0 
may now be retrieved by scanning the DTB for a node whose ``compatible`` field contains 
``"snps,dw-apb-uart"``. This node should also contain a ``reg`` property, which specifies 
the base address and size of the UART registers.

Using the ``compatible`` field rather than a fixed node name (such as ``serial@10000000``)
makes your implementation more robust and portable across board versions or different UART configurations.

The ``reg`` field declares the UART base address using a 64-bit format:

    ``reg = <0x0 0x10000000 0x0 0x10000>;``

To extract the actual base address, combine the first two 32-bit words of the ``reg`` property:

    ``base = (uint64_t)reg[0] << 32 | reg[1];``

More details can be found in Section 2.3 of the 
`JH7110 UART Developing Guide <https://github.com/nycu-caslab/OSC-RISCV-Web/raw/refs/heads/main/uploads/SDK_DG_UART.pdf>`_.

As a practical code reference, you may also consult the minimal DTB-parsing example ``fdtget.c``
from the `dgibson/dtc repository <https://github.com/dgibson/dtc/>`_. 
This example demonstrates how to locate a device node by path and retrieve a property using:

- ``fdt_path_offset(fdt, "/path")``
- ``fdt_getprop(fdt, offset, "property", &len)``

While your implementation in this lab does not rely on full-featured device tree parsing,
it may still be helpful to understand how the standard library interfaces with the DTB structure.

.. note::

    The ``model`` property is *optional* according to the device tree specification,
    but it is typically present at the root node to provide a descriptive label for the board.


