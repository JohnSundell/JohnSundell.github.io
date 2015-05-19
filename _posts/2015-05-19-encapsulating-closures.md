---
layout: post
title:  Code encapsulation using closures
date:   2015-05-19
---

Swift's closures are awesome - and super handy when it comes to encapsulating code - especially in `init` methods. You usually want to keep your `init` methods short & sweet, and in Objective-C many developers accomplish this by delegating setup to separate methods. Like this pattern:

```
MyClass {
    init() {
        self.setupView()
        self.setupObservers()
    }
    
    func setupView() {
        ...
    }
    
    func setupObservers() {
        ...
    }
}
```

This doesn't work very well in Swift, considering that you want to use constants (`let`) as much as possible, and you cannot call methods on an object that hasn't been fully initialized yet. So, I've started using closures for this. Say you have a closure called `Calculate`, like this one:

```
func Calculate<T>(closure: () -> T) -> T {
    return closure()
}
```

This allows you to encapsulate your code, without creating tons of utility methods, and still keeping all your setup code in your `init` method, like this:

```
MyClass {
    let view: UIView
    let observers: [Observer]

    init() {
        self.view = Calculate {
            var frame = CGRect()
            frame.size.origin = 100
            frame.size.width = 300
            
            return UIView(frame: frame)
        }
        
        self.observers = Calculate {
            var observers = [Observer]()
            
            observers.append(OrdinaryObserver())
            
            if someCondition {
                observers.append(SpecialObserver())
            }
            
            return observers
        }
    }
}
```

