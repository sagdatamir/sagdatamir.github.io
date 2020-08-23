---
layout: post
title: "About KVO (Key-Value Observing)"
date: 2018-05-04 21:00:08 +0600
---

## KVO. Key-Value Observing 

* must be KVC-compliant (key-value coding)

### What is KVC?

> *Key-value coding is a mechanism for accessing an object’s properties indirectly, using strings to identify properties, rather than through invocation of an accessor method or accessing them directly through instance variables.*

```objc
[self setValue:@"myValue" forKey:@"firstName"];
[self setValue:@"This is a text" forKeyPath:@"someObject.someProperty.text"];

[self valueForKey:@"firstname"]
```

> *The default implementation relies on the accessor methods normally implemented by objects (or to access instance variables directly if need be).*

So, this means that the getter/setter are accessed when using the KVC mechanism. All NSObject descendants comply to the NSKeyValueCoding protocol.

If we access an unexisting value at key/keypath, **NSUnknownKeyException** is the runtime error thrown then. By the way, it is common error thrown when the outlet or action from the storyboard in not connected to a property/method.

To avoid this error caused by a possible typo, I consider using the builtin method **NSStringFromSelector(@selector(selector))** to be a good workaround.	

### KVO

To implement KVO, these must be followed:

1. The observable class must be KVO compliant
   * KVC compliant
   * must send notifications about changes manually or automatically
2. The observing class must be set as an observer.
3. The observing class must implement *observeValueForKeyPath:ofObject:change:context:* method.

Making one class observant of another is made by calling *addObserver:forKeyPath:options:context:* method on the observable class. 

In the observing method we can check the sender object and the keypath. The change object has a such structure: 

```
{
    kind = 1;
    new = Michael;
    old = George;
}

```

we can access the values simply by access dictionary keys

### Introducing context.

To correctly identify the object which has sent the notification about the change we can provide the context to the methods. The context must be global and exist throughout the code exectution. Creating a static global variable may be a solution.

```objc
static void* context1 = &context1;
```

Yes, this is a pointer pointing to itself, this is recommended as of [this tutorial](https://www.appcoda.com/understanding-key-value-observing-coding/). Well, I personally consider this to be an efficient approach. Having a property may be a good idea, too.

### Unobserving a class

It is recommended to unsubscribe from the observable classes. This may be done in the *viewWillDisappear:* method.

### Changing the automatic notifications

This method must be overriden in the class to turn the automatic notifications off.

```objc
+ (BOOL)automaticallyNotifiesObserversForKey:(NSString *)key {
    if ([key isEqualToString:@"name"]) {
        return NO;
    }
    else{
        return [super automaticallyNotifiesObserversForKey:key];
    }
}
```

### Sending manual notifications

This must be done in order to send a notification about changes to a property.

```objc
[self.child1 willChangeValueForKey:@"name"];
self.child1.name = @"Michael";
[self.child1 didChangeValueForKey:@"name"];
```

### Working with arrays

Arrays are not KVC compliant. 

The methods must be overriden and called to make the array KVO compliant. Assume that the array name is **siblings**.

```objc
-(NSUInteger)countOfSiblings;

-(id)objectInSiblingsAtIndex:(NSUInteger)index;

-(void)insertObject:(NSString *)object inSiblingsAtIndex:(NSUInteger)index;

-(void)removeObjectFromSiblingsAtIndex:(NSUInteger)index;
```

The infixes are automatically suggested by Xcode.

A solution to avoid creating a code for each array property is to create a custom class with all the above methods implemeted. This class therefore must accepts an array as a property and set in the initializer. They suggest the KVCMutableArray name for this class.

Basically, AFAIK it is used in the reactive MVVM frameworks, such as Reactive Cocoa and RxSwift. It can be implemented without using frameworks therefore.

That was a small introduction to the KVO and KVC in Objective-C. Thanks for reading :)