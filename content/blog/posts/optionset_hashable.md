---
title: "Hashable OptionSet in Swift"
date: 2018-08-29T11:00:08-07:00
Description: "Implementing a Hashable OptionSet in Swift for game achievements"
Tags: ["swift", "swift 4"]
Categories: ["code"]
draft: false

---

This post explains how to create an [OptionSet](https://developer.apple.com/documentation/swift/optionset) in Swift that also implements the [Hashable](https://developer.apple.com/documentation/swift/hashable) protocol. That way it can be used in a [Dictionary](https://developer.apple.com/documentation/swift/dictionary), for example. Even though it's super simple, I couldn't find a documented example online so I thought I'd put one together.

# Motivation

I've been recently working on a game using Apple's SpriteKit and reached a point where I needed to implement and track in-game achievements. [A user on reddit](https://developer.apple.com/documentation/swift/dictionary) suggested *using an OptionSet to store which achievements have been unlocked and a Dictionary as an "achievements encyclopedia" of sorts*, which turned out to be a great suggestion! In order to make that work, the OptionSet also had to implement the Hashable protocol.

Why it made sense for my use case:

* OptionSets make bitwise operations relatively painless, so headaches are avoided there

* I'm not planning on a ton of achievements (the limit is the number of bits in an Int)

* Storing an integer (the OptionSet's rawValue) in UserDefaults is really easy, and it's storage efficient.


# Hashable Protocol Requirements

* A _hashValue_ integer is required as a way to uniquely profile (ideally) the contents of a hash

* An `==` function to compare hashes.


# Final Code
</br>
<script src="https://gist.github.com/nbw/5d4b2e73d76738afcbf3de4dd1b2a98e.js"></script>

This implementation uses the OptionSet's _rawValue_ as a unique indicator. Since _hashValue_ defers to _rawValue_, it's being used in the `==` function. This way we're consolidating responsibility to _hashValue_. But alternatively you could:

```
return lhs.rawValue == rhs.rawValue
```

# Using the OptionSet in a Dictionary
</br>
<script src="https://gist.github.com/nbw/c3dd4ecc3bb92666e821c3199db35e87.js"></script>


# Resources

* [Joyce Matos' breakdown of Hashable](https://medium.com/@JoyceMatos/hashable-protocols-in-swift-baf0cabeaebd)
