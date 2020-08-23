---
layout: post
title: "Multithreading in iOS"
date: 2018-03-29 08:41:11 +0600
---

# Multithreading in iOS

Hello, reader. This is really a draft textual represenation of [my talk at Kolesa Conf](https://www.youtube.com/watch?v=x99n-i8BUO8) about multithreading in iOS.

Multithreading is one the two main topics that are resource related. The other is memory management. Both are about effective usage of the computer resources, such as CPU and memory.

It is also top question asked during interviews 😉.

Let's get started. 

As for me, the topic can be divided into 4 parts.

1. Process and threads
2. Synchronization
3. GCD (Grand Central Dispatch)
4. Operation and OperationQueue

## Process and threads

> Process – instance of a computer program that is being executed by one or many threads ([wikipedia](https://en.wikipedia.org/wiki/Process_(computing)))

We talk about threads later. So process is actually a running program, an iOS application in our case. You can find the process names in Xcode by going **Product -> Attach to Process**. It contains the compiled code of the application to run, memory used, file descriptors, context and etc.

![](/assets/attach_to_process.png)

_One program can spawn multiple processes, this is how extensions work. For example, the **Notification Service Extension** is an extension that intercepts the push notifications before they are received. Since the main application process is not active when in background, a new process handles this work._

Okay, the process is clear, what about threads? As wikipedia says, it is executed by one or many threads. You probably have heard about the main thread of the applicaton and that UI related activies must run on the main thread do avoid lags and delays.

> Thread of execution – the smallest sequence of programmed instructions that can be managed independently... ([wikipedia](https://en.wikipedia.org/wiki/Thread_(computing)))

Every process has at least one (main) thread of execution. Multithreading is about multiple threads. To serve different parts of the application, multiple threads are utilized. Back then it was okay to serve a single purpose, maybe complete some calculation and the program would exit. Today, with user interactive application it is no more the case. 

The main thread if an iOS application runs the **Run Loop**, the event handler. Such actions as touch, drag, accelerometer and gyroscope are processed by the hardware and then the application is notified. If the main thread must be always available to react, then we are obliged to use other threads. Okay, nuf said, let's see what is a thread.

### pthreads (libpthread)

Of course a thread is completely an Operating System concept and the CPU does not care about which thread to execute at any moment of time. In Apple-like (read Unix-like) operating systems there is a library available for working with threads. It is called **libpthread** or simply **pthreads**.

This is how a new thread can be created, a block is passed as instructioins to be executed.

```swift
var thread: pthread_t? = nil
let result = pthread_create(&thread, nil, { _ in
    print("Hello world!")
    return nil
}, nil)
```
_reference: [pthread_create](http://man7.org/linux/man-pages/man3/pthread_create.3.html)_


Simple as that, for the sake of simplicity, 2 arguments are skipped, these are the attributes and the context.

This snippet of code created and fires the thread. It would be great if we could wait until it finishes and we continue.

```swift
pthread_join(thread, nil)
```
_reference: [pthread_join](http://man7.org/linux/man-pages/man3/pthread_join.3.html)_

Adding this line of code lets the current thread to pause and join the spawned thread until in finishes.

A thread can be canceled (stopped). 

```swift
pthread_cancel(thread)
```
_reference: [pthread_cancel](http://man7.org/linux/man-pages/man3/pthread_cancel.3.html)_

This is not very safe if there are any descriptors or connections open. They may be left open and other threads will not be able to use these resources.

This syntax is a bit cryptic and way of working with threads is neither convenient nor safe. Agree?

### NSThreads

An attempt was made to make a better interaction with threads by using object oriented approach.

Creating a thread:
```swift
let thread = Thread {
    print("Hello World!")
}

thread.start()
```


Canceling a thread: 
```swift
thread.cancel()
```

Waiting for a thread to finish:

```swift
while !thread.isFinished { }
```

In a real application all threads must be created by the developer and managed by the developer. It implies:
1. A lot of code for creating and configuration
2. A lot of code for thread management

And it is not very simple. It would great if there was an **abstraction** over threads which would **create, configure and manage threads**, maybe reusing them to avoid unnecessary creation (pool of threads) and handle them according to the task performed (priority and importance based).

## Grand Central Dispatch

Forget what was written before. Kidding. 

GCD is a mechanism for working in multithreaded environment. It is about two things:
1. Dispatch queues
2. Synchronous and asynchronous execution

### Dispatch queues

There are 2 types of queues: **serial** and **concurrent**. As the name implies serial queues execute tasks one after another. While concurrent may execute many tasks simultaneously. But what is concurrent?

> There are two terms that may be confused: parallelism and concurrency. While the former means actual simultaneous execution like two cars running, the latter means the opposite. Concurrency is achieved by  continuously switching between tasks. On a single core CPU it is the only way to run multiple programs.

These are the queues supplied by the operating system: 
- Main queue (serial)
- Global queues (concurrent, have QoS - quality of service)
    - User Interactive
    - User Initiated
    - Utility
    - Background

The main queue is for everything UI related.
The others serve as the job requires. 

**User Interactive** - for the interface related jobs, the main thread may be used insted.

**User Initiated** - for the jobs that disallow user progress, but the app remains interactive. It may be waiting for flight tickets to load white the progress bar is animating.

**Utility** - Data upload/download, text and image processing.

**Background** - Sync, backups. A user may not even know that something is done in the app.

It is crucial to select the appropriate queue. Correct choice helps us to reach smooth user experience and effective resource (CPU) utilization. If we don't succeed at this, we get this:

>gif

While something like this must be achieved, when each queue handles the work as the priority requires.

> gif

### sync & async

Synchronous execution - fire and wait until finished.

Asynchronous execution - fire and return to own thread of execution.

## Working with GCD

To run a job on the main queue synchronously:
```swift
before()
DispatchQueue.main.sync {
    inside()
}
after()
```

The order above is **before -> inside -> after**

or asynchronously: 
```swift
before()
DispatchQueue.main.async {
    inside()
}
after()
```

The order above may be either **before -> inside -> after** or **before -> after -> inside**, since we don't know the the asynchronoues block will run, it is up to the system scheduler.

To perform a job in the global queue, a similar action is done.
```swift
DispatchQueue.global(qos: .userInitiated).async {
    inside()
}
```
Here the quality of service is supplied as an argument. 

I told you before that dispatch queues are an abstraction over threads and this is how the are mapped:

> image with thread pool

A common approach working with queues is:
```swift
loader.startAnimating()
DispatchQueue.global(qos: .utility).async {
   image = processImage()

    DispatchQueue.main.async {
        loader.stopAnimating()
        imageView.image = image
    }
}
```
Image processing here is a time and resource consuming task, which is handled in another queue, utility in this specific case. To update the UI we "return" to the main queue. Xcode will warn you if you are performing UI related activities not in the main queue.

A helpful library method is **asyncAfter** which delays the execution of some block to the future. The exact time in the future is passed as an argument. It is guaranteed to execute not earlier than the deadline, but may be delayed. Not very real-time, to be honest.
```swift
DispatchQueue.main.asyncAfter(deadline: .now() + 1.0) {
    processData()
}
```

Another mechanism that GCD provides is **Dispatch Groups**. These are for a group of similar tasks that must be performed before some event.
In a classified app scenario an example would be image uploading before posting an advert.
```swift
let group = DispatchGroup()

for image in images {
    queue.async(group: group) {
        processImage(image)
    }
}

group.notify(queue: .main) {
    callback()
}
```
In the loop, multiple tasks are pushed onto the group and when they all finish the notify callback is executed. In this example on the main queue. Convenient for a batch execution.

We haven't covered the synchronization issues yet but let's see the classic [readers-writers problem](https://en.wikipedia.org/wiki/Readers–writers_problem). Is about some data structure which is used for writing and fetching data at the same time. Apparently, reading the data may be perfomed in parallel, while writing into this data structure may result in undefined behavior since two threads may be modifying the same memory area.

Dispatch queues solve this problems easily with **Dispatch barriers**.
Simply mark the block execution as a barrier task and all other tasks in the queues will be paused until a barrier task finished.

Reading data, simple: 
```swift
queue.sync {
    return self.value
}
```
Writing data, simple, too:
```swift
queue.async(flags: .barrier) {
    self.value = value
}
```

This is how it can be illustrated:

> image barrier

Well, while GCD is a solution, it turns out we don't have a proper control over the queues and the tasks. For instance, canceling a task. It may be achieved by using **DispatchWorkItem**-s which are wrapping the block. But the control is not that granular.

Let's see what operations and operation queues represent.

## Operation and OperationQueue
Operation queues are about queues. \*roll safe meme here\*

 Same main queue given. Same quality of service, but global queues are not provided and must be created by the developer. Operations and queues are designed to work together.

 **Operation queues** have many characteristics but I will mention the main only: **maxConcurrentOperationCount**. It is the number of cuncurrent tasks run by the queue. Limiting this number may set your queue as a serial. It also help utilizing the CPU correctly.

 **Operations** are simply a FSM (Finite State Machine) with 4(5) states:
- Ready
- Executing
- Canceled
- Finished

They have a quality of service and a priority. These two are also always confused. The priority of a task defines how quick this task moves within the queue to be executed. The QoS of a task defines how many resources (CPU, time, memory) are given to execute it.

I must mention that operations may be used without queues, be on their own, per se. But I haven't found an application of this approach. You can read about operations more [here](https://developer.apple.com/documentation/foundation/operation).

**Operation** as a class is abstract. It cannot be instantiated. Two implementation are given in the Foundation, BlockOperation and NSInvocationOperation (not available in Swift). 

```swift
let operation = BlockOperation {
    processData()
}

let queue = OperationQueue()
queue.addOperation(operation)
```

Workflow:

**Create an operation -> Create a queue -> Add operation to the queue**

An operation can be canceled:
```swift
operation.cancel()
```
but not always will. If it started, then this one will run until finishes. Otherwise it just won't start.

To gain more control over operation, we can subclass and make our own implementation. Overriding the **main** method is enough.
```swift
class MyOperation: Operation {
    override func main() {
        processData()
    }
}
```

This operation can be canceled during execution by checking the state of the operation: 
```swift
class MyOperation: Operation {
    override func main() {
        processData()
        if isCancelled {
            return
        }

        processData()
    }
}
```
If this was a database transaction, a rollback could be performed in the if block. A common application of cancellation is if a user opened a new view controller, the data began to load and all of a sudden the user goes back. 

Operations can be canceled individually or in a batch. The **cancelAllOperations** called on a queue will cancel every operation contained in the queue. Apart from cancellation, the queue can be suspended. Setting the property **isSuspended** on the queue disallows new operations to startm while running operation complete.

Operations can also have a completion block to run a certain task upon a completion of the operation.