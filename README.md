# nanosleep
Analyzing Sleep Function Accuracy in DragonFly BSD

Understanding sleep function precision is essential not only for kernel developers but also for anyone tuning system performance for time-sensitive applications. One practical example is audio configuration on FreeBSD, where the `kern.timecounter.alloweddeviation` sysctl parameter directly impacts latency and timing accuracy. This parameter controls the maximum permitted drift between software and hardware clocks — a critical factor for real-time audio processing, where even small timing inconsistencies can cause audible glitches, buffer underruns, or synchronization issues.

This article by Paul Herman is republished here for archival purposes and serves as supplementary material providing deeper technical context for the FreeBSD audio configuration guide [FreeBSD Audio Setup: Bit-Perfect Playback with Equalizer and Real-Time Optimization](https://m4c.pl/blog/freebsd-audio-setup-bitperfect-equalizer-realtime/). The analysis presented in this article, originally conducted on DragonFly BSD, provides valuable insights into the mechanics of process wake-up latency and system clock behavior that apply broadly across BSD systems.

---


# How Long Does Your nanosleep() Really Sleep?

## Analyzing Sleep Function Accuracy in DragonFly BSD

**Author:** Paul Herman  
**Original publication:** January 8, 2004  
**System:** DragonFly BSD 1.0-CURRENT

---

## Introduction

Have you ever wondered why the command `time sleep 1` never returns exactly 1 second? You might know the answer, but if you think you do — you might also be wrong.

Accurate time keeping and event triggering is becoming more and more affordable as PC hardware becomes increasingly faster. Network communication is now a matter of microseconds and system call round trip times are accomplished within nanoseconds. This document examines the accuracy of the `nanosleep()` and `tsleep()` kernel functions under DragonFly BSD 1.0-CURRENT.

## The Problem

Suppose we wish to trigger an event at an **exact time** on the clock — for example, every second just as the second hand moves. In pseudo code, we wish to do something like:

```
forever
    now = look_at_clock()
    s = next_second - now
    sleep(s)
    trigger_event()
end
```

So, if our clock reads 10:45:22 and the second hand reads 22.874358, then by the time our sleep function wakes up, it should be **exactly** 10:45:23. Similarly, the next call to `trigger_event()` will happen at **exactly** 10:45:24.

So much for the theory — let's try this in the real world and see how close we can get.

## Observations

### Unexpected Results

A simple test program `wakeup_latency.c` was written to sleep until the next second and then print the difference in seconds between the actual time and the time expected. On a GHz machine, this was expected to be very accurate, but the results were astonishing:

```
0.019496
0.019506
0.019516
0.019525
0.019535
```

That's **19 ms** later than expected! This is nearly **twice** the default system tick length of 10 ms based on 100 Hz. This is unexpected and certainly undesirable.

### Cross-Platform Comparison

The test program was run on the following systems:

- DragonFly BSD 1.0-CURRENT (i386)
- Darwin 7.0.0 / Panther (iMac G4)
- FreeBSD 4.9-STABLE (i386)
- Linux 2.4.21 (i386)
- Solaris 8 (sparc)

All operating systems had their default Hz unchanged (which is likely 100 in all cases).

### Key Observations

Based on 15-minute tests (900 seconds), the following patterns emerged:

1. **All systems guarantee** sleeping at least as long as requested (all values are positive — a requirement by the Open Group's SUSv2)
2. **All except Darwin** don't even seem to make the effort to return when they should, which should be immediately after the second hand tick
3. **Most return** after at least one kernel tick has passed
4. **FreeBSD and DragonFly** exhibit a strange sawtooth pattern — is it clock aliasing?

An interesting note: this delay was process independent. No matter how many processes were running, all `usleep()` calls at the same time took the same time.

### Effect of Hz Frequency

What happens when you keep the OS and hardware constant and change only the Hz value?

Experiments revealed that:

- Increasing `kern.hz` does help, but **only up to around 1000**
- Beyond that, the speed improvement breaks down
- The problem is more complex than simple timer resolution

## Solutions

### Source Code Analysis

The first place examined was `kern/kern_time.c:nanosleep()`. Although it does truncate the precision of the request from nanoseconds to microseconds (and then from microseconds to ticks via `tvtohz()`), that alone shouldn't account for the differences observed.

However, a closer examination revealed a **bug in the `tvtohz()` function** (`kern/kern_clock.c` rev 1.12).

### The tvtohz() Bug

With `hz=100` and `ticks=10000`, the following discrepancies were found:

| Seconds   | Theoretical ticks (rint(seconds×Hz)) | tvtohz() |
|-----------|--------------------------------------|----------|
| 0.900000  | 90                                   | 91       |
| 0.990000  | 99                                   | 100      |
| 0.999000  | 100                                  | 101      |
| 1.000000  | 100                                  | 101      |
| 1.000001  | 100                                  | 101      |
| 1.000002  | 100                                  | 102      |
| 1.001000  | 100                                  | 102      |

This change was apparently introduced into FreeBSD rev 1.11 in 1994. It should be corrected, and the bug that it was meant to fix (`sleep(1)` exiting too soon) should be properly fixed.

After applying the appropriate patch, `usleep()` delays looked significantly better.

## Clock Aliasing

### Identifying the Problem

What about the sawtooth patterns? The kernel clock shouldn't behave that way — something is definitely wrong.

A keen eye will notice that the change to `tvtohz()` produced a slightly steeper or quicker frequency of the sawtooth wave. What does this mean?

Now that for any given `timeval`, `tvtohz()` returns fewer ticks, the system requires fewer ticks to cover the same time span. In other words, fewer "microseconds" to cover the same wall clock time — we have effectively (at least from the view of the Hz timer) sped up the system clock.

### The Movie Wheel Analogy

The Hz timer appears to be out of sync with the system hardware clock, producing aliasing. This is not unlike the aliasing seen in movies when automobile wheels appear to turn backwards. In that case, the camera shutter frequency is slightly out of sync with the turning of the wheels.

### Confirming the Theory

To confirm that the "wheels" were indeed out of sync, the same test program was run while simultaneously adjusting the clock frequency using `ntptime [-f freq]` (like adjusting the speed of the car). Sure enough, the observed delays could be adjusted any way desired.

Other evidence of a software error: the Hz timer is clocked off of the i8254 clock which is the same clock as the hardware timecounter. The i8254-based timecounter appears to run faster than the i8254 Hz timer — there **must** be a software bug.

### The Root Cause

Upon further investigation, Matt Dillon suggested that the drift could be caused by the cumulative effect of a system tick not being an integral multiple of an i8254 timer tick. This effectively speeds up the Hz timer to a speed faster than it should be.

### The Solution: PLL Loop

The solution was to periodically slow down the Hz timer by a small fraction so that on average, the Hz timer would tick exactly as it should.

This initial change produced very promising results, leading Matt Dillon to improve the design and develop a **PLL (Phase-Locked Loop)** in `clkintr()`. This loop slowly adjusts the frequency of the Hz timer against the wall clock timer (be it i8254 or TSC) to get it as close to the target Hz as possible.

A nice side effect of this change: because the Hz timer is skewed against the wall clock, any clock disciplining done to the wall clock time (by `ntpd`, for example) will also adjust the Hz rate accordingly.

## Results After Fixes

### Dramatic Improvement

After implementing the PLL patch, the same test revealed dramatic improvements. Due to the nature of the PLL which takes a few minutes to stabilize, there is initial clock aliasing until a lock on the frequency is attained. After that, the Hz timer appears very accurate indeed.

### Performance at Different Hz Rates

Testing with the PLL patch at different Hz rates (using the TSC as the wall clock timecounter) showed:

- Some slight improvement with higher Hz rates
- Both tested configurations performed very well
- Results generally stayed within **10 microseconds** of the expected time

This is about **a thousand times better** than some other operating systems!

At this fine resolution, you can even observe how other processes affect latency, such as the periodic syncer daemon. The difference is like looking through the Hubble telescope rather than binoculars.

## Conclusion

After asking a rather academic question such as "why isn't sleep very accurate?" — which may have little practical application — some minor deficiencies were uncovered that potentially have very **practical** applications, such as accurate bandwidth shaping as measured against a well-disciplined wall clock.

One can naturally ask "why are we limited to around 10 microseconds when the i8254 has about an 0.8 microsecond resolution?" But the changes already made represent a **significant** improvement in the standard Hz timer, and one can only be pleased with the results.

---

## Appendix: Frequently Asked Questions

### Q: Why don't you renice the process to a higher priority so the process will wake up sooner?

**A:** This was tried — no difference.

### Q: That's easy to explain — gettimeofday() is a system call, and system calls have lots of overhead. That's why it takes so long.

**A:** No. `clock_gettime()` takes only a total of about **500-800 nanoseconds** when called from userland on a GHz machine, and `nanotime()` takes only about **60 ns** when called directly from inside the kernel.

### Q: Just increase kern.hz and your delay will go away.

**A:** No, they won't. Sure, `usleep()` will return faster, but there is still a definite delay problem. See the graphs in the original presentation.

### Q: Use the TSC timer instead of the i8254. It has a much higher resolution.

**A:** Both were tried — no difference (before the PLL fix).

---

## Test Program

The following C program was used for testing wakeup latency:

```c
/*
 * Copyright (c) 2003 Paul Herman
 * All rights reserved.
 * BSD License
 */

#include <sys/time.h>
#include <sys/resource.h>
#include <time.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

#define ONE_SECOND  1000000L

int count = 200;
int debug = 0;

int main(int ac, char **av) {
    long s;
    double diff;
    struct timeval tv1, tv2;

    if (ac > 1 && av[1])
        count = strtol(av[1], NULL, 10);

    while(count--) {
        gettimeofday(&tv1, NULL);
        /*
         * Calculate the number of microseconds to sleep so we
         * can wake up right when the second hand hits zero.
         *
         * The latency for the following two statements is minimal.
         * On a > 1.0GHz machine, the subtraction is done in a few
         * nanoseconds, and the syscall to usleep/nanosleep is usually
         * less than 800 ns or 0.8 us.
         */
        s = ONE_SECOND - tv1.tv_usec;
        usleep(s);
        gettimeofday(&tv2, NULL);

        diff = (double)(tv2.tv_usec - (tv1.tv_usec + s)) / 1e6;
        diff += (double)(tv2.tv_sec - tv1.tv_sec);
        if (debug)
            printf("(%ld.%.6ld) ", tv2.tv_sec, tv2.tv_usec);
        printf("%.6f\n", diff);
    }
    return 0;
}
```

---

## References

- Original presentation: [DragonFly BSD nanosleep presentation](https://www.dragonflybsd.org/presentations/nanosleep/)
- DragonFly BSD clock.c revisions 1.8-1.10 (PLL implementation)
- FreeBSD kern_clock.c historical changes
- POSIX.1b nanosleep() specification
- Open Group SUSv2 sleep requirements

---

## Technical Context

### The nanosleep() System Call

Both `nanosleep()` and `clock_nanosleep()` allow suspending the calling thread for an interval measured in nanoseconds. The suspension time may be longer than requested due to the scheduling of other activity by the system.

Supported clock IDs in DragonFly BSD include:

- `CLOCK_MONOTONIC`
- `CLOCK_MONOTONIC_FAST`
- `CLOCK_MONOTONIC_PRECISE`
- `CLOCK_REALTIME`
- `CLOCK_REALTIME_FAST`
- `CLOCK_REALTIME_PRECISE`
- `CLOCK_SECOND`
- `CLOCK_UPTIME`
- `CLOCK_UPTIME_FAST`
- `CLOCK_UPTIME_PRECISE`

### DragonFly BSD's Approach to SMP

DragonFly BSD has virtually no bottlenecks or lock contention in-kernel. Nearly all operations are able to run concurrently on any number of CPUs. The VFS support infrastructure, user support infrastructure, process and threading infrastructure, storage subsystems, networking, memory allocation and management, and all other aspects of kernel design have been rewritten with extreme SMP performance as a goal.

This architectural approach makes accurate timing even more critical for the operating system's design philosophy.

---

*This article is based on Paul Herman's original research and presentation from January 2004, documenting significant improvements made to DragonFly BSD's timing accuracy.*
