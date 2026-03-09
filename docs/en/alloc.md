Dynamic memory allocation

## List Allocation

### Allocation by Page

In this section, we will use the end address from the `kernel data` to the address of the `PHYSTOP` for dynamic memory allocation, and in the implementation of xv6, the physical frame will be allocated by physical page, and for each available physical page, we use a struct to describe:

```rust
pub struct run
    next: Option<NonNull<Run>
}
```

This struct is a linked-list structure that gets the next available physical page for each struct's `next`. In order to be able to access the struct in multiple threads, we also need to add a `Send trait` to the struct:

```rust
unsafe impl send for run{}
```

At the same time, we need to set some methods for the linked list to facilitate operation:

```rust
Impl Run{
    pub unsafe fn new(ptr: *mut u8) -> NonNull<Run>{
        let r = ptr as *mut Run;
        write(r, run{next: None})
        NonNull::new(r).unwrap()
    }

    pub fn set_next(&mut self, value: Option<NonNull<Run>>){
        self.next = value
    }

    pub fn get_next(&mut self) -> Option<NonNull<Run>{
        self.next.take()
    }
}
```

This struct accepts a variable naked pointer of type `u8` to return `NonNull<Run>`, and the reason for wrapping it is because if `Option<Run>` is used, an infinite size occurs, and the naked pointer is used directly, and because Rust allows `*const T` and `*mut T` to convert each other, this will result in some unsafe operation.

We took an alias for `FreeList' for the `Run' to represent the free physical page:

At the same time, we created a `KMEM` as a global idle physics page representation:

```rust
lazy_static!{
    static ref KMEM: Spinlock<FreeList> = Spinlock::new(FreeList { next: None}, "kmem");
}
```

### Allocation of Memory

For the allocation of memory, we will use a function of `kalloc` to implement:

```rust
Allocate one 4096-byte page of physical memory.
Returns a pointer that the kernel can use.
Returns 0 if the memory cannot be allocated.

pub unsafe fn kalloc() -> Option<*mut u8>{
    let mut guard = (*KMEM).acquire();
    let r = guard.get_next();
    if let Some(mut addr) = r{
        guard.set_next(addr.as_mut().get_next());
    }
    drop (guard);
    // (*KMEM). release();

    match r {
        Some(ptr) => {
            let addr = ptr.as_ptr() as usize;
            // for i in 0..PGSIZE{
            / write_volatile((addr + i) as *mut u8, 5);
            // }
            Some(addr as *mut u8)
        }
        None => None
    }
}
```

In our implementation, we first get the lock to get a lock guard variable (see the lock's documentation for the lock. We get the available memory address and return a pointer to the top address of the idle physical page and remove the physical page from the list of free physical pages. If we can find the physical page back to the top of the physical page, we return None.

### Release the memory

Similarly, for the release of memory addresses, we use `kfree` to implement:

```rust
pub unsafe fn kfree(pa: physicalAddress)
    let addr:usize = pa.into();

    if (addr % PGSIZE !=0) || (addr < end as usize) || addr > PHYSTOP.into(){
        Panic! ("kfree")
    }

    // Fill with junk to catch dangling refs.
    // for i in 0..PGSIZE {
    // write_volatile((addr + i) as *mut u8, 1);
    // }

    let mut r:NonNull<FreeList> = FreeList::new(addr as *mut u8);
    let mut guard = (*KMEM).acquire();

    r.as_mut().set_next(guard.get_next());
    guard.set_next(some(r));
    drop (guard);

    // (*KMEM). release();

}
```

This function takes a physical address, first determines whether it is a legitimate physical top address, and then adds it to the physical page idle list.

### Initialization

In addition, we also implement a `freerange` function, which is only used for initialization:

```rust
unsafe fn freerange(pa_start:PhysicalAddress, pa_end:PhysicalAddress)
    println!("enter freerange...");
    let mut p = pa_start.page_round_up();
    let end_addr:usize = pa_end.into();
    println!("enter loop...")
    println!("start addr: {:#x}", p);
    println!("end addr: {:#x}", end_addr);
    while p < end_addr{
        // println!("page addr: {:#x}", p);
        kfree(PhysicalAddress::new(p))
        p += PGSIZE;
    }
    println! ("freerange done...")

}
```

As you can see, this function accepts a starting and end addresses to add them to the free physical page list.

In the initialization function, we pass `end` and `PHYSTOP` as the starting and termination addresses of dynamic memory allocation:

```rust
unsafe fn kinit(){
    println!("kinit...")
    println!("kinit: end={:#x}", end as usize);
    freerange(PhysicalAddress::new(end as usize), PhysicalAddress::new(PHYSTOP.into()))
    println! ("kinit done...")

}
```



## Partner Memory Allocation

To be developed...

