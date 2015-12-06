---
layout: post
title:  "Bindings, Generics, Swift and MVVM"
date:   2014-12-10 15:17:58
categories: ios swift
permalink: bindings-generics-swift-and-mvvm
disqus: true
---

In previous post I've introduced MVVM design pattern as an improvement to MVC, but I've ended with the solution that covers only certain scenarios - immutable models and view models. To cover rest of them we need mutable view models that can propagate changes to views or view controllers they populate.

In this post I'll present simple binding mechanism by implementing observer pattern using Swift generics and closures.

Objects of basic data types and values of primitive types do not provide us a way to observe their changes. In order to do that we would have to gain control of their setters (or of that which sets them) and in there notify those who are interested. Fortunately, Swift is smart enough not to allow that. Having that level of freedom would quickly go wrong. However it is possible to create our own data types and tailor them by our desires. We could make them encapsulate basic data or primitive type. That would make modifying value of encapsulated type go through our type's interface and that's where we can do what pleases us. Let's try doing that with, say, basic String type. We'll call new type DynamicString.

```swift
class DynamicString {
  var value: String
 
  init(_ v: String) {
    value = v
  }
}
```

This is called wrapping or <a href="http://en.wikipedia.org/wiki/Object_type_(object-oriented_programming)#Boxing">boxing</a> of an object. It doesn't do much but it put us in control of encapsulated object. We can take that control with Swift feature introduced in previous post - property observers. It allows us to intercept change of property's value, like this.

```swift
class DynamicString {
  var value: String {
    didSet {
      println("did set value to \(value)")
    }
  }
  
  init(_ v: String) {
    value = v
  }
}
```

We attached some code to <code>didSet</code> property observer of <code>value</code>. Here is an example of what does it do.

```swift
let name = DynamicString("Steve")   // does not print anything
println(name.value)  // prints: Steve
name.value = "Tim"   // prints: did set value to Tim
println(name.value)  // prints: Tim
```

As you can see, assigning new value to <code>value</code> property triggers property observer which prints that value. And this is our knight in shining armour. Instead of printing new value, we will use property observer to inform interested parties, let's call them <em>listeners</em>, that change occurred. What is a listener? In our MVVM example from previous post that is a view controller. It's interested in changes in ViewModel so that it can update its views accordingly. But do we want to reference the view controller from each string object we create? I hope not. Maybe we could create a Listener protocol and have our listeners adopt it? Not gonna work - listeners will probably want to observe many other objects (properties). We need another knight - and Swift has it - it's called closure (block in Objective-C or lambda in some other languages). We can define listener to be a closure that accepts one argument of type String.

```swift
class DynamicString {
  typealias Listener = String -> Void
  var listener: Listener?

  func bind(listener: Listener?) {
    self.listener = listener
  }

  var value: String {
    didSet {
      listener?(value)
    }
  }
  
  init(_ v: String) {
    value = v
  }
}
```

We've used <code>typealias</code> command to introduce new type, <code>Listener</code>, which is a closure that accepts one String argument and returns nothing. With that type we declared a <code>listener</code> property, making it an Optional so it's not required to be set (our DynamicString object does not have to have a listener). Next, we've created a setter for listener property just to make syntax a bit nicer. Finally, we've changed property observer to call that listener closure if it is set. That's it! Let's go through the example.

```swift
let name = DynamicString("Steve")

name.bind({
  value in
  println(value)
})

name.value = "Tim"  // prints: Tim
name.value = "Groot" // prints: Groot
```

So, each time we set a new value to our dynamic string, listener is triggered and prints out that value. Notice that our binding syntax doesn't look very nice. Fortunately, Swift is full of syntactic sugars and two of them will help us here. First one says that if last argument of a function is a closure, the closure expression can be defined after parentheses of the function call it supports. Closure like that are called <a href="https://developer.apple.com/library/ios/documentation/Swift/Conceptual/Swift_Programming_Language/Closures.html#//apple_ref/doc/uid/TP40014097-CH11-XID_160">trailing closure</a>. Additionally if function has only one argument, parentheses can be omitted altogether. Second syntactic sugar is that "Swift automatically provides <a href="https://developer.apple.com/library/ios/documentation/Swift/Conceptual/Swift_Programming_Language/Closures.html">shorthand argument names</a> to inline closures, which can be used to refer to the values of the closure’s arguments by the names $0, $1, $2, and so on." If we exploit that knowledge we get this:

```swift
name.bind {
  println($0)
}
```

That's much nicer! We can implement listener closure however we want. Instead of printing new value, we could make it update the label text.

```swift
let name = DynamicString("Steve")
let nameLabel = UILabel()

name.bind {
  nameLabel.text = $0
}

println(nameLabel.text)  // prints: nil

name.value = "Tim"
println(nameLabel.text)  // prints: Tim

name.value = "Groot"
println(nameLabel.text)  // prints: Groot
```

As you can see, label text is updated whenever the value of our string is changed, but what about first time? Where is Steve? We should not forget him. If you think about it for a bit you'll notice that <code>bind</code> method only sets the listener, it does not trigger it. We can implement another method that does both things. Let's call it bindAndFire.

```swift
class DynamicString {
  ...
  func bindAndFire(listener: Listener?) {
    self.listener = listener
    listener?(value)
  }
  ...
}
```

If we modify our example to use this method instead, we'll get Steve back. In a way.

```swift
...
name.bindAndFire {
  nameLabel.text = $0
}

println(nameLabel.text)  // prints: Steve
...
```

Great, we've come a long way. We've introduced a new string type that allows us to attach a listener to itself which observes value changes and we've shown how it can be used to perform certain actions like label text updating. But String is not the only type we work with so let's continue by expanding this idea with new types. We could create similar class for Integer... Hmm, and then for Float, Double and Boolean? What about NSDate, UIView or dispatch_queue_t? That seems like a lot of pain. It is, and we would be crazy to take that approach. Instead, we will use one of the most powerful features of Swift - <a href="https://developer.apple.com/library/ios/documentation/Swift/Conceptual/Swift_Programming_Language/Generics.html">generics</a>. It enables us "to write flexible, reusable functions and types that can work with any type." If your are not familiar with generics, follow that link to learn more about them. When you're done, we'll rewrite our DynamicString type to generic type type we'll call Dynamic. It goes like this:

```swift
class Dynamic<T> {
  typealias Listener = T -> Void
  var listener: Listener?
  
  func bind(listener: Listener?) {
    self.listener = listener
  }
  
  func bindAndFire(listener: Listener?) {
    self.listener = listener
    listener?(value)
  }

  var value: T {
    didSet {
      listener?(value)
    }
  }
  
  init(_ v: T) {
    value = v
  }
}
```

We've renamed our DynamicString class to Dynamic, marked it as a generic class by appending <code>&lt;T&gt;</code> to its name and replaced all occurrences of <code>String</code> with <code>T</code>. Our Dynamic type can now wrap any other type and expand it with listener mechanism. Here are some examples.

```swift
let name = Dynamic<String>("Steve")
let alive = Dynamic<Bool>(false)
let products = Dynamic<[String]>(["Macintosh", "iPod", "iPhone"])
```

Isn't that impressive. It can get even better. Swift compiler is so powerful it can infer type from function's (in this case constructor's) arguments so it's enough to write just this:

```swift
let name = Dynamic("Steve")
let alive = Dynamic(false)
let products = Dynamic(["Macintosh", "iPod", "iPhone"])
```

Bindings work the same way. Type of the argument in listener closure would be the one specified in generic list (or inferred by compiler when list is omitted). For example:

```swift
products.bindAndFire {
  println("First product is \($0.first)")
}
```

That's our binding mechanism. It's simple, yet powerful. It can work with any type, you can bind any logic you need and you don't have to go the through pain of registering and unregistering observers (listeners). You just bind a closure. There is one limitation though - you can have only one listener. That should be enough for most of the cases and for our MVVM example, but should you need support for more than one listener try improving this idea by having an array of listeners - it is possible but it might introduce some other implications.

To conclude, let's fix MVVM example from previous post so that thumbnail can be propagated to image view. We can start by redefining view model protocol so it requires properties to be bindable, i.e. Dynamic. We can do that for all of them, just to show how it's done.

```swift
protocol ArticleViewViewModel {
  var title: Dynamic<String> { get }
  var body: Dynamic<String> { get }
  var date: Dynamic<String> { get }
  var thumbnail: Dynamic<UIImage?> { get }
}
```

Be mindful about the optional there. Wrapped type is the one that is optional, not Dynamic wrapper! Next, we continue by refactoring view model.

```swift
class ArticleViewViewModelFromArticle: ArticleViewViewModel {
  let article: Article
  let title: Dynamic<String>
  let body: Dynamic<String>
  let date: Dynamic<String>
  let thumbnail: Dynamic<UIImage?>
  
  init(_ article: Article) {
    self.article = article
    
    self.title = Dynamic(article.title)
    self.body = Dynamic(article.body)
    
    let dateFormatter = NSDateFormatter()
    dateFormatter.dateStyle = NSDateFormatterStyle.ShortStyle
    self.date = Dynamic(dateFormatter.stringFromDate(article.date))
    
    self.thumbnail = Dynamic(nil)
    
    let downloadTask = NSURLSession.sharedSession()
                                   .downloadTaskWithURL(article.thumbnail) {
      [weak self] location, response, error in
      if let data = NSData(contentsOfURL: location) {
        if let image = UIImage(data: data) {
          self?.thumbnail.value = image
        }
      }
    }
    
    downloadTask.resume()
  }
}
```

This should be straightforward, however please note few things. All properties are still constants (defined with <code>let</code>). That is important because we must not change them after we set them for the first time. Listeners get notified when the value of Dynamic changes, but not when Dynamic itself changes! That means that we have to initialize all of them in the constructor (initializer). Here is a golden rule: Dynamics that cannot be initialized with concrete value in the constructor must encapsulate optional type, like as <code>thumbnail</code> encapsulates optional <code>UIImage</code>. In cases like those, we initialize Dynamic with <code>nil</code> and update it later when concrete or new value becomes available - like when thumbnail is downloaded.

What's left is to bind all those properties in the view controller.

```swift
class ArticleViewController {
  var bodyTextView: UITextView
  var titleLabel: UILabel
  var dateLabel: UILabel
  var thumbnailImageView: UIImageView
  
  var viewModel: ArticleViewViewModel {
    didSet {
      viewModel.title.bindAndFire {
        [unowned self] in
        self.titleLabel.text = $0
      }
      
      viewModel.body.bindAndFire {
        [unowned self] in
        self.bodyTextView.text = $0
      }
      
      viewModel.date.bindAndFire {
        [unowned self] in
        self.dateLabel.text = $0
      }
      
      viewModel.thumbnail.bindAndFire {
        [unowned self] in
        self.thumbnailImageView.image = $0
      }
    }
  }
}
```

And that's it! Our view will reflect any change done to view model. Just be careful not cause retain cycles with closures. Always use <code>unowned</code> or <code>weak</code> <code>self</code> within the closure. We're ok with unowned here because view controller is owner of view model so view model will not outlive view controller.