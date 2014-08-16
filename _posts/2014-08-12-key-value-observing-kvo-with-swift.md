---
title: "Key Value Observing (KVO) with Swift Closures"
layout: "post"
permalink: "/2014/08/key-value-observing-kvo-with-swift.html"
uuid: "4452058501132139565"
guid: "tag:blogger.com,1999:blog-32090046.post-4452058501132139565"
date: "2014-08-12 03:18:00"
updated: "2014-08-12 13:10:28"
description: 
blogger:
    siteid: "32090046"
    postid: "4452058501132139565"
    comments: "0"
categories: [swift]
author: 
    name: "rectalogic"
    url: "undefined?rel=author"
    image: "http://img2.blogblog.com/img/b16-rounded.gif"
---

This is an Swift class to allow KVO observing using Swift closures, useable from a Swift class that does not subclass `NSObject`.

From Swift, create a `KeyValueObserver` instance with the object being observed, the key path to observe and a closure to be called. As long as this instance remains alive, observations will be reported to the closure. To remove the observer, release the `KeyValueObserver` instance (so assign it to an optional so you can assign that to `nil` to release it).

```swift
let button = UIButton()

var kvo: KeyValueObserver? = KeyValueObserver(source: button, keyPath: "selected", options: .New) {
    (kvo, change) in
    NSLog("observing %@ %@", kvo.keyPath, change)
}

button.selected = true
button.selected = false
kvo = nil
button.selected = true
```

The implementation uses a global singleton `NSObject` subclass `KVODispatcher` dispatcher as the observer. `KeyValueObserver` instance is marshalled unretained into an `UnsafeMutablePointer<KeyValueObserver>` and passed as the context to `addObserver:forKeyPath:options:context:`.

`KVODispatcher.observeValueForKeyPath()` retrieves the `KeyValueObserver` instance from the context pointer and invokes the closure. Note that the `KeyValueObserver` is not retained when it is passed as the context or when it is extracted again - `KeyValueObserver.deinit` removes the observer so `observeValueForKeyPath()` should never be called with a deallocated instance. When you have finished observing, assign your `KeyValueObserver` optional to `nil` to remove the observer.

```swift
typealias KVObserver = (kvo: KeyValueObserver, change: [NSObject : AnyObject]) -> Void

class KeyValueObserver {
    let source: NSObject
    let keyPath: String
    private let observer: KVObserver

    init(source: NSObject, keyPath: String, options: NSKeyValueObservingOptions, observer: KVObserver) {
        self.source = source
        self.keyPath = keyPath
        self.observer = observer
        source.addObserver(defaultKVODispatcher, forKeyPath: keyPath, options: options, context: self.pointer)
    }

    func __conversion() -> UnsafeMutablePointer<KeyValueObserver> {
        return pointer
    }

    private lazy var pointer: UnsafeMutablePointer<KeyValueObserver> = {
        return UnsafeMutablePointer<KeyValueObserver>(Unmanaged<KeyValueObserver>.passUnretained(self).toOpaque())
    }()

    private class func fromPointer(pointer: UnsafeMutablePointer<KeyValueObserver>) -> KeyValueObserver {
        return Unmanaged<KeyValueObserver>.fromOpaque(COpaquePointer(pointer)).takeUnretainedValue()
    }

    class func observe(pointer: UnsafeMutablePointer<KeyValueObserver>, change: [NSObject : AnyObject]) {
        let kvo = fromPointer(pointer)
        kvo.observer(kvo: kvo, change: change)
    }

    deinit {
        source.removeObserver(defaultKVODispatcher, forKeyPath: keyPath, context: self.pointer)
    }
}


class KVODispatcher : NSObject {
    override func observeValueForKeyPath(keyPath: String!, ofObject object: AnyObject!, change: [NSObject : AnyObject]!, context: UnsafeMutablePointer<()>) {
        KeyValueObserver.observe(UnsafeMutablePointer<KeyValueObserver>(context), change: change)
    }
}

private let defaultKVODispatcher = KVODispatcher()
```

A version of this as an Xcode playground is available [on github](https://github.com/rectalogic/KVOPlayground).
