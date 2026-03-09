# to start

## Disable the Standard Library

The project is linked by default to the Rust standard library std, which relies on the operating system, so we need to explicitly disable it by `#![no_std]` .

Remove the runtime environment dependency

For most languages, they use the Runtime System, which may result in the `main` function not being the first function actually executing.

In Rust, for example, a typical Rust program that links to a standard library will first jump to `crt0`(C Runtime Zero) in the C runtime environment into the C runtime environment to set the environment required for the C program to run (such as creating a stack or setting register parameters, etc.).

The C runtime environment then jumps to the Entry Point of the Rust runtime environment to enter the Rust runtime entry function to continue setting the Rust runtime environment, and this Rust runtime entry point is marked by the start` semantic entry. The main function is called to the main program after the entry point of the Rust runtime environment is over.

The C runtime environment and the Rust runtime environment require standard library support, and our programs are inaccessible. If the `start` semantic term is covered, it still needs `crt0` and doesn't solve the problem. So you need to rewrite to overwrite the entire `crt0` entry point:

We use `#![no_main] to declare that the `main` function is not used as the entry point of the project.

## Compile to bare metal target

By default, Rust tries to adapt to the current system environment and compiles the executable. For example, if you use a Windows system on the x86_64 platform, Rust tries to compile a Windows executable with the extension `.exe` and uses the `x86_64` instruction set. This environment is also called your host system.

To describe different environments, Rust uses a string called Target Triple, `<arch><sub>-<vendor>-<sys>-<abi>`. To view the target trune of the current system, we can run `rust --c --version --verbose`.

Here we use `riscv64gc-unknown-none-elf` as the runtime environment, which does not have an operating system, which is determined by the `none'.

Adjusting the memory layout

We use **Linker Script** to generate memory layouts for the specified program:

** kernel.ld:**

```linker script
OUTPUT_ARCH("riscv")
ENTRY (_entry)

SECTIONS
{
  /*
   * ensure that entry.S /_entry is at 0x80000000,
   Where qemu's -kernel jumps.
   */
  . = 0x8000000;

  .text:
  {
    * (.text .text.*)
    . = ALIGN (0x1000)
    (trampsec)
  }

  .rodata:
  {
    * (.rodata .rodata.*)
  }

  . = ALIGN (0x1000)
  PROVIDE(etext = .);

  /*
   * Make sure the end is after data and bss.
   */
  .data: {
    * (.data.data.*)
  }
  .bss: {
    * (.bss .bss.*)
    * (.sbss* .sbss.*)
  }
  PROVIDE(end=.);
}
```

Here we first specify the instruction set architecture using `OUPUT_ARCH`, then the entry point of the program using `ENTRY`, where we specify the start-up address of the operating system to `0x80000000`, then set the address of the `text`, `rodata`, `data`, `data`, etc., and then the last address is marked as `end` so that the address of the in-memory allocation will change as the address of our program continues to change, not the same value.

Rewrite the program entry point

Since we specify the entry point of the program in `kernel.ld`, we need to fill in the entry point of the program in `entry.asm`:

```assembly
# qemu -kernel starts at 0x1000. the instructions
    # there seems to be provided by qemu, as if it
    # were a ROM, the code at 0x1000 jumps to
    # 0x8000000, the _entry function here,
    # in machine mode. each CPU starts here.
    .text
    .globl_entry
The entry:
	Set up a stack for Rust.
    # stack0 is below declared,
    # with a 4096-byte stack per CPU.
    # sp = stack0 + (hartid * 4096)
    La Sp, stack0
    li a0, 1024*4
	csrr a1, mhartid
    addi a1, a1, 1
    mul a0, a0, a1
    add sp, sp, a0
	# jump to start() in start.rs
    Call to start


    Section .data
    .align 4
The stack0:
    .space 4096 * 8 # 8 is NCPU in param.rs
```

In the assembly file we declare the `text` paragraph and write the assembly of the `_entry`, where we declare the stack address and place the stack address in the `text` segment, `_entry` mainly by reading the `mhartid` register to get the number of current hardware CPUs (we use the QEMU hardware emulator to simulate, you can modify the CPU format in the compilation rule) and move the `sp` register (the stack register) to the corresponding position, and then call the `start` function to make the `bootloader`,`start` function defined in `start.rs`:

```rust
use crate::register::{
    mstatus, mepc, satp, medeleg, medeleg, sie, mhartid, tp, clint, 
    mscratch, mtvec, mie
(a);

use crate::rust_main::rust_main;
use crate::define::param::NCPU

static mut timer_scratch:[[u64; 5]; NCPU] = [[0u64; 5]; NCPU];

#[no_mangle]
pub unsafe fn start() -> !{
    Set M Previlege mode to Supervisor, for mret
    mstatus::set_mpp()

    // set M Exception Program Counter to main, for mret.
    / require gcc -mcmodel=medany
    mepc::write(rust_main as usize)

    Disable paging for now.
    satp::write(0);

    delegate all interrupts and exceptions to supervisor mode.
    medeleg::write(0xffff);
    mideleg::write(0xffff);
    sie:::write() | sie::SIE::SEIE as usize | sie::SIE::SIE as usize

    Ask for clock interrupts.
    timerinit();

    // keep each CPU's hartid in its tp register, for cpuid().
    let id:usize = mhartid::read(); 
    tp::write(id);

    Switch to supervisor mode and jump to main()
    core::arch::asm!("mret"::::"volatile");

    loop{}
    
}

// set up to receive timer interrupts in machine mode,
// which arrive at timervec in kernelvec.S,
// What turns them into software interrupts for
// devintr() in trap.rs.
unsafe fn timerinit(){
    each CPU has a separate source of timer interrupts.
    let id = mhartid::read();

    Ask the CLINT for a timer interrupt.
    let interval = 1000000; // cycles; about 1/10th second in qemu.
    clint::add_mtimecmp(id, interval);


    // prepare information in scratch[] for timervec.
    scratch[0.2]: space for timervec to save registers.
    scratch[3]: address of CLINT MTIMECMP register.
    scratch: desired interval (in cycles) between timer interrupts.

    timer_scratch[id][3] = clint::count_mtiecmp(id) as u64;
    timer_scratch[id][4] = interval;
    mscratch::write(timer_scratch[id].as_ptr() as usize);

    set the machine-mode trap handler.
    extern "C" {
        fn timervec();
    }

    mtvec::write(timervec as usize)

    Enable machine-mode interrupts.
    mstatus::enable_interrupt();

    // enable machine-mode timer interrupts.
    mie::write(mie::read() | mie::MIE::MTIE as usize;

}
```

 In the `start` function, we do some initialization operations on the M-state register, and the specific operation is found in the comments in the code. Our series of operations on the register are defined as `mod` in the `register/` directory, which we can call as a package.

After doing a series of initialization operations, we performed the `mret` directive switching from M Mode to S Mode, officially starting the operating system kernel.
