---
layout: post
title: "weakSelf, strongSelf"
date: 2018-04-24 18:13:57 +0600
---

## weakSelf, strongSelf

This post assumes you have basic understanding of memory management in Objective-C and know what are retain (reference) cycles.

The most common case of getting retain cycle in my experience is working with blocks (aka callbacks). We normally have to do some action on the caller object after the method calls the completion block. Moreover, it is generally **self**. It must be noted that whatever is passed to a block is captured by it, i.e. it retains the object. 

In regular development we pass **self** to a block. Whenever **self** is released by other objects which retained it, the block still holds a retain count. What it means is that the actual object cannot be released until the block finishes. In case we don't have a strong reference to a block this is not critical, but if we do – it causes a retain cycle.

**Apple** recommends developers to pass a weak reference to a block, such as in the example below.

```objc
__weak typeof(self) weakSelf = self;
self methodWithCompletion:^(void){
    [weakSelf updateSomething];
}
```

The block here retains the weakSelf object, not **self**. This is not dangerous at all since passing a message to a nil object results in nothing (thank you, objc creators!).

However, there might non-trivial cases, as mentioned in [Transitioning to ARC Release Notes](https://developer.apple.com/library/content/releasenotes/ObjectiveC/RN-TransitioningToARC/Introduction/Introduction.html) (section beginning with *"For non-trivial cycles…"*), it is possible that there are multiple calls to weakSelf. It is recommended to obtain a new strong reference to a weak reference, such as:

```objc
__weak typeof(self) weakSelf = self;
self methodWithCompletion:^(void){
    __strong typeof(weakSelf)strongSelf = weakSelf;
    [strongSelf updateSomething];
   	[strongSelf updateSomethingAgain];
}
```

I put the qualifier in the beginning to provide better readability. As soon as the block is accessed it creates a strong reference to **self**, this guarantees that the object is not released until the block returns.

Imagine that there are blocks as properties and we are willing to call these blocks. Revisit the example above and replace a message pass with a block call.

```objc
__weak typeof(self) weakSelf = self;
self methodWithCompletion:^(void){
    __strong typeof(weakSelf)strongSelf = weakSelf;
    strongSelf.updateSomething();
}
```

Well, this looks fine. But if you call a nil block, you'll get a **EXC_BAD_ACCESS** error and the program will certainly crash. Under the hood it is a segmentation fault. I think that we must check the block for nil value before executing it, let's modify the code above.

```objc
__weak typeof(self) weakSelf = self;
self methodWithCompletion:^(void){
    __strong typeof(weakSelf)strongSelf = weakSelf;
    if (strongSelf.updateSomething) {
    	strongSelf.updateSomething();
    }
}
```

This fix prevents our app from crash and this check is essential before calling any block if it is assumed that it may be nil. However, if the block is a non-null parameter/variable, in my opinon fail-fast is preferred and the app must crash here. This will inform the developer with the error he may have produced.

But what if the problem is not that the block **updateSomething** is nil, but the **strongSelf**? This will cause the same error, since nil block is returned and called. I have come to a solution here: check the strongSelf after assigning to a weakSelf. At the moment of block execution it is possible that it is already nil. Here is a quick fix:

```objc
__weak typeof(self) weakSelf = self;
self methodWithCompletion:^(void){
    __strong typeof(weakSelf)strongSelf = weakSelf;
    if (!strongSelf) {
        return;
    }
    
    if (strongSelf.updateSomething) {
    	strongSelf.updateSomething();
    }
}
```

This small if statement prevents the future checks and code execution if strongSelf is initilized with nil.

So this was a tiny solution to a common problem causing retain cycles and program failures.

Thanks for reading this post. The comment section is disabled for now, but feel free to reach me by the contacts provided on the About page.