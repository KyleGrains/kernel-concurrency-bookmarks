# Linux Kernel Concurrency Bookmarks

Short notes for revisiting kernel synchronization topics while reading *Linux Device Drivers*.

## Seqlock

Use a seqlock when:

- reads are very frequent
- writes are rare and short
- readers can cheaply retry
- the data is small enough to read/copy again

Core idea:

```text
writer:
    mark sequence as changing
    update data
    mark sequence as stable

reader:
    read sequence
    read data
    check sequence again
    retry if writer changed data during the read
```

Simplified reader pattern:

```c
do {
    seq = read_seqbegin(&lock);

    a = data.a;
    b = data.b;

} while (read_seqretry(&lock, seq));
```

Simplified writer pattern:

```c
write_seqlock(&lock);
data.a = new_a;
data.b = new_b;
write_sequnlock(&lock);
```

Why it works:

- readers do not block writers
- writers do not wait for readers
- readers may see inconsistent data, but detect it and retry

Avoid seqlock when:

- writers are frequent
- reads are expensive to repeat
- readers cannot tolerate retry loops
- readers follow pointers to objects that writers may free

Mental model:

```text
Read the page. If someone edited it while you were reading, reread the page.
```

## RCU

RCU means Read-Copy-Update.

Use RCU when:

- reads are very frequent
- writes are less frequent
- readers mostly follow pointers or traverse structures
- readers must be extremely cheap
- old versions can remain alive temporarily

Core idea:

```text
reader:
    enter RCU read-side critical section
    read current pointer
    use object
    exit RCU read-side critical section

writer:
    allocate/copy a new version
    modify the new version
    publish the new pointer
    wait until old readers are done
    free old version
```

Simplified reader pattern:

```c
rcu_read_lock();

p = rcu_dereference(global_ptr);
use(p);

rcu_read_unlock();
```

Simplified writer pattern:

```c
new = kmalloc(sizeof(*new), GFP_KERNEL);
*new = updated_data;

old = rcu_dereference_protected(global_ptr, lockdep_is_held(&lock));
rcu_assign_pointer(global_ptr, new);

synchronize_rcu();
kfree(old);
```

Why it works:

- readers do not take normal locks
- readers can keep using the old object safely
- writers publish a new object instead of modifying in place
- old objects are freed only after a grace period

Avoid RCU when:

- writes are frequent
- you need immediate deletion
- readers need to modify the protected data
- a normal mutex/spinlock would be simpler and fast enough

Mental model:

```text
Readers keep reading the old copy.
Writer publishes a new copy.
Old copy is thrown away only after old readers are finished.
```

## Seqlock vs RCU

```text
seqlock:
    protects small snapshots
    readers may retry
    writer updates data under sequence protection

RCU:
    protects pointer-based data structures
    readers usually do not retry
    writer replaces/copies data and frees old version later
```

## First-Pass Rule

For driver work, learn these in this order:

1. mutex/semaphore
2. spinlock
3. spin_lock_irqsave
4. circular buffer
5. atomic_t
6. seqlock
7. RCU

Do not reach for seqlock or RCU first. They are tools for specific read-heavy patterns.

