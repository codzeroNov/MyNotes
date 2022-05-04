# Process vs Thread (frequently asked 8%)

- Processes are independent and contain their own state information. Threads run within a process and share the same state and memory space
- Processes have to communicate with each other using interprocess communication mechanisms. Threads communicate directly because they share the same variables.
- Context switching between threads is faster than between processes.
- What is a mutex? Semaphore? Binary semaphore? (frequently asked 4%)
- mutex is a binary lock, same as binary semaphore. Generic semaphore allows access to more then one thread at time (depending on number of resources)

# What is a small IO?What is a large IO? Numbers for seek times and scan rates?

- Small IO is where a single(or few) block random IO is done. Main cost is seek time (~10ms)
- Large IO is where large number of blocks are read sequentially. Seek time is amortized. Main cost is scan rate.

# What is RAID?

- Redundant Array of Independent Disks (expanding the acronym is not important - candidate should intuitively know what RAID is)
- Two main ideas: mirroring and striping. Mirroring gives protection against drive failure and striping gives performance. This is the gist

# What is a transaction?

- Candidate does not necessarily need to define ACID (Atomicity, Consistency, Isolation, Durability) but should intuitively know what it means to provide transactional semantics in a concurrent, multi-threaded environment. Should at least describe atomicity and isolation.

# Stack vs Heap (frequently asked 5%)

- Stack is for a thread/program to allocate local program variables. Heap is for dynamic allocation of memory.  Stack memory is allocated at compile time. Heap memory is allocated at runtime.
