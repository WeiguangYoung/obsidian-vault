---
{"dg-publish":true,"permalink":"/knowledge/course/mit6.1810/14-协调（sleep与wakeup）/","title":"MIT 6.1810 第 14 讲 协调（sleep与wakeup）","tags":["MIT6.1810","操作系统","xv6","讲义"],"dg-note-properties":{"title":"MIT 6.1810 第 14 讲 协调（sleep与wakeup）","date":"2026-11-13","tags":["MIT6.1810","操作系统","xv6","讲义"]}}
---


# 第 14 讲 · 协调与睡眠唤醒（6.1810 2026 Lecture 14: Coordination (sleep&wakeup)）

> 麻省理工学院 6.1810 操作系统工程 · 2026 秋季
> 官方讲义原文：`pdos.csail.mit.edu/6.1810/2026/lec/l-coordination.txt`

返回 [[knowledge/course/mit6.1810/index\|📚 6.1810 目录]] · [[首页\|🏠 首页]]

---

```
# plan
  review three points about xv6 locks and threads
  sleep & wakeup
  termination

review #1: why does acquire() disable interrupts?
  look at spinlock.c
  interrupt handler code might need the same lock
  but the interrupted code, which holds the lock, can't release
    until the interrupt handler returns!
  could acquire() spin first and only afterwards push_off()?
  release() calls pop_off()

review #2: why hold p->lock across swtch()?
   yield()                     scheduler()
   -------                     -----------
   acquire(&p->lock);
   p->state = RUNNABLE;
   swtch();
                               swtch();
                               ...;
                               release(&p->lock);

Q: why is p->lock held across swtch()?
  1. prevent another core's scheduler from seeing p->state == RUNNABLE
     until after original core has finished saving registers
     until after original core has stopped using thread's stack
  2. prevent timer interrupt from calling yield() during swtch()
     (remember: acquire() turns off interrupts)
     2nd swtch() would overwrite already-saved registers in context

review #3: why does xv6 never hold a spinlock when yielding the CPU?
   (other than p->lock)
   on a single-core machine, imagine this:
     P1           P2
     acq(x)
     sched()
                  acq(x)

Q: what is the problem here, on a single-core machine?
   P2 will wait forever:
     P2 will spin waiting for P1 to release x
     P2 disables interrupts before spinning
     No clock interrupts, so P2 won't yield() and P1 won't run
     So P1 won't ever release x
   possible even on multi-core, with more locks/threads
   solution: forbid holding lock when yielding the CPU!
     (other than p->lock)

# topic: coordination
  kernel threads often wait for specific events / "conditions":
    a disk read is complete (signaled by an interrupt)
    a pipe has data to read (written by another process)
    a child has exited

# Q: why not just have a while-loop that spins until condition is true?
  pipe read:
    while buffer is empty {
    }
  pipe write:
    put data in buffer

# better solution: coordination primitives that yield the CPU
  operating systems use a variety of schemes
    e.g. barriers, semaphores, event queues, condition variables.
  xv6 uses sleep & wakeup
    similar to condition variables

# example: look at uartwrite() and uartintr() in kernel/uart.c
  recall: the UART is the device hardware that connects to the console
    h/w takes a long time to send each byte, perhaps a millisecond
  so uartwrite() has to wait after each byte until UART ready for next
  keeps track w/ tx_busy
  if tx_busy, sleep()
    gives up the CPU
  uartintr() clears tx_busy when UART is done
    and wakeup()

# xv6's sleep/wakeup interface
  sleep_prepare(wait_chan)
  sleep()
  --------
  wakeup(wait_chan)

# semantics
  the wait_chan links the sleep and wakeup
    just a number, tells wakeup which sleeping threads to wake
    can be anything known to both sleeper and waker
    usually the address of some relevant object e.g. &tx_chan
  sleep_prepare sets the "wait channel"
  sleep yields the CPU
  wakeup wakes up every process in sleep() w/ the given wait_chan

# sleep/wakeup don't know what event we're waiting for
  the code manages the event/condition separately:
    tx_busy := 0 is what uartwrite() is waiting for
    set by uartwrite(); cleared by uartintr()

# why the release/sleep/acquire?
  remember kernel code is not allowed to hold a lock while sleeping

# why the separate sleep_prepare() and sleep()?
  could we eliminate sleep_prepare() and have just sleep(wait_chan)?
  no straightforward way to do this

# suppose just sleep(chan); how would we implement?
  here's a BROKEN sleep/wakeup
  [on board, not in vi]
  broken_sleep(chan)
    acquire(&p->lock)
    p->state = SLEEPING;
    p->chan = chan;
    swtch(...);
    (now scheduler() won't run it -- not RUNNABLE)
  wakeup(chan)
    for each p:
      acquire(&p->lock);
      if p->state == SLEEPING && p->chan == chan:
        p->state = RUNNABLE

# I'll try it -- modify uart.c to call broken_sleep()
  // comment out sleep_prepare()
  release(&tx_lock);
  broken_sleep(&tx_chan);
  acquire(&tx_lock);

make qemu
cat README

the problem:
  uartwrite() sees tx_busy != 0 (UART not done with previous byte)
  releases tx_lock (so that it can yield the CPU)
    which enables interrupts
  interrupt!
    uartintr() sets tx_busy = 0
    uartintr() calls wakeup() -- but nothing to wake up!
  then uartwrite() calls broken_sleep()
  uartwrite() went to sleep EVEN THOUGH UART TX INTERRUPT WAS DONE
    and tx_busy is zero...
  now there is nothing to wake up uartwrite(), it will sleep forever
    really, until a UART *input* interrupt

this is the "lost wakeup" problem
  must be solved by any coordination mechanism

could uartwrite() have *not* released the lock?
  no #1: not allowed
  no #2: uartintr() needs that lock

we need to eliminate the window between uartwrite()'s check of
  the condition, and sleep() marking the process as asleep on the wait chan.

xv6's solution to lost wakeups:
  acquire
  if condition is false
    sleep_prepare(wait_chan)
    release
    sleep()

(repair uart.c)

look at sleep_prepare(chan) in proc.c
  just sets p->chan, nothing else

look at sleep() in proc.c
  returns if p->chan zero
  otherwise p->state = SLEEPING, yield the CPU

look at wakeup(chan) in proc.c
  it scans the process table, looking for any p->chan == chan
  it grabs each p->lock
  sets p->chan to zero
  changes p->state from SLEEPING to RUNNABLE

key facts:
  sleep_prepare() sets p->chan = wait_chan
  wakeup() sets p->chan to zero
  sleep() returns immediately if p->chan is zero

for uartwrite(), look at times that wakeup() might be called
  by uartintr(), which needs tx_lock, and clears tx_busy
  - between release() and sleep()?
    wakeup() clears p->chan, so sleep() will return immediately
  - just after condition check (i.e. just after while(tx_busy != 0))?
  - while sleep() is executing?
    look at sleep() and wakeup() in proc.c
    - interrupt just before sleep()'s acquire()?
    - interrupt after sleep()'s acquire()?

thus: no lost wakeup
  
rule: sleep_prepare() must be called before corresponding wakeup()
  could possibly be called
  usually enforced with a lock

Q: why does uartwrite() wrap the sleep() in a loop?
  i.e. re-checks the condition after sleep() returns, may sleep again
  could be multiple processes waiting to write the UART,
    interrupt wakes them all up,
    one thread runs first, sets tx_busy,
    others have to *not* write to the UART,
    so they must re-check after sleep() returns
  all xv6 calls to sleep are wrapped in loops, to re-check

Q: why is it enough for wakeup() to just set p->state = RUNNABLE?
   why does that cause the thread to run?

# A non-interrupt example: piperead() in pipe.c
  line 129: sleep()
  line 122: the condition is data waiting to be read
  pipewrite() calls wakeup() at the end
  what is the race if piperead() used broken_sleep()?
    i.e. release the lock, then broken_sleep()
    lost wakeups are not just about interrupts!
  note the the loop around sleep()
    multiple processes may be reading the same pipe
  Q: why the wakeup() at the end of piperead()?

# summary of xv6 rules:
  prevent wakeup() between check and sleep_prepare() (lock...)
  release/re-acquire around sleep()
  re-check condition (i.e. a loop)

*************

# another coordination challenge -- how to shut down a thread?
  look at how kill works()

# kill() puzzle: cannot just destroy target thread X
  what if X is executing on another core?
    can't just free its kernel stack or user memory!
  what if X holds locks?
  what if X is in the middle of a complex update to kernel data structures?
    must let it finish!
  so: kill() can't actually destroy the target.

# kill() solution: 
  kill() asks the target to destroy itself as soon as convenient
  proc.h
    kill() sets p->killed
  trap.c
    the target process itself checks for p->killed
      and calls kexit() itself
    not in the middle of anything here, and no locks are held
      so it's safe to exit

# what if kill() target is SLEEPING -- in sleep()?
  it may sleep for a long time: how to get its attention?

# xv6 solution to kill() of sleep()ing process
  kkill() in proc.c
    changes SLEEPING to RUNNABLE -- like wakeup()
  so sleep() will return, probably before condition is true
    another reason to always wrap sleep a loop to re-check condition

# is it OK to exit from a sleep() loop if p->killed?
  sometimes yes: waiting for UART input, pipe read
  sometimes no: waiting for disk while creating file
  some sleep loops check for p->killed
    pipe.c piperead()
    otherwise read could hang indefinitely for a killed process
  some sleep loops don't check p->killed
    e.g. virtio_disk.c
    can't just quit since maybe midway through a file system update
  so a kill()ed process may continue for a while
    but usertrap() will exit() after the system call finishes

# what happens in kexit() -- how does a process destroy itself?
  in proc.c
  close files &c
  parent is in wait()...
  state = ZOMBIE tells parent child has exited
  wakeup(p->parent)
  swtch() for the last time
  parent's kwait() frees child's last resources e.g. proc[] slot

# Next week:
  Networking
```
