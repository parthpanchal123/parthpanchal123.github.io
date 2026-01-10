---
date: "2026-01-09T22:51:59+05:30"
draft: true
title: "Write your own Thread Pool in Java 🧵"
summary: "Nahh , I'll just use ExecutorService . Okay but still hear me out :)"
cover:
  image: "/images/thread-pool-blog.webp"
tags:
  [
    "java",
    "backend",
    "thread-safety",
    "thread-pool",
    "concurrency",
    "multithreading",
  ]
---

## _Ever wondered how Java's_ `ExecutorService` works under the hood? Let's build our own thread pool and discover why it's like running a well-organized chai tapri instead of hiring a new chaiwala for every single order.

## The Problem: When Threads Go Bonkers 🎭

Imagine you're running a chai tapri. Every time a customer walks in asking for chai, you hire a new chaiwala, they make one cup, and then... they quit. Forever. Sounds ridiculous, right? But that's exactly what happens when you create a new thread for every task in your application.

```java
// The naive approach (only do this if you wanna get fired :p )
for (Task task : tasks) {
    new Thread(() -> processTask(task)).start(); // Hire, work, fire, repeat
}
```

This works fine for a few customers, but when rush hour hits (read: high concurrency), you'll have:

- 💸 **Memory overhead**: Each thread consumes ~1MB of stack space
- ⚡ **CPU goes Brrrrr**: Context switching between hundreds of threads
- 🔥 **Resource exhaustion**: Your JVM will throw a tantrum (OutOfMemoryError, anyone?)

## The Solution: A Thread Pool (Your Tapri Manager) ☕

A thread pool is like having a fixed team of chaiwalas who stay on the job. Customers (tasks) wait in a queue, and when a chaiwala (worker thread) finishes an order, they immediately pick up the next one. Efficient, predictable, and scalable - just like a well-run tapri during morning rush hour!

Let me show you how to build one from scratch.

## The Architecture: Four Heroes, One Mission 🦸

Our thread pool has four key components:

### 1\. **MiniThreadPool** - The Manager

The orchestrator that creates workers, maintains the queue, and handles the chaos.

### 2\. **PoolWorker** - The Chaiwalas

Dedicated threads that continuously grab tasks from the queue and execute them - like your tapri's chaiwalas who keep making chai one after another.

### 3\. **WorkTask** - The Order Slip

A wrapper around your actual task that adds metadata (ID, timestamps) and handles callbacks - think of it as the chai order slip with customer details.

### 4\. **BlockingQueue** - The Order Queue

A thread-safe queue where tasks wait patiently (or not so patiently) for a worker - like customers standing in line at your tapri, waiting for their turn.

## Let's Build It! 🛠️

### Step 1: The Task Queue

```java
private BlockingQueue<WorkTask> taskQueue = new ArrayBlockingQueue<>(maxQueueSize);
```

**Why** `ArrayBlockingQueue`? It's bounded (fixed capacity), which prevents memory bloat. Unlike `LinkedBlockingQueue` which can grow unbounded, or `PriorityBlockingQueue` which adds sorting overhead, `ArrayBlockingQueue` gives us predictable memory usage and simple FIFO ordering - perfect for our use case. Think of it as a tapri with a fixed number of seats - once full, new customers wait outside instead of the queue growing infinitely.

Think of this as your order queue. When it's full, new orders wait. When it's empty, workers wait. It's like a perfectly balanced ecosystem - just like a well-managed tapri where customers queue up during peak hours and chaiwalas take a breather when it's quiet.

### Step 2: The Workers

```java
public class PoolWorker extends Thread {
    private final BlockingQueue<WorkTask> taskQueue;
    private volatile boolean isBusy = false;

    @Override
    public void run() {
        while (!Thread.currentThread().isInterrupted()) {
            try {
                WorkTask task = taskQueue.take(); // Blocks if queue is empty

                if (task.isPoisonPill()) {
                    break; // Time to go home
                }

                isBusy = true;
                task.run(); // Do the actual work
                isBusy = false;
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            }
        }
    }
}
```

Each worker runs in an infinite loop (until interrupted), taking tasks from the queue. The `take()` method is **blocking** - if the queue is empty, the thread goes to sleep. No busy-waiting, no CPU waste. It's like a chaiwala who actually sits down and relaxes when there are no customers instead of frantically checking every millisecond if someone's coming.

Notice the `isPoisonPill()` check? That's our graceful shutdown mechanism - we'll dive into the **Poison Pill Pattern** in Step 4, so stay tuned!

### Step 3: Submitting Tasks

```java
public void submit(Runnable task) {
    if (isShutdown) {
        throw new IllegalStateException("We're closed!");
    }

    long taskId = taskIdGenerator.incrementAndGet();
    WorkTask workTask = new WorkTask("Task-" + taskId, task, completionCallback);
    taskQueue.put(workTask); // Blocks if queue is full
}
```

When you submit a task, it gets wrapped in a `WorkTask` (with an ID and timestamp) and added to the queue. If the queue is full, `put()` blocks until space is available. This is **backpressure** in action - your system naturally slows down when overwhelmed instead of crashing.

### Step 4: The Poison Pill Pattern 💊

How do you shut down gracefully? You can't just kill threads mid-task (that's like telling a chaiwala to stop mid-pour - messy and wasteful). Instead, we use the **Poison Pill Pattern**:

```java
public void shutDown() {
    isShutdown = true;
    // Send one poison pill per worker
    for (int i = 0; i < workers.size(); i++) {
        taskQueue.offer(WorkTask.createPoisonPill());
    }
}
```

A poison pill is a special "task" that tells workers to stop. When a worker picks it up, they finish their current task and exit gracefully. It's like telling your chaiwalas, "Bhai, yeh last order hai, iske baad aap ghar ja sakte ho" (Bro, this is the last order, after this you can go home).

**Why not just use** `Thread.interrupt()`? Good question! While `interrupt()` works, it has two major drawbacks: (1) It can interrupt a thread at any point - even during critical operations, and (2) When a thread is interrupted, all pending tasks in the queue are completely ignored. With poison pills, workers process tasks normally from the queue until they encounter the poison pill, ensuring no work is wasted. It's like letting your chaiwala finish all the cups in the queue before closing, rather than snatching the current cup mid-pour and dumping the rest. Much cleaner, much safer, and no wasted chai!

## Using It: The Fun Part 🎉

```java
MiniThreadPool pool = new MiniThreadPool(2, 5); // 2 workers, queue size 5
CountDownLatch latch = new CountDownLatch(10);

// Submit 10 tasks
for (int i = 0; i < 10; i++) {
    final int taskNum = i;
    pool.submit(() -> {
        try {
            Thread.sleep(500); // Simulate work
            System.out.println("Task " + taskNum + " done!");
        } finally {
            latch.countDown();
        }
    });
}

latch.await(); // Wait for all tasks
pool.shutDown(); // Close tapri gracefully (no chai left behind!)
```

### Output

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1767519605200/f36146a4-132c-4350-86b8-e02e64304d68.png align="center")

## Why This Matters 🎯

Building a thread pool from scratch teaches you:

1. **Thread Safety**: How `BlockingQueue`, `AtomicLong`, and `volatile` work together
2. **Concurrency Patterns**: Producer-consumer, poison pill, graceful shutdown
3. **Resource Management**: Controlling concurrency prevents system overload
4. **Performance**: Reusing threads is orders of magnitude faster than creating new ones

## The Real World 🌍

In production, you'd use Java's `ExecutorService` (which is battle-tested and optimized), but understanding the internals makes you a better developer. It's like learning to drive a manual car - you might use automatic, but you understand what's happening under the hood.

## Key Takeaways 💡

- **Thread pools reuse threads** - like keeping a team of chaiwalas instead of hiring/firing constantly (saves money and chaos!)
- **Blocking queues are magic** - they handle synchronization and waiting for you
- **Poison pills are elegant** - graceful shutdown without chaos
- **Backpressure is your friend** - when the queue is full, the system naturally slows down

## Wrapping Up 🎁

Thread pools are everywhere in modern software - web servers, database connection pools, task schedulers. Understanding them isn't just academic; it's practical knowledge that helps you write better concurrent code.

Want to see the full implementation? Check out the [GitHub repository](https://github.com/parthpanchal123/Worker-Pool) and experiment with it yourself!

---

_Happy coding! Ab chai peete hue code karo!_ ☕✨
