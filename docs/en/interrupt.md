# Interrupted
## Kernel interrupt mechanism
### Interrupt the agent
```rust
delegate all interrupts and exceptions to supervisor mode.
    medeleg::write(0xffff);
    mideleg::write(0xffff);
    sie:::write() | sie::SIE::SEIE as usize | sie::SIE::SIE as usize
```  
First, we interrupt the proxy in the bootloader stage, at which point we are still at the M privilege level, and our kernel needs to run at the S-privilege level. By default, all the trap interrupts are processed in the M state, so we need to proxy it to the S state so that we can handle it.    
   

Here is a brief introduction to the role of the machine exception delegation register, the machine interrupt delegation register, and the “sie” (supervisor interrupt register):  
  

`medelege` and `mideleg` indicate that the current interrupt or exception should be dealt with at a lower privileged level. In the system we have three privileged levels (M/S/U) in which the interrupt agent that occurs in the system can be processed to the S state by setting up `medelegec` and `mideleg`. If U-state is supported, we can also process the interrupt agent to the U-state by setting up the `sedlege` and `sideleg` registers. 


The `sie` is the meaning of the interrupt to enable the register, 0-15 bits are assigned as the standard interrupt cause, and more than 16 bits are set for a specific platform or client.  
Here we need to set the `sie` register to be writeable and set its `SEIE`, `STIE`, and `SSIE` bits.    
   

Among them, the `SEIE` setting indicates the external interrupt of the opening S-state; the `STIE` indicates the interrupt of the opening S-state clock; and the `SSIE` setting indicates that the S-state-on software interrupt is turned on.
 
### Start the interrupt
First, we write `kernelvec` as an address to the `stvec` register in the `trapinit` function, and the `stvec` register is the `Supervisor Trap Vector Base Address Register`, including the vector base address and vector mode. In this way, when our operating system kernel detects an interrupt, we go to `stvec` to view the address of the processing trap function, and then go into the trap processing.  
   
where our trap handler is represented by a compilation:
```Asm
Section .text
The global kertrap
.globl kernelvec
.align 4
The kernelvec:
        Make room to save registers.
        addi sp, sp, -256

        Save the registers.
        sd ra, 0 (sp)
        sd sp, 8(sp)
        sd gp, 16(sp)
        sd tp, 24(sp)
        sd t0, 32(sp)
        sd t1, 40 (sp)
        sd t2, 48(sp)
        sd s0, 56(sp)
        sd s1, 64(sp)
        sd a0, 72(sp)
        sd A1, 80(sp)
        sd a2, 88(sp)
        sd A3, 96(sp)
        sd A4, 104(sp)
        sd A5, 112(sp)
        sd A6, 120 (sp)
        sd A7, 128(sp)
        sd s2, 136(sp)
        sd s3, 144(sp)
        sd s4, 152 (sp)
        sd s5, 160 (sp)
        sd s6, 168 (sp)
        sd s7, 176 (sp)
        sd s8, 184 (sp)
        sd s9, 192 (sp)
        sd s10, 200(sp)
        sd s11, 208(sp)
        sd t3, 216(sp)
        sd t4, 224 (sp)
        sd t5, 232 (sp)
        sd t6, 240(sp)

	Call the C trap handler in trap.rs
        call kerneltrap

        // restore registers.
        ld ra, 0 (sp)
        ld sp, 8 (sp)
        ld gp, 16(sp)
        // not this, in case we move CPUs: ld tp, 24(sp)
        ld t0, 32(sp)
        ld t1, 40 (sp)
        ld t2, 48 (sp)
        ld s0, 56(sp)
        ld s1, 64(sp)
        ld a0, 72 (sp)
        ld a1, 80 (sp)
        ld a2, 88 (sp)
        ld A3, 96 (sp)
        ld a4, 104 (sp)
        ld A5, 112(sp)
        ld A6, 120 (sp)
        ld A7, 128 (sp)
        ld s2, 136(sp)
        ld s3, 144(sp)
        ld s4, 152 (sp)
        ld s5, 160 (sp)
        ld s6, 168 (sp)
        ld s7, 176 (sp)
        ld s8, 184 (sp)
        ld s9, 192 (sp)
        ld s10, 200 (sp)
        ld s11, 208(sp)
        ld t3, 216 (sp)
        ld t4, 224 (sp)
        ld t5, 232 (sp)
        ld t6, 240 (sp)

        addi sp, sp, 256

        Return to whatever we were doing in the kernel.
        Sret
```
  
It can be seen that we save the necessary register values into the stack, and then call the `kerneltrap` function to process the kernel trap, and when we perform the `kerneltrap' function, we recover the stack content and restore the saved context content, and then execute the `srett' command to return the trap's instruction and continue to run.  

####Open the PLIC
The "PLIC" is the riscv Platform Level Interrupt Controller, as shown in the following figure:
![] (static/PLIC.jpg)  
![] (static/PLICArch.jpg)
It can be seen that the `PLIC` receives the interrupt and forwards it to a different `Hart` for processing.
  
The `PLIC` structure shows as shown in the above figure, when the `PLIC` receives the interrupt signal, the logical signal processing is performed internally, and the final output signal is processed to the corresponding CPU.  
  
Under the `QEMU` platform, we do not need to access specific external devices, we only need to write some values to the test address provided by QEMU to open the detection of external interruptions.  
  
For example, the `plicinit` function in `xv6-rust`:
```rust
pub unsafe fn plicinit(){
    println!("plic init...");
    Set desired IRQ priorities non-zero (otherwise disabled).
    let plic:usize = Into::<usize>::into(memlayout::PLIC);
    let mut addr = plic + memlayout::UART0_IRQ*4;
    ptr::write_volatile(addr as *mut u32, 1);

    addr = plic + memlayout::UART0_IRQ*4;
    ptr::write_volatile(addr as *mut u32, 1);
}
```  
  
We only need to write the corresponding peripheral value to the corresponding physical address as required, and we can open the external interrupt mechanism of the peripheral.  
   
#### Kernel Interrupt Processing  
When we enter the kernel interrupt processing function (i.e., `kerneltrap`), we need to obtain the cause of the interrupt or exception through the value of the `scause` register, and thus process it for different reasons. At the same time, we also need to obtain the context switch between the interrupts by reading the `sstatus` and the `sepc' register.  
  
Where the `sepc` register records the virtual address where the interrupt or exception occurs, RISC-V writes the value of the `sepc` register to the program counter after our interrupt is finished, so we need to modify the value of the `sepc' and rewrite it to the `sepc` to save the context.  
  
Similarly, the `sstatus` register holds the current level of the break or the privilege level where the exception is located, and when the interrupt or exception occurs in the U state, the `SPP` bit of `sstatus` is placed at 0, and then the `sret` command is returned to the U state; when the `status` bit is placed at 1, the `sret` command returns the S-state, and the `SPP` bit is still placed at 0.  
  
```rust
/ interrupts and exceptions from kernel code go here via kernelvec,
On whatever the current kernel stack is.
#[no_mangle]
pub unsafe fn kerneltrap(
   arg0: usize, arg1: usize, arg2: usize, _: usize,
   _: usize, _: usize, _: usize, which: usize
) {

    let mut sepc = sepc::read();
    let sstatus = sstatus::read();
    let scause = scause::read();

    // if !sstatus::is_from_supervisor() {
    panic! ("kerneltrap: not from supervisor mode")
    // }

    if sstatus::intr_get() {
        panic! ("kerneltrap(): interrupts enabled");
    }
    
    let which_dev = devintr();

    match which_dev {
        0 => {
            modify sepc to countine running after restoring context
            sepc += 2;
            
            let scause = Scause::new(scause);
            match cause.cause() {
                Trap::Exception(Exception::Breakpoint) => println!("BreakPoint!")

                Trap::Exception(Exception::LoadFault) => panic!("Load Fault!")

                Trap::Exception(Exception::LoadPageFault) => panic!("Load Page Fault!")

                Trap::Exception(Exception::StorePageFault) => panic!("Store Page Fault!")

                Trap::Exception(Exception::KernelEnvCall) => kernel_syscall(arg0, arg1, arg2, which),

                _ => panic!("Unresolved Trap! scause:{:?}", scause.cause())
            }

        }

        1 => {
            Panic! ("Unsolved solution!")
            

        }

        2 => {
            CPU_MANAGER.yield_proc();
        }

        _ => {
            Unreachable! (
        }
    }

    The store context
    sepc::write(sepc);
    sstatus::write(sstatus);
}
```
The `devintr` function detects whether it is an external interrupt, a clock interrupt, and other interrupts or exceptions.
