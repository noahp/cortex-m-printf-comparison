# ARM Cortex-M printf Comparison  <!-- omit in toc -->


- [printf](#printf)
  - [Redirecting `_write`](#redirecting-_write)
  - [Reentrancy](#reentrancy)
- [UART](#uart)
- [Semihosting](#semihosting)
- [SWO](#swo)
- [Real Time Terminal (RTT)](#real-time-terminal-rtt)
- [Summary](#summary)

## printf

Ah, the first and last of debugging techniques!

On unhosted platforms (eg bare-metal or minimal RTOS), you may not have built-in
support for `printf`/`scanf`/`fprintf`. Fear not, you can get some printf in
your life by using newlib-nano's support:

```bash
# add these to your linker command line
LDFLAGS += --specs=nano.specs --specs=nosys.specs
```

That adds libc and "nosys" support to your application. This means you can add
`printf` calls and your application will link. But we're not quite done.

### Redirecting `_write`

`_write()` is the function eventually called by `printf`/`fprintf` that is
responsible for emitting the bytes formatted by those functions to the
appropriate file descriptor.

If you want to redirect the output of your `printf`/`fprintf` calls (by default
it's written out to RDIMON, more on that later), you might want to implement
your own copy of `_write()`. The version of that function supplied by newlib
libc is marked as `weak`, so a non-weak implementation will override it.

For a better explanation of this process for `_write` and other hosting
functions, check out this excellent guide:

https://www.embecosm.com/appnotes/ean9/ean9-howto-newlib-1.0.html#id2719973

The minimal implementation might look like this:

```c
// stderr output is written via the 'uart_write' function
int _write (int file, const void * ptr, size_t len) {
  if (file == stderr) {
    uart_write(ptr, len);
  }

  // return the number of bytes passed
  return len;
}
```

Note that the libc `setbuf` API controls I/O buffering for newlib. By default
`_write` will be called for every byte. You might want to instead buffer up to
line ends, by:

```c
#include <stdio.h>
setvbuf(stout, NULL, _IOLBF, 0);
```

See `man setvbuf` or http://www.cplusplus.com/reference/cstdio/setvbuf/ for
details.

### Reentrancy

Fancy word for if a function can be safely premepted and called simultaneously
from multiple (interrupt) contexts.

Some libc functions in newlib share a context structure that contains global
data structures. If you want to safely use them in multiple threads, you need to
either disable interrupts while using these functions (oof), or switch to using
the reentrant versions.

A good explanation of this problem and how to solve it can be found in the same
porting guide:

https://www.embecosm.com/appnotes/ean9/ean9-howto-newlib-1.0.html#sec_reentrant_syscalls

A third option is to override the global reentrancy structure when inside
interrupts; a little trickier but can be useful in certain systems.

The global reentrant structure is referenced by the pointer `__impure_ptr`,
defined here:

https://sourceware.org/git/?p=newlib-cygwin.git;a=blob;f=newlib/libc/reent/impure.c;h=76f67459e48d3efc20698f8fda39620b1359f63f;hb=HEAD#l27

You can instantiate your own copy of the structure:

```c
#include <reent.h>
struct _reent my_impure_data = _REENT_INIT(my_impure_data);
```

And then temporarily override `_impure_ptr`:

```c
_impure_ptr = &my_impure_data;
printf("yolo\n");
// restore default global reentrancy struct
_impoure_ptr = &impure_data;
```

This makes it safe to call the above snippet in an interrupt handler, because it
won't stomp on the global reentrancy data if it preempted an in-progress
non-reentrat function!

## UART

UART: asynchronous serial. Most embedded microcontrollers will have at least
one.

Pros:
- simple and widely available protocol (lots of available software and hardware
  tools interface to it)
- doesn't require an attached debugger; you can use it in PROD 😀

Cons:
- requires extra hardware to interface to the PC host, typically (USB to
  asynchronous serial adapter, like the FTDI or CP2102 adapters)
- requires configuring and using the UART peripheral (if your microcontroller
  doesn't have many UARTs, might be a problem using it for `printf`)

The biggest downside is you're spending one of your UART peripherals for printf,
which might be needed for your actual application.

Also, configuring a UART might require managing clock configuration, chip
power/sleep modes, figuring out buffering the data (DMA?), making sure
everything is interrupt safe, etc.

And finally, you'll usually need some kind of adapter dongle to interface the
UART with your PC (eg USB-to-TTL or USB-to-asynchronous serial).

But! because the interface is contained entirely on the microcontroller, it can
be used without a debug probe, which can make it useful when not actively
debugging your system (eg you can use it to write log statements to an attached
PC!).

Worth noting, if your microcontroller has support for USB, you might be able to
skip the need for an external FTDI-like adapter by implementing CDC
(Communications Device Class) on-chip. That adds a lot more software to your
system, but can be useful. Out of scope of this FAQ.

## Semihosting

Next up the food chain, semihosting is a protocol that proxies the "hosting"
(meaning libc hosting, eg `printf`/`scanf`/`open`/`close` etc) over an attached
debug probe to the PC.

Some good descriptions:

- https://wiki.segger.com/Semihosting
- https://www.keil.com/support/man/docs/armcc/armcc_pge1358787046598.htm

And the reference documentation can be found in:
> ARM Developer Suite (ADS) v1.2 Debug Target Guide, Chapter 5. Semihosting

TLDR, semihosting operates by the target executing the breakpoint instruction with
a special breakpoint id, `bkpt 0xab`, with the I/O opcode stored in `r1` and
whatever necessary data stored in a pointer loaded into `r2`.

The debug probe executes the I/O operation based on the opcode and data, then
resumes the microcontroller.

Pros:
- only requires SWD connection (SWCLK + SWDIO)
- built into most debug probes (pyocd, stlink, blackmagic, segger jlink)
- basic implementation is provided by newlib (no extra user code) and is dead
  simple to use
- can do lots more than just printf; open, fwrite, read from stdin!

Cons:
- slowwww. really slow.
- doesn't work when debugger is disconnected (target will be stuck on `bkpt`
  instruction; either a forever hang or a crash depending on cortex-m
  architecture set)

To use it, the simplest approach is to use newlib's rdimon library spec:

```bash
# add these to your linker command line
LDFLAGS += --specs=nano.specs --specs=rdimon.specs
```

Then add this call somewhere in your system init, prior to calling any libc I/O
functions:

```c
// this executes some exchanges with the debug monitor to set up the correct
// file handles for stdout, stderr, and stdin
extern void initialise_monitor_handles(void);
initialise_monitor_handles();
```

`printf` and friends should work after that! Be sure to check the user manual
for your debug probe on how it exposes the semihosting interface. For example,
segger jlink requires running some commands to enable semihosting:

```bash
# from gdb client
monitor semihosting enable
monitor semihosting ioclient 2
```

(From https://wiki.segger.com/J-Link_GDB_Server).

For pyocd, you can pass the `--semihosting` argument when starting the gdb
server.

## SWO

## Real Time Terminal (RTT)

## Summary
