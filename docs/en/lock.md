# The Lock

## Spinlock (Spinlock)

The spin locks we implement are defined as follows:

```rust
#[debug(Debug, Default)]
pub struct spinlock<T: ?Sized>{
    locked: AtomicBool,
    name: &'static str,
    cpu_id: Cell<isize>,
    data:UnsafeCell<T>
}
```

`locked` is declared by `core::atmoic::AtomicBool`, which is an atomic boolean type, which is a thread-safe value. The value of the data is `UnsafeCell` to wrap (wrap), which indicates that there will be some unsafe operations that will act in the value of the inner package, and we will not have the means to obtain a variable reference to the internal variable. We can use the `.get()` method to operate on the internal.

For a lock variable, we need to implement the `acquire()` and `release()` methods for it:

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

In our implementation, for the `acquire` method, we first need to close the interrupt and wait for the lock variable to be obtained and atomically lock it, returning a `SpinlockGuard' variable after the variable is locked.

For the `release` method, we first need to determine the state of the current lock, and when the lock is `acquire' state, we unlock it and open it to interrupt.

The definition of `SpinlockGuard' is as follows:

```rust
pub struct spinlockGuard<'a, T>{
    spinlock:&'a Spinlock<T>
}
```

The lock guard returns a lock variable for the lock-to-lock thread to operate. At the same time, we overload the dereference operator, which allows the lock-enable thread to call the data method:

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
```



The Sleep Lock (Sleeplock)

To be developed...
