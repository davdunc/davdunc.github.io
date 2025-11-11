---
layout: post
title: "Debugging Kernel Panics in Production Linux Systems"
date: 2025-11-05 14:30:00 -0500
categories: [linux, debugging]
tags: [kernel, debugging, production, crash-dumps]
---

Last week, I encountered a series of kernel panics on production systems running a custom Linux distribution. Here's how I tracked down and resolved the issue.

## The Problem

Production servers were experiencing random kernel panics, roughly once every 48-72 hours. The systems were running kernel 6.1.x with custom patches for specific hardware support.

Initial symptoms:
- No consistent pattern to the crashes
- Various call traces in kernel logs
- Systems would hard-lock requiring physical reboot

## Investigation Process

### Step 1: Enable kdump

First, I ensured kdump was properly configured to capture crash dumps:

```bash
# Install kdump tools
dnf install kexec-tools

# Configure crashkernel reservation in grub
grubby --update-kernel=ALL --args="crashkernel=256M"

# Enable and start kdump service
systemctl enable kdump
systemctl start kdump
```

### Step 2: Analyze Crash Dumps

Once I had some crash dumps, I used `crash` to analyze them:

```bash
crash /usr/lib/debug/lib/modules/6.1.0/vmlinux /var/crash/vmcore
```

The backtrace revealed the issue was in the network driver code:

```
crash> bt
PID: 0      TASK: ffff88803fc00000  CPU: 4   COMMAND: "swapper/4"
 #0 [ffffc90000013d08] machine_kexec at ffffffff81063eb4
 #1 [ffffc90000013d60] __crash_kexec at ffffffff811a2d72
 #2 [ffffc90000013e28] panic at ffffffff8109c5e1
 #3 [ffffc90000013ea8] oops_end at ffffffff81028d3a
 #4 [ffffc90000013ed0] do_general_protection at ffffffff81c03c6e
 #5 [ffffc90000013ef8] general_protection at ffffffff81c03491
 #6 [ffffc90000013f40] custom_net_poll at ffffffffc05a3210 [custom_driver]
```

### Step 3: Root Cause

After reviewing the driver code, I found a race condition in the interrupt handler:

```c
// Buggy code
static irqreturn_t custom_net_interrupt(int irq, void *dev_id)
{
    struct net_device *dev = dev_id;
    struct custom_priv *priv = netdev_priv(dev);

    // BUG: priv->buffer could be freed by another thread
    if (priv->buffer) {
        process_packet(priv->buffer);
    }

    return IRQ_HANDLED;
}
```

The fix required proper locking:

```c
static irqreturn_t custom_net_interrupt(int irq, void *dev_id)
{
    struct net_device *dev = dev_id;
    struct custom_priv *priv = netdev_priv(dev);
    unsigned long flags;

    spin_lock_irqsave(&priv->lock, flags);
    if (priv->buffer) {
        process_packet(priv->buffer);
    }
    spin_unlock_irqrestore(&priv->lock, flags);

    return IRQ_HANDLED;
}
```

## Testing the Fix

Before deploying to production:

1. Reproduced the issue in a test environment using stress testing
2. Verified the fix with extended stress tests (72+ hours)
3. Ran static analysis tools (sparse, coccinelle)
4. Peer review of the patch

## Deployment

Rolled out the fix gradually:
1. Canary deployment on 2 servers
2. 48-hour monitoring period
3. Gradual rollout to remaining fleet
4. Continued monitoring for 2 weeks

## Lessons Learned

1. **Always enable kdump on production systems** - You never know when you'll need it
2. **Race conditions are subtle** - Code review and static analysis help, but aren't foolproof
3. **Gradual rollouts save lives** - Even when you're confident in a fix
4. **Document everything** - Future you will thank present you

## Tools Used

- `kdump/kexec-tools`: Crash dump collection
- `crash`: Crash dump analysis
- `sparse`: Static analysis for kernel code
- `ftrace`: Function tracing for debugging
- `perf`: Performance profiling

---

*Working on kernel issues? I'd love to hear about your debugging experiences. Connect with me on [GitHub](https://github.com/davdunc).*
