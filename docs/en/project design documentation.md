<h1>
    <center>xv6-rust design report</center>
<h1>

<h3><center>2021 National College Computer System Competence Competition Operating System Design Competition</center></h3>

<h3><center>Ko-oK</center></h3>

<h4><center>Qi Zhenxiang (captain), 闫璟, Li Fengjie</center></h4>



### Introduction to Design

The project is a re-engineering and implementation of MIT's `xv6-riscv` project in Rust language, which can be seen as both an exploration of Rust language in operating system development and a practical job for universities in the course of operating systems.

#2: Design to achieve

Rust, as a more modern language, is more abstract than the traditional system development language C, which makes Rust more recedential. At the same time, Rust, in order to ensure the security of development, it forces users to use the ownership mechanism to ensure the security of the system. For the type of non-realization of the `Copy` feature, the assignment operation is handled in a mobile sense. At the same time, Rust sets immutable references and variable references, Rust references are encapsulated by bare pointers, and some checks can occur during the compilation period: for example, multiple immutable references can occur in a scope.At the same time, Rust treats naked pointers for direct operation as `unsafe`, and if we want to read and write naked pointers directly, we must identify the block of code as `unsafe`. Rust's features intercept most of the imperceptible errors that use traditional system languages during the compilation period, greatly facilitating our debugging process. But these also allow us to redevelop OS with the full use of Rust's features.

**1. Implementation of the lock**

In `xv6-riscv`, for the structure that needs to be locked, only the pointer of the lock structure is placed in the domain in it. In the process of obtaining the lock, only the "lock field" of the variable is checked to determine whether it can be acquire. This writing requires the programmer to be extremely high, because he can still have the content of the variable without obtaining the lock, or because the programmer forgets to release the lock will cause the program deadlock and difficult to debug.

Rust has a more complete type system that supports generics. So we can design the lock as a form of a smart pointer, wrap the specific content of the variable in the lock, return a guard variable when we call the `acquire` method, and access the fields of the original data in the variable. In addition, Rust has the `Drop` feature, as long as we implement the `Drop` feature for our locks, and automatically release the lock variable when the variable leaves the scope, and the RAII is implemented in the true sense, thus avoiding deadlock.

For example, in `xv6-riscv`, it uses the following structure to lock:

```c
struct {
  spinstructlock lock;
  struct run *freelist;
} kmem;
```

And we locked it in a way that wraps the lock as a pointer wrapped outside the variable:

```rust
pub struct KernelHeap (Spinlock<BuddySystem>)
```

- Spin lock

In the implementation of the spin lock, we call the `acquire` method and return a variable of type `SpinlockGuard`, and implement the `Drop` and `DeferMut` features for `SpinlockGuard`, making it less likely to have a deadlock.

where `acquire` is implemented by CAS(Compare And Swap), the spin lock is checked by the memory sorting of atomic variables, and if not, it remains busy waiting. It can be noted that in the middle of the lock and unlock, accompanied by the operation of `push_off` and `pop_off`, which is the encapsulation of `intr_on` and `intr_off`, which prevents external interruptions during the lock and cause deadlock.

```rust
    pub fn acquire(&self) -> SpinlockGuard<'_, T> {

        push_off();
        if self.holding() {
            Panic! ("acquire");
        }
        
        while self.locked.swap(true, Ordering::Acquire){
            Now we signal the processor that it is inside a busy-wait spin-loop 
            spin_loop();
        }
        fence (Ordering::SeqCst)
        unsafe {
            self.cpu_id.set(cpuid() as isize);
        }

        SpinlockGuard{spinlock: &self}
    }

    pub fn release(&self) {
        if !self.holding() {
            Panic! ("release");
        }
        self.cpu_id.set(-1);
        fence (Ordering::SeqCst)
        self.locked.store(false, Ordering::Release)

        pop_off();
    }
```

We implement `Drop` and `DerefMut` features for spin locks, making it easy to modify package values through `SpinGuardlock` and automatically unlock them when `SpinlockGuard` is destroyed.

```rust
impl<T> Deref for SpinlockGuard<'_, T>{
    Type Target = T;

    fn deref(&self) -> &Self::Target {
        unsafe{
            &*self.spinlock.data.get()
        }
    }
}   

impl<T> DerefMut for SpinlockGuard<'_, T>{
    fn deref_mut(&mut self) -> &mut Self::Target{
        unsafe{
            &mut *self.spinlock.data.get()
        }
    }
}

impl<T> Drop for SpinlockGuard<'_, T>{
    fn drop(&mut self) {
        self.spinlock.release()
    }
}

```

The Sleep Lock**

Sleep locks are different from spin locks, which are performed by CAS (Compare And Swap), while sleep locks in addition to using the spin lock to protect the sleep lock, but also need to call the current process to sleep and enter the scheduler for scheduling, and when the call is released the spin lock. The reason for using the spin lock is that we can ensure that we will not miss any time `wake_up` to wake up the sleeping process.

**2. Static variables**

In Rust, static variables have an accurate memory address after compilation, which means that the address space cannot be allocated for static variables at runtime. The same thing is that it can be declared as a global variable in C, and then initialized it at the program runtime, which is not allowed in Rust. Therefore, we initialize the `new() method for a particular type of variable; for variables that need to be assigned on the heap, you need to load it with a lazy_static!' macro to achieve dynamic memory allocation.

In the case of programs in `xv6-rust`, we need to assign a page table to the kernel during the operating system startup process so that the physical address to the virtual address is recorded through the page table item:

```rust
pub static mut KERNEL_PAGETABLE:PageTable = PageTable::empty();
```

```rust
impl PageTable{
    pub const fn empty() -> Self{
        Self{
            entries:[PageTableEntry(0); PGSIZE/8]
        }
    }
}
```

At the same time, in the `e1000` network card driver, according to Intel's standards, we need to assign a message buffer queue for sending messages and receiving messages, and write the header address of the queue to the register, and when we implement sending or receiving messages, the network card sends and receives packets from these queues based on the information of other registers. So we use `lazy_static` to allocate space for our global variables:

```rust
It's lazy_static! {
    static ref RECEIVE_MBUF: Spinlock<[MBuf;RECEIVE_RING_SIZE]> = Spinlock::new(array![_ => MBuf::new(); RECEIVE_RING_SIZE], "receive_mbuf");
    static ref TRANSMIT_MBUF: Spinlock<[MBuf;TRANSMIT_RING_SIZE]> = Spinlock::new(array![_ => MBuf::new(); TRANSMIT_RING_SIZE], "transmit_mbuf");
}
```

Writing for static variables is not secure, so we use locks to wrap the contents of variables, simplifying our program.

**3. Ownership mechanisms and RAII**

The ownership mechanism is good, but it is a torture for the programmer, and there are many problems caused by the ownership mechanism in the process of implementing `xv6-rust`, especially when writing a scheduling algorithm. For example, when implementing the `alloc_proc() function (that is, assigning an address space to the process):

```rust
pub fn alloc_proc(&mut self) -> Option<&mut Process> {
        for p in self.proc.iter_mut() {
            let mut guard = p.data.acquire();
            if guard.state == Procstate::UNUSED {
                .
                .
                .
                An empty user page table
                if let Some(page_table) = unsafe { extern_data.proc_pagetable() } {
                    .
                    .
					.
                } else {
                    p. freeproc();
                    drop (guard);
                    return None
                }
            }else {
                p. freeproc();
                drop (guard);
            }
        }
    	None of the
    }
```

It can be seen that in the original implementation, we first traversed all the processes inside the scheduler, then found an empty process with a state that was unused, and assigned it a page table, and released it if it failed. But since the variable reference to `p` was held simultaneously in the if and else joint code block at this time, which is not allowed in the rust compiler, the rust compiler does not pass, so we can only modify the `proc_page()' function to make it internally check whether the distribution fails when the text is released.

```rust
pub fn alloc_proc(&mut self) -> Option<&mut Process> {
        for p in self.proc.iter_mut() {
            let mut guard = p.data.acquire();
            if guard.state == Procstate::UNUSED {
				.
                .
                .
                An empty user page table
                unsafe{
                    extern_data.proc_pagetable();
                }
               	.
                .
				.
            }else {
                drop (guard);
            }
        }

        None of the
    }
```

At this point we complete the inspection process in `proc_pagetable()`, and simply call its method in `alloc_proc()` to avoid ownership errors.

**4. Design of the process**

In C, although programmers are required to be highly self-disciplined, they have a high degree of flexibility to design the operating system. For example, when a process is implemented, some content of a process needs to be locked when the thread is accessed, and some content does not need to be locked. For example, the state of the process, channel, whether it is killed, exit state, and the readable variable of the process ID need to be locked; and the virtual memory of the kernel, the memory of the process, the user page, the context, the context.6-riscv` can be easily achieved:

```c
The Per-process State
struct proc {
  spinstructlock lock;

  p->lock must be held when using these:
  enum procstate state; // process state
  void *chan; // If non-zero, sleeping on chan
  int killed; // if non-zero, have been killed
  int xstate; // Exit status to be returned to parent's wait
  int pid; // Process ID

  // proc_tree_lock must be held when using this:
  struct proc *parent; // Parent process

  These are private to the process, so p->lock need not be held.
  uint64 kstack; // Virtual address of kernel stack
  uint64 sz; // Size of process memory (bytes)
  pagetable_t pagetable; // User page table
  struct trapframe *trapframe; // data page for trampoline.S
  struct context; // swtch() here to run process
  struct file *ofile[NOFILE]; // open files
  struct inode *cwd; // Current directory
  char name[16]; // Process name (debugging)
(a);
```

In C, you only need to put a locked domain in the structure of the process, and when you need to access the variable that needs to be locked, only need to call the lock's `acquire() and `release() method to get and release the lock to access the domain; when accessing content that does not need to be locked, you only need to directly access the domain in it.

And because Rust's lock form is wrapped in the outer layer of the variable in the form of a smart pointer, if we wrap the lock directly in the outer layer of the variable as if we wanted to access the public domain or the private domain, we had to enable it to `acquire()`and `release()`. Then imagine that when we go to access the user page table, open the file, and the current directory, these time-consuming variables will lock the current process, which will result in a great drop in efficiency!

Therefore, our implementation is to separate the public domain from the private domain and put it into the `Process` structure, and lock the public domain structure, so when we need to access the public domain, we only need to lock the public domain part.

```rust
pub struct process {
    pub data: spinlock<ProcData>
    pub extern_data: UnsafeCell<ProcExtern>,
}

pub struct ProcData {
    p->lock must be held when used by these
    State pub: ProcState,
    pub channel: usize, // If non-zero, sleeping on chan
    pub killed: bool, // if non-zero, have been killed
    pub xstate: usize, // exit status to be returned to parent's wait
    pub pid: usize, // Process ID
}

pub struct ProcExtern {
    These are private to the process, so p->lock to be held
    pub kstack:usize, // Virtual address of kernel stack
    pub size:usize, // size of process memory
    pub pagetable: Option<Box<PageTable>, // User page table
    trapframe: *mut Trapframe, // data page for trampoline.S
    context: Context, // swtch() here to run processes
    pub name: [u8; 16], // Process name (debugging)
    // proc_tree_lock must be held when using this:
    pub parent: Option<*mut Process>   
    pub ofile: Vec<Arc<RefCell<VFile>,
    pub cwd: Option<Inode>
}
```

**5. Interrupted design**

In the original `xv6-riscv` , less information was given for the kernel interrupt information, only the values of registers such as `scause`, `sepc`, `sscause`, and specific interrupt types needed to be checked for RISC-V. So we refer to the implementation of `rust-embedded/riscv` for our registers.

In our interrupt implementation, it is mainly based on the value read in the `scause` register. In it, it can be distinguished into two types of exceptions and interrupts according to the highest bit of `scause`, and can distinguish them according to their low level.

In our interrupts, we analyze the cause to determine the different types of interrupts for specific processing, see [Interruptt Document] (interrupt.md).

**6. Implementation of the file system**

In `xv6-riscv`, the implementation of the file system is divided into seven layers: `File Descriptor Layer`, `Pathname Layer`, `Directory Layer`, `Inode Layer`, `Buffer Layer`, `Disk Layer`. First, the file system obtains the file information (i.e. file descriptor) that needs to be operated by the system call, and the device is operated directly through the device ID when the file to be operated is a directory or file.inode`, then go to complete the corresponding operation.

- Design of Inode (see xv6-riscv-rust)

    In the design of `xv6-riscv`, `inode struct` is a copy of `inode` in memory, and we can only obtain information about the inode stored on disk after we get its sleep lock.

    Based on our previous design experience, we should implement the `lock` method for the `SleepLock<Inode> type, which is theoretically feasible, but it is very inconvenient. For example, in the design of abstract files, we need to store the content of `Option<Inode>`, and it is also inconvenient to process with a sleep lock. In addition, some of the information of `inode` does not need to be locked or read, so it involves the design of `field`, so it is not wise to design `SleepLock<I>node`.

    In our design, the data before and after the lock is processed separately, the data before the lock is obtained, and the data before the lock is called the data stored on the disk. In this case, when the metadata is performed, we obtain the global `InodeCache' cached data based on the metadata, refresh the `InodeData' according to whether the cache data is dirty, and return the disk data that has been locked. Similarly, when we unlock the inode, we make full use of the `Drop' feature, and when the `Inode's life cycle is manually destroyed, when the "Inode's life cycle is destroyed.Cache` automatically executes the `put` method to check if the inode should be recycled.

    ```rust
    Inode handed out by inode cache. 
    // It is actually a handle pointing to the cache. 
    pub struct Inode {
        pub dev: u32,
        pub inum: u32,
        The pub index: usize
    }
    ```

    ```rust
    impl Inode {
        // Lock the inode. 
        // Load it from the disk if its content not cached yet. 
        pub fn lock<'a>(&'a self) -> SleepLockGuard<'a, InodeData> {
            let mut guard = ICACHE.data[self.index].lock();
            
            if !guard.valid {
                let buf = BCACHE.bread(self.dev, unsafe { SUPER_BLOCK.locate_inode(self.inum)};
                let offset = locate_inode_offset(self.inum) as isize;
                let dinode = unsafe{ (buf.raw_data() as *const DiskInode).offset(offset)};
                guard.dinode = unsafe{ core::ptr::read(dinode)};
                drop(buf);
                guard.valid = true;
                if guard.dinode.itype == InodeType::Empty {
                    "Inode lock: trying to lock an inode whose type is empty."
                }
            }
            The guard
        }
    }
    
    ```

    

Design of the Device

    In the design of `xv6-riscv`, there are two methods of reading and writing in each struct in the device table. In my implementation, I choose to use two pointers instead, the pointer points to the read and write method of the corresponding device, and converts it to the corresponding method when called.

    ```Rust
    // map major device number to device functions.
    #[Derive(Clone, Copy)]
    pub struct device {
        pub read: *const u8,
        pub write: *const u8
    }
    ```

    

- Design of block buffers

    In `xv6-riscv` , the block buffer is designed using a bidirectional linked list, and in rust, the implementation of the two-way linked list is not easy due to the design of ownership. Therefore, we still need to separate different domains so that the compilation problem of ownership can be avoided as much as possible. Among them, `BCache` is divided into two domains: LRU and Buf, where Buf is the full content stored in the buffer, and the LRU is used to read the LRU replacement algorithm, where the LRU has the `prev` and `next`..

    The `Buf` struct is a package of the buffer's original data, which is an abstraction of the buffer data at a more underlying level, and the resulting modification can be written back to the data on the disk. In our implementation, the buffer queue can obtain the corresponding block buffer content through the metadata (i.e., the device number or block number), and the corresponding block buffer content is involved in the corresponding buffer data when obtaining the contents of the block (i.e., the content on the disk is directly copied in memory).

**7. Design of memory systems**

- Memory allocation system

    Because rust has the `alloc` feature, an API for memory allocation is provided. There is a global memory allocator for memory allocation in the program that provides the standard library, and in the `no_std` environment, as long as we implement the `alloc` feature and provide the global memory allocator, we can use smart pointers such as `Box` or `Vec`.

    **In-memory allocation system architecture:**

    ![Memory Allocation System Architecture] (static/kalloc.jpg)

    ** Kernel state**

    In the kernel state, we use the partner memory allocation system**. The partner memory allocation system is mainly to reduce the waste of memory, such as to allocate a large piece of memory, using the traditional chain memory allocation can only find a large enough memory to allocate, then there are many small pieces of memory in the middle will be ignored, which will cause memory waste.

    In the partner memory allocation system, the number of memory bytes allocated each time must be a power of 2, and the range size is freely set. The free blocks of memory of different sizes are managed separately. If you want to allocate a 512 bytes-sized memory block, look for the corresponding idle block to the idle list of maintaining 512 bytes, and if not, find an idle block in the free block of 1024 bytes, and add 512 bytes to the empty block of the maintenance 1024 bytes.

    where `Buddy System' is allocated on a page basis (i.e. at least 4096 bytes) is allocated for each page's memory. `slab` is required for each page. In it, we need to maintain a bitmap for each page, where the current memory is allocated or divided into smaller blocks.

    When allocating memory for a given Layout, you first need to align the size of the Layout with the integer pend of 2, and then traverse the corresponding size list, and if you find the free memory, you return the corresponding pointer, otherwise you will eraterate through the larger size list to find the free block.

    When recycling memory, we need to find the corresponding address and size based on the given Layout, depending on the allocation situation or directly release the memory push to the original memory list or after merging and pushing again.   

    ```Rust
    pub struct BuddySystem {
        initialized: bool,
        base: usize, // the starting addr managed by the buddy system
        actual_end: usize, // actual the end addr managed by the buddy system
        nsizes: usize, // the number of different sizes of blocks
        leaf_size: usize,
        max_alignment: usize,
        infos: MaybeUninit<*mut [BuddyInfo]>,
    }
    
    // Buddy info for block of a certain size k, k is a power of 2 
    #[repr(C)]
    struct BuddyInfo {
        free: List, // record blocks of a certain size
        alloc: MaybeUninit<*mut [u8]>, // tell if a block is allocated
        split: MaybeUninit<*mut [u8]>, // tell if a block is split into smaller size
    }
    ```

    As shown above, we have the data structure design for our `Buddy` system, where `BuddySystem` maintains some meta-information, such as the assigned start address, the termination address, and the smallest number of assistable bytes, among other things. where `infos` points to an array pointer of type `BuddyInfo`. Each `BuddyInfo` represents the maintenance of a different `size` linked list. `free` maintains the list of available memory information, while `alloc` and `split` are maintained in the form of the bitmap to maintain the current `size` for the reference.and division.

    ```Rust
        // Allocate a block of memory satisifying the layout.
        pub fn alloc(&mut self, layout: Layout) -> *mut u8 {
            if layout.size() == 0 {
                return ptr::null_mut();
            }
    
            Only guarantee the alignment not bigger than max_alignment
            if layout.align() > self.max_alignment {
                return ptr::null_mut();
            }
            Note: the size of a value is always a multiple of its alignment
            now we only have to consider the size
            Because base and actual_end are already align to max_alignment
    
            // find the smallest block can contain the size
            let smalli = if layout.size() <= self.leaf_size {
                0 
            } else {
                (layout.size().next_power_of_two() / self.leaf_size).trailing_zeros() as usize
            (a);
            let mut sizei = smalli
            while sizei < self.nsizes {
                let info = unsafe { self.get_info_mut(sizei)};
                if !info.free.is_empty() {
                    Break;
                }
                sizei += 1;
            }
            if sizei >= self.nsizes {
                No free memory
                return ptr::null_mut()
            }
    
            // pop a block at self.infos[sizei]
            let info = unsafe { self.get_info_mut(sizei)};
            let raw_addr = match unsafe { info.free.pop() } {
                Some(raw_addr) => raw_addr,
                None => return ptr::null_mut(),
            (a);
            let bi = self.blk_index(sizei, raw_addr);
            unsafe { self.get_info_mut(sizei).alloc_set(bi, true); }
    
            Split the block until it reach smallest block size
            while sizei > smalli {            
                Split two buddies at sizei
                let bi = self.blk_index(sizei, raw_addr);
                let info = unsafe { self.get_info_mut(sizei)};
                info.split_set(bi, true)
    
                alloc one buddy at sizei-1
                let bs1 = self.blk_size(sizei-1);
                let bi1 = self.blk_index(sizei-1, raw_addr);
                let info1 = unsafe { self.get_info_mut(sizei-1)};
                info1.alloc_set(bi1, true)
    
                Free the other buddy at sizei-1
                let buddy_addr = raw_addr + bs1;
                unsafe { info1.free.push(buddy_addr); }
    
                sizei -= 1;
            }
    
            raw_addr as *mut u8
        }
    
    ```

    In the implementation of our `alloc` method, we first go from small to large, we go to the empty list, find a block with enough memory, and get it out of the list, and then get the corresponding block index number based on the value of the block pointer and update the allocation of the `alloc` bitmap. At the same time, if the allocated block size is greater than the size of all the allocated memory, the recursively split the block size until the allocated block size.

    Similarly, `dealloc` does the opposite, recursively restoring the allocated memory by the acquired `layout`:

    ```Rust
        Deallocate the memory.
        SAFETY: The raw ptr passed-in should be the one handed out previously.
        pub fn dealloc(&mut self, ptr: *mut u8, layout: Layout) {
            Check ptr in range [self.base, self.actual_end]
            let mut raw_addr = ptr as usize;
            #[cfg(debug_assertions)]
            if raw_addr < self.base || raw_addr >= self.actual_end {
                panic! ("allocator: dealloc ptr out of range")
            }
    
            // find the size of block pointing by ptr
            / and check with the layout
            let mut sizei = self.nsizes;
            for i in 0...self.max_size() {
                let bi = self.blk_index(i+1, raw_addr);
                let info = unsafe { self.get_info_mut(i+1)};
                if info.is_split_set(bi) {
                    sizei = i;
                    Break;
                }
            }
            #[cfg(debug_assertions)]
            if sizei == self.nsizes {
                panic! ("allocator: dealloc cannot recycle ptr")
            }
    
            Check the layout
            #[cfg(debug_assertions)]
            if layout.size() > self.blk_size(sizei) {
                panic!("allocator: layout {:?} > blk size {}", layout, self.blk_size(sizei));
            }
    
            Free and coalesce
            while sizei < self.max_size() {
                let bi = self.blk_index(sizei, raw_addr);
                let buddyi = if bi % 2 == 0 { bi+1 } else { bi-1 };
                let info = unsafe { self.get_info_mut(sizei)};
                info.alloc_set(bi, false)
                
                Test if buddy is free
                if info.is_alloc_set(buddyi) {
                    Break;
                }
                let buddy_addr = self.blk_addr(sizei, buddyi);
                unsafe { (buddy_addr as *mut List).as_mut().unwrap().remove(); }
                if buddyi % 2 == 0 {
                    raw_addr = buddy_addr
                }
    
                Coalsce and continue
                sizei += 1;
                let spliti = self.blk_index(sizei, raw_addr)
                let info = unsafe { self.get_info_mut(sizei)};
                info.split_set(spliti, false)
            }
    
            let info = unsafe { self.get_info_mut(sizei)};
            unsafe { info.free.push(raw_addr); }
        }
    
    ```

    **The following is a short hand drawing of a `Buddy` system:**

    ![](.../../slides/assets/buddy.jpg)

    **User's attitude**

    In the user state, we use `sys_sbrk` system calls to allocate memory for user processes, while `sys_sbrk` uses `Buddy System` to allocate memory. So in the user state we only need to implement a bidirectional list to manage the memory allocated per system call. After each system call allocates memory, we record the currently allocated information on the header of the assigned pointer and move the available memory pointer back `_of::<Frame>()` to the end of the chain that is maintained.

    ```rust
    // User memory allocator based on linked list.
    pub struct UserAllocator {
        base: UnsafeCell<Frame>
    }
    
    // frame record every memory allocation information,
    // which will write in the header of allocation memory.
    #[repr(C)]
    #[Derive(Clone, Copy)]
    pub struct Frame {
        addr: *mut u8,
        Size: usize,
        next: Option<*mut Frame>,
        prev: Option<*mut Frame>
    }
    ```

    When the user program needs to allocate memory, we will traverse the list to find a memory of a sufficient size, and if the memory size of the block is exactly equal to the memory size to be allocated, it will be `pop` out of the list. Otherwise, we will move the corresponding head information backward to the memory size to be allocated, modify the head information, and return the memory pointer.

    When the user program releases memory, we get its pointer and size based on Layout's information. When the modified memory block is still in the list, the head pointer position is moved to modify the header information. If the memory block has been moved out of the linked list, we can add it back to the corresponding position of the linked list based on the corresponding header information.

Improvements in the stack

    Because the size of the kernel stack in `xv6-riscv` is 4KB (i.e. 1 page). But the size of 4KB is not enough for some processes, we uniformly set the stack size of each process to 16KB (i.e. 4 pages). When we have memory for each process mapping stack, we need to allocate 5 pages, of which 4 pages are used to map the stack memory, while the last page is not mapped, and the kernel is protected, and when the process is exposed to the protected page, it is forced to terminate without security vulnerabilities.

#3: Description of Implementation

See other documentation in this directory, in addition to which we also provide detailed comments in the code. For reasons of time, the document is vacant or not updated.

- [Environmental construction] (env.md)
- [gdb debugging] (gdb.md)
- [Start] (boot.md)
- [lock] (lock.md)
- [Interrupt] (interrupt.md)
- [Physical Memory Allocation] (alloc.md)
- [Virtual memory] (vm.md)
- [Process and Thread] (process.md)
- [Scheduler] (Scheduler.md)
- [xv6 file system] (xv6 file system.md)

###4 Problems and Solutions

**1. Kernel stack**

Since in the boot boot phase of the operating system, we only assign 4KB of kernel stack size to each core:

```assembly
    # qemu -kernel starts at 0x1000. the instructions
    # there seems to be provided by qemu, as if it
    # were a ROM, the code at 0x1000 jumps to
    # 0x8000000, the _entry function here,
    # in machine mode. each CPU starts here.
    Section .text
    .globl_entry
The entry:
	Set up a stack for Rust.
    # stack0 is below declared,
    # with a 16KB stack per CPU.
    # sp = stack0 + (hartid * 16384)
    # PS: 16KB stack for few stack bumps
    La Sp, stack0
    li a0, 16384
	csrr a1, mhartid
    addi a1, a1, 1
    mul a0, a0, a1
    add sp, sp, a0
	# jump to start() in start.rs
    Call to start

    Section .data
    .align 4
The stack0:
    .space 16384*8 #8 is NCPU in param.rs
```

When I realized the mapping of virtual memory and physical memory, I once set the return parameter of a function to the page table, and the memory size of the page table was exactly 4KB, which caused a bug to occur at runtime, regardless of the interruption point or the use of `GDB` debugging did not find the problem.

Then when writing the network stack, when using `lazy_static` to allocate memory for the message buffer, `Instruction Page Fault` is found to be caused by the stack memory overflow, so I then modify the kernel stack memory size from 4KB to 16KB (i.e. 4 pages), and the stack space of each process is modified to 16KB, which is reflected in the mapping process of the page table for the process.

**2. Deadlock**

Since I wanted to go to the test time interrupt, I output in the `clockintr() function, and found that the program was not output and deadlocked. After the query, it was found that the spin lock did not implement the `push_off()` and `pop_off()` methods. `push_off() and `pop_off()` were encapsulated with the `intr_on()` and `intr_off()` methods, and the interrupt should be closed when the lock was obtained.Even worse, the current thread may already hold the lock, in which case it will cause a single thread deadlock.

**3. Interrupts of equipment**

After writing `console`, it is planned to try the `UART` and `Virtio Disk` interrupts and find that the external interrupt cannot be received. At first it was thought to be a problem with `qemu`, but the commissioning found that it was not `qemu`.

**4. Interrupt nesting**

After opening the `sie` bit of `status`, it was found that the `Instruction Page Fault` was more strange as to why the opening interrupt occurred, and later found that the clock interrupt occurred due to the opening interrupt, and I mistakenly updated the program counter in all interrupts and exceptions, so the exception was generated again after each update of the clock.

#5: Summary and Prospects

**1. Summary of work**

- Completed the porting of the core to rust `xv6-riscv`

- Familiar with how to write an operating system and various debugging methods using Rust
- Optimize physical memory allocation with `Buddy System` and `slab` and use it as a `GlobalAllocator` to use Rust's `alloc` library
- Achieves basic network stack protocol and implements partial network card drivers 

**2. Future prospects**

- Implementation of the network protocol stack so that it can send and receive network packets
- Writing and testing user programs
- Transplant `RustSBI` to `xv6-rust`
- Port `xv6-rust` to K210

**3. Unresolved issues**

- System calls get stuck in the springboard page when they return to the kernel state from the user state, and are suspected to be an abnormal nesting problem, but it is more difficult to debug using `gdb` .

##6, Division of Labor

- Qi Zhenxiang: kernel implementation, design document writing, user library implementation
- 闫璟: Part of the document writing
- Li Fengjie: Some user program writing

####7, The Code Structure

The code structure of the project is as follows:

```
The Allocator
├ – Bin
├ – DOCS
│ └ - static
├ -fs-lib
│ ├ – Src
│ └ – Target
│                 
by Kernel
│ ├ . . cargo
│ └ – Src
│ ├ - Asm
│ ├ - console
│ ├ - Define
│ ├ -driver
│ ├ - fs
│ ├ – interrupt
│ ├ – Lock
│ ├ - Logo
│ ├ - memory
│ │ └ - mapping
│ ├ - - Net
│ ├ - Process
│ ├ - Register
│ ├ – syscall
│ └ - - test
├ – MKFS
│ ├ – Src
│ └ – Target
├ – User
│ ├ . . cargo
│ └ – Src
│ └ - bin
└ – utils
```

`allocator` implements `Buddy System` as a `crate` alone, introduced in the kernel

`bin` to store binary executables for user programs

`docs` is used to store documents

`fs-lib` is used to implement some basic file system data structures and methods for `mkfs`

`kernel` is used to store code for the operating system kernel

- `asm` to store assembly code
- `console` is used to store input-output-related code, including implementations of `uart` and `console`
- `define` is mainly used to store the definition of constants
- `driver` Store device drive code
- `fs` implementation of storing file system
- `interrupt` stores the handler of the `trap`
- `lock` implements `spinlock` and `sleeplock`
- `logo` for the project logo
- `memory` storage memory allocation and address mapping and other related code implementations
- `net` Stores implementation code for the network stack
- `process` Stores code related to scheduling of processes
- `register` is mainly used to store the relevant code of various registers
- `syscall` is used to store implementations of system calls
- `test` mainly stores some code for testing in the kernel

`mkfs` is used to store the implementation code for building file images

`user` is used to store user programs, including libraries, system calls, and user program implementations

`utils` is mainly used to store some of the tool code that helps the program run

###8, The Game Gain

Using Rust to implement the operating system kernel from scratch, a deeper understanding of how the operating system and hardware work together
- Familiar with the development of the Rust language on system-level applications
- Enhanced ability to debug the program
- Improved ability to develop larger projects

####9, the reference is achieved

- [xv6-riscv] (https://github.com/mit-pdos/xv6-riscv)

- [xv6-riscv-rust] (https://github.com/Jaic1/xv6-riscv-rust)

- [rCore-Tutorial-v3] (https://github.com/rcore-os/rCore-Tutorial-v3)

####10, References

- [book-riscv-rev1] (https://pdos.csail.mit.edu/6.828/2020/xv6/book-riscv-rev1.pdf)
- [PCI/PCI-X Family of Gigabit Ethernet Controllers Software Developer's Manual][https://pdos.csail.mit.edu/6.S081/2020/readings/8254x_GBe_SDM.pdf]
- [rCore-Tutorial-Book 3rd Edition] (https://rcore-os.github.io/rCore-Tutorial-Book-v3/index.html)
