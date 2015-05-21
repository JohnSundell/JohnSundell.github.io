---
layout: post
title:  Iterating through enum members
date:   2015-05-21
---

Sometimes you want to perform a sequence of operations for each member of an `enum`. Say you have the following:

```
enum Profession: Int {
    case Engineer
    case Designer
    case Tester
}
```

Now, say you want to list all the professions available; loop through each member and turn it into a string, like this:

```
Engineer
Designer
Tester
```

What you *could* do, is add a static method to `Profession` that returns all its members, like this:

```
enum Profession: Int {
    case Engineer
    case Designer
    case Tester
    
    static fun allProfessions() -> [Profession] {
        return [.Engineer, .Designer, .Tester]
    }
}
```

This, while simple, has the big downside of being error-prone each time a new member is added. There's no way to enforce at compile time that `allProfessions()` actually returns all possible values.


### Introducing EnumIterator


I chose to solve this by creating a class - `EnumIterator` that loops through all the members of en enum that conforms to `IterableEnum` by constantly incrementing an integer, and stopping whenever that integer cannot be converted into a member of the `enum` in question. Like this:

```
protocol IterableEnum {
    var rawValue: Int { get }
    init?(rawValue: Int)
}

class EnumIterator<T: IterableEnum> {
    class func iterate(forEachCase: T -> Void) {
        var currentRawValue = 0
        
        while let enumCase = T(rawValue: currentRawValue) {
            forEachCase(enumCase)
            currentRawValue++
        }
    }
}
```
This is nice, because it doesn't require the `enum` that you want to iterate through to implement any additional methods, it simply has to be an `Int` type enum.

Let's give it a spin:

```
EnumIterator<Profession>.iterate() {
    switch $0 {
        case .Engineer:
            println("Engineer")
        case .Designer:
            println("Designer")
        case .Tester:
            println("Tester")
    }
}
```

And we get the result we were looking for!


### But, what happens if the Enum is not linear?

One problem with the current implementation of `EnumIterator` is that it will stop mid-way if the `enum` is not exactly linear. Say we add the following to `Profession`:

```
enum Profession: Int, IterableEnum {
    case Engineer
    case Designer
    case Tester
    case TeamLeader = 9
}
```

Now, we still get the same output as before, that's because our `EnumIterator` will attempt to initialize `Profession` with the raw integer value of `3`, and fail - because our 4th member has a raw value of `9` - and thus stop. Not cool.

But, we can fix this! Let's break out the sequencing (i++) logic from the `iterate()` method, and add a new method that lets our API user override the default sequencing:


```
class EnumIterator<T: IterableEnum> {
    class func iterate(forEachCase: T -> Void) {
        self.iterateWithSequenceOverride({ Int in
            return nil
        }, forEachCase: forEachCase)
    }
    
    class func iterateWithSequenceOverride(sequenceOverride: Int -> T?, forEachCase: T -> Void) {
        var currentRawValue = 0
        
        while true {
            if let enumCase = sequenceOverride(currentRawValue) {
                forEachCase(enumCase)
                currentRawValue = enumCase.rawValue + 1
                continue
            } else if let enumCase = T(rawValue: currentRawValue) {
                forEachCase(enumCase)
                currentRawValue++
                continue
            }
            
            return
        }
    }
}
```

Our API user can now choose to return a custom value `T` for a given index. Let's implement it for `Profession`:

```
enum Profession: Int, IterableEnum {
    case Engineer
    case Designer
    case Tester
    case TeamLeader = 9
    
    static func iterationSequenceOverride() -> (Int -> Profession?) {
        return {
            if $0 == 3 {
                return .TeamLeader
            }
            
            return nil
        }
    }
}
```

Now, we will get the correct expected output by calling:

```
EnumIterator<Profession>.iterateWithSequenceOverride(Profession.iterationSequenceOverride()) {
    switch $0 {
    case .Engineer:
        println("Engineer")
    case .Designer:
        println("Designer")
    case .Tester:
        println("Tester")
    case .TeamLeader:
        println("TeamLeader")
    }
}
```

That's it! We can now iterate through both linear and non-linear `Int` based `enums` with ease!

**Documented, final implementation:**
[github.com/JohnSundell/EnumIterator](https://github.com/JohnSundell/EnumIterator)