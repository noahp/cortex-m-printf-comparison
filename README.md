# ARM Cortex-M printf Comparison  <!-- omit in toc -->

- [printf](#printf)
  - [Redirecting `_write`](#redirecting-_write)
  - [Reentrancy](#reentrancy)
    - [One thread only](#one-thread-only)
    - [Disable interrupts](#disable-interrupts)
    - [Only use reentrant libc functions](#only-use-reentrant-libc-functions)
    - [Dynamic reentrancy](#dynamic-reentrancy)
    - [Overriding `__impure_ptr`](#overriding-__impure_ptr)
- [Redirecting stdio](#redirecting-stdio)
  - [UART](#uart)
  - [Semihosting](#semihosting)
  - [SWO](#swo)
  - [Real Time Terminal (RTT)](#real-time-terminal-rtt)
  - [Summary/Comparison](#summarycomparison)

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
#include <stddef.h>  // for size_t
#include <unistd.h>  // for STDERR_FILENO
int _write (int file, const void * ptr, size_t len) {
  if (file == STDERR_FILENO) {
    uart_write(ptr, len);
  }

  // return the number of bytes passed (all of them)
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
data structures. If you want to safely use them in multiple threads, you have a
couple of options:

- only use these functions in one thread in your program
- disable interrupts when using non-reentrant functions
- only use the reentrant versions of the functions (denoted by `_xxx_r()`, eg
  `_printf_r` in newlib)
- recompile newlib with `__DYNAMIC_REENT__` to have the non-reentrant functions
  call a user-defined function to get the current thread's reentrancy structure
- pretty hacky, override the default global reentrancy structure as necessary

TODO link to reent.h in newlib stdlib!!

Some details on those options below.

#### One thread only

It might not be necessary to use the non-reentrant functions in small interrupt
handlers (and is probably a reasonable idea to prohibit them).

Here's an approach that will cause a runtime error if a non-reentrant function
is called in an interrupt context. Unfortunately it requires rebuilding newlib,
so it's really only for curiousity or the very paranoid.

First, rebuild newlib (or build it as part of your project build; this is
probably really only practical if you're using a compiler cache like ccache),
setting `__DYNAMIC_REENT__` when building.

Then, when building your application source, define a function with the
following contents:

```c
#include <reent.h>
struct _reent * __getreent (void) {
  // TODO check if in an interrupt context!!!!
  if (0) {
    __asm__ ("bkpt 0");
  }
  // return the built in global reentrancy context
  return _impure_ptr;
}
```

Also be sure to also define `__DYNAMIC_REENT__` in your CFLAGS when building
your application source.

This will trip a software breakpoint if a non-reentrant function is called from
an interrupt context!

#### Disable interrupts

This approach requires that every lower-priority thread enclose any
non-reentrant calls in a critical section, eg:

```c
__disable_irq();
printf("phew!\n");
__enable_irq();
```

This can add latency to interrupts (on cortex parts, typically interrupts will
pend until enabled after the call), particularly bad if your printf output is
relatively slow, eg 115200 baud UART for example.

It also requires you to be diligent about where you are using non-reentrant
functions.

#### Only use reentrant libc functions

This approach requires only calling the reentrant versions of the libc
functions. See an example below.

```c
#include <reent.h>
#include <stdio.h>

// call printf from an interrupt without breaking other threads
void Foo_Interrupt_Handler(void) {
  struct _reent my_impure_data = _REENT_INIT(my_impure_data);
  _printf_r(&my_impure_data, "hey!\n");
}
```

This is a pretty manual technique but it is relatively simple, and doens't
require disabling interrupts or knowing the relative preemption priority of the
current call site.

#### Dynamic reentrancy

See the section above about building newlib; we need to enable
`__DYNAMIC_REENT__` in the library build to have the non-reentrant functions
call `__getreent()`. Then we can implement a version that provides the correct
reentrancy structure depending on our thread context:

```c
// this system only needs 1 reentrancy context
static struct _reent isr_impure_structs[] = {
  _REENT_INIT(isr_impure_structs[0]),
};

struct _reent * __getreent (void) {
  struct _reent *isr_impure_ptr;

  // this might use NVIC calls to figure out which ISR is active, or if there is
  // an OS, return current executing thread
  int thread_id = get_thread_id();

  switch thread_id {
    case 0:
      isr_impure_ptr = isr_impure_structs[0];
    default:
      // crash
      __asm__("bkpt 0");
  }

  return isr_impure_ptr;
}
```

Note that if you're using an RTOS, this may already be handled in some way; or
there might be a hook when switching thread contexts where we can move
`__impure_ptr`, and we don't even need to recompile newlib!

#### Overriding `__impure_ptr`

Another option is to override the global reentrancy structure when inside
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
non-reentrant function!

Note that you'll need to do the same thing in every preemption context, i.e. any
call site that can interrupt another thread that is using non-reentrant
functions. So this approach would be primarily useful in cases with only a
small number of threads, or a well-considered preemption hierarchy where we know
exactly where to move the pointer and where we don't need to.

## Redirecting stdio

This section describes a few approaches for redirecting libc standard
input/output functions, and compares the tradeoffs each one makes.

### UART

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

### Semihosting

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
- built into most debug servers (pyocd, openocd /* TODO confirm */, blackmagic,
  segger jlink)
- basic implementation is provided by newlib (no extra user code) and is dead
  simple to use
- can do lots more than just printf; open, fwrite, read from stdin!

Cons:
- slowwww. really slow.
- doesn't work when debugger is disconnected (target will be stuck on `bkpt`
  instruction; either a forever hang or a crash depending on cortex-m
  architecture set)
- newlib implementation is relatively code-heavy (8k or bigger!)

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

As an extra bonus, see here /* TODO!!! */ for a simplified semihosting solution
based on the newlib one that doesn't require linking against `rdimon.specs`, and
saves some code space.

### SWO

### Real Time Terminal (RTT)

### Summary/Comparison
