---
layout: post
title:  "Solving the iOS Binding Problem with Swift"
date:   2015-02-19 15:17:58
categories: ios swift
permalink: solving-the-ios-binding-problem-with-swift
disqus: true
---

<em>This text was originally published at <a href="http://five.agency/solving-the-binding-problem-with-swift/">http://five.agency/solving-the-binding-problem-with-swift/</a>. Check out the original for some nice illustrations! </em>

In a fight for great software architecture, our hands are often tied by the shortcomings of frameworks or languages we work with. Simple tasks like downloading a photo and presenting it in a view, updating a view when underlying data changes, or propagating user input to a model tend to introduce unnecessary complexity to our code.

Problem is that we're so used to thinking in terms of value assignments that we try to apply same approach to situations that require something different. Assignment is a one-time operation. It's low-level. It's good for static values. Assignment is, however, terrible for dynamic data. In order to accommodate prices that change, followers that rise or faces that get prettier we need something of a higher level. We need to think in terms of bindings.

In this post we'll present a solution that makes binding of dynamic data to user interfaces as simple as it gets, giving you an opportunity to focus on what needs to be done, rather than on how to do it.
<h2>Problem analysis</h2>
Let's explore a simple example of dynamic data. Say that we're developing some kind of a discussion thread feature. We'll consider a one-thread scenario. Thread will have a name and a list of posts. Each post will be composed of a body and number of likes. In the real world, models would have more properties like user reference or a date, but we'll ignore those as they are not important for our problem analysis.

```swift
class Thread {
  var name: String
  var posts: [Post]
}

class Post {
  var body: String
  var likeCount: Int
}
```

We need a view that will display the thread. Let's define a view with a name label and a posts table view composed of a post cells.

```swift
class ThreadView {
  var nameLabel: UILabel
  var postsTableView: UITableView
}

class PostCell {
  var bodyTextView: UITextView
  var likeCountLabel: UILabel
}
```

What's left is coupling all that together. Say we're doing it from a view controller.

```swift
class ThreadViewController: UIViewController, UITableViewDataSource {
  var thread: Thread
  var threadView: ThreadView

  override func viewDidLoad() {
    super.viewDidLoad()
    threadView.nameLabel.text = thread.text
    threadView.postsTableView.dataSource = self
  }

  func tableView(tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
    return thread.count()
  }

  func tableView(tableView: UITableView, cellForRowAtIndexPath indexPath: NSIndexPath) -> UITableViewCell {
    let cell = tableView.dequeueReusableCellWithIdentifier("PostCell") as PostCell
    let post = thread.posts[indexPath.row]

    cell.bodyTextView.text = post.body
    cell.likeCountLabel.text = "\(post.likeCount)"

    return cell
  }
}
```

Simple enough. Sure, but there is problem. Users can write posts. Thread name might be updated. Some of the currently visible posts might even get liked. What will happen to our view? Exactly - nothing. Data from the model is assigned once. Any subsequent change to it will not be reflected in the view. Your likes won't count. We don't want that to happen, do we?
<h2>OK, but what are our options?</h2>
There are a couple of ways to fix that. The simplest and probably dumbest solution would be to <strong>expose methods on the view controller that update its state</strong>, then give the model the reference to the view controller and have it call exposed methods whenever change occurs. Oh, and also to say goodbye to any future reusability of those components, to simple object graphs, to single responsibility principle and other “unimportant” principles.

A better solution would be to have the model <strong>define a protocol that parties interested in observing its changes would implement</strong>. In that case, thread view controller would implement the protocol and register itself to the model as an observer (a delegate). This approach makes components more reusable but it's cumbersome to implement, especially if you have many models. When you think about it, it's actually same as the first approach, just wrapped up in nice colors.

Another solution might be to <strong>use Key-Value-Observing mechanism to observe data changes and update the view accordingly</strong>. While this approach keeps architecture simple, it's quite hard to use and, due to its nature, unsafe in some situations. You need to be vigilant while adding and removing observers. You also need to manually downcast objects, as KVO is not type-safe. Additionally, you can't do "fine grain" observing of an array, so you'd have to reload whole table regardless of the kind of array update.

Of course, you could do something totally different. For example, you could <strong>introduce a Functional reactive paradigm to your architecture with ReactiveCocoa framework and use streams and signals to solve the issue</strong>. While it's a good solution, it introduces a whole new paradigm as well as the complexity you might not need.

What we would like is something as simple as static assignment, but without the burden of additional protocols, complexities or limitations.
<h2>A Swift solution</h2>
In order to keep the view up to date with the underlying data, we must somehow observe changes in the data and propagate them to the view. There is no way to have some third party observe the changes of a variable, but if you give it some thought, same thing can be achieved by wrapping a variable in an object and updating its value through that object. Now we have a way to intercept changes and notify interested parties of what happened.

In order to do that, we'll use a neat Swift feature called <em>property observers</em>. Additionally, we'll use <em>generics</em> to make our wrapper compatible with any type. Let's call it Dynamic, as it encapsulates dynamic data.

```swift
class Dynamic<T> {
  var value: T {
    didSet {
      // Inform interested parties
    }
  }

  init(_ v: T) {
    value = v
  }
}
```

That should be straightforward - a class that encapsulates a value of generic type T. Setting of a value is intercepted by <em>didSet</em> property observer. What's left if to inform interested parties that change occurred, but before we can do that, we need to define what those parties actually are. Time for another round of thinking.

What's on the other end of our problem? A view. All kinds of views. Sometimes even something other than a view, maybe another model or an action. It makes sense to think of that side of the problem as a task that needs to be done - view updated, model changed or an action performed. Tasks can be defined by a closure, and that's how we'll define our task. We'll call that task closure Listener and define it as follows:
```swifttypealias Listener = T -> Void```
In other words, Listener is a closure that accepts a value of generic type T and performs some task with received value. It doesn't return anything.

Next, we need to somehow couple our Dynamic and our Listener, so that a Listener gets called whenever a value of a Dynamic changes. We could have a property on the Dynamic that retains a Listener, but that would not be a good idea because we don't want tasks to be owned by the data. We want tasks to be alive when they are needed (for example while the view is alive) and we want them out of our way when we don't need them. For that reason, we'll wrap them in an object.

We are aiming for a type that can encapsulate a Listener closure and that can be bound to the Dynamic so that data changes can trigger closure calls. Let's call it Bond as it creates a bond between a Dynamic and a Listener. Here is the definition:

```swift
class Bond<T> {
  var listener: Listener

  init(_ listener: Listener) {
    self.listener = listener
  }

  func bind(dynamic: Dynamic<T>) {
    // Bind to given dynamic
  }
}
```

Nothing too hard. To wrap things up, let's implement actual binding support. For that, we'll need to extend the Dynamic with an array of Bonds and implement <em>didSet</em> property observer so it iterates over that array and calls wrapped closures.

```swift
class Dynamic<T> {
  var value: T {
    didSet {
      for bondBox in bonds {
        bondBox.bond?.listener?(value)
      }
    }
  }

  var bonds: [BondBox<T>] = []
  
  init(_ v: T) {
    value = v
  }
}

class BondBox<T> {
  weak var bond: Bond<T>?
  init(_ b: Bond<T>) { bond = b }
}
```

Notice how we've wrapped array elements in a helper class named BondBox. That's because we don't want to strongly reference bonded Bonds. Their life cycle is their own business.

Actual binding is done by appending Bonds to <em>bonds</em> array. We'll do that in <em>bind</em> method of class Bond.

```swift
class Bond<T> {
  var listener: Listener

  init(_ listener: Listener) {
    self.listener = listener
  }

  func bind(dynamic: Dynamic<T>) {
    dynamic.bonds.append(BondBox(self))
  }
}
```

That's it - we have our binding mechanism. We can use our thread name property as an example. For start, we'll make it dynamic String instead of just String, like this:

```swift
var name: Dynamic<String>
```

Setting of a value is done through value property:

```swift
name.value = "Steve"
```

How do we bind it to <em>nameLabel</em>? First we create a Bond with a closure that updates the label.

```swift
let nameBond = Bond() { [unowned self] name in
  self.nameLabel.text = name
}
```

Then we bind it:

```swift
nameBond.bind(name)
```

Mission accomplished. Whenever a value of name property changes, the text of <em>nameLabel</em> will also change to newly set name.

<h2>Brushing it up</h2>
Although presented solution solves the problem of complex architecture and is type-safe, it's still inconvenient to use and potentially unsafe. We have to create Bonds, retain them, write closures and worry about strongly referencing self.

If we think a bit more about it, it becomes obvious that the views themselves should provide Bonds to which we could attach Dynamics. For example, task of updating the label doesn't change, it's always: "update the text property to new value". Doing that manually is bad, but fortunately, Swift allows us to extend existing types with new functionality.

Let's explore how that would work on a label. We'll extend it, so it provides another property - a Bond object:

```swift
private var handle: UInt8 = 0;

extension UILabel {
  var textBond: Bond<String> {
    if let b: AnyObject = objc_getAssociatedObject(self, &handle) {
      return b as Bond<String>
    } else {
      let b = Bond<String>() { [unowned self] v in self.text = v }
      objc_setAssociatedObject(self, &handle, b, objc_AssociationPolicy(OBJC_ASSOCIATION_RETAIN_NONATOMIC))
      return b
    }
  }
}
```

Looks somewhat ugly, but it's actually very simple. As UIKit is an Objective-C framework, we can use <em>associated objects</em> mechanism to associate arbitrary object to a UILabel. In this case, we're associating a Bond object with a closure that updates label's text property. We can do similar thing for other types of views like image views, text views, buttons, sliders, etc.

Let's see how that simplifies our bindings. Instead of creating and retaining a Bond object, writing a closure and dealing with self, we can do this:

```swift
nameLabel.textBond.bind(name)
```

Much easier and safer, but there is still room left for an improvement. We could define a custom operator that replaces that bind method.

```swift
infix operator ->> { associativity left precedence 105 }

public func ->> <T>(left: Dynamic<T>, right: Bond<T>) {
  right.bind(left)
}
```

With that, we can bind our data to view like this:

```swift
name ->> nameLabel.textBond
```

It can get even better. Let's define a common protocol for types that can be "bonded to" like this:

```swift
public protocol Bondable {
  typealias BondType
  var designatedBond: Bond<BondType> { get }
}
```

It defines the <em>designatedBond</em> property. In case of UILabel, the designated bond would be <em>textBond, </em>in case of an image view it might be <em>imageBond</em>. For example:

```swift
extension UILabel: Bondable {
  var designatedBond: Bond {
    return self.textBond
  }
}
```

Reason we want it defined through protocol is that we want to perform certain actions on objects that adhere to it without caring of what concrete type object really is, which could be UILabel, UIImageView, UISlider or something else. Why? Because of this:

```swift
public func ->> <T, U: Bondable where U.BondType == T>(left: Dynamic<T>, right: U) {
  left ->> right.designatedBond
}
```

It overloads the ->> operator so it can bind a Dynamic to an object that adheres to Bondable protocol (given that generic types match - you cannot bind an image to a label).

With that, our binding is reduced to something as simple as:
```swift
name ->> nameLabel
```

Awesome.

<h2>Handling mismatching types</h2>

Let's consider <em>likeCount</em> property. It's a number, so we'll redefine it as follows:

```swift
let likeCount: Dynamic<Int>
```

How do we bind it to a label? If you're thinking about

```swift
likeCount ->> likeCountLabel
```

you're mistaken. Problem is that <em>likeCount</em> holds an Int value, but label expects a String value. That line would not even compile, which is a good thing as it means that our solution is type-safe. We need to somehow add support for type transformations. In functional paradigm that's called mapping. As Swift partially leans toward the function paradigm, let's see how a map function that operates on a Dynamics would look like.

```swift
func map<T, U>(dynamic: Dynamic<T>, f: T -> U) -> Dynamic<U>
```

For the sake of length of this post, let's ignore implementation details. You can look it up on GitHub should you be interested. You will find the link at the bottom.

What map does is converts Dynamic of some type T to a Dynamic of some type U. It does that by applying given closure that converts value from type T to type U. With that, we can solve out like count problem.

```swift
map(likeCount) { "\($0)" } ->> likeCountLabel
```

It's also possible extend Dynamic class with map function and do binding like this:

```swift
likeCount.map { "\($0)" } ->> likeCountLabel
```

With all that, we can now rewrite our initial example with new approach. We need to convert static properties of the model to dynamic ones

```swift
class Thread {
  let name: Dynamic<String>
  let posts: [Post]
}

class Post {
  let body: Dynamic<String>
  let likeCount: Dynamic<Int>
}
```

and update the view controller to do binding instead of assignment

```swift
class ThreadViewController: UIViewController, UITableViewDataSource {
  var thread: Thread
  var threadView: ThreadView

  override func viewDidLoad() {
    super.viewDidLoad()
    thread.text ->> threadView.nameLabel.text
    threadView.postsTableView.dataSource = self
  }

  func tableView(tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
    return thread.count()
  }

  func tableView(tableView: UITableView, cellForRowAtIndexPath indexPath: NSIndexPath) -> UITableViewCell {
    let cell = tableView.dequeueReusableCellWithIdentifier("PostCell") as PostCell
    let post = thread.posts[indexPath.row]

    post.body ->> cell.bodyTextView.text
    post.likeCount.map { "\($0)" } ->> cell.likeCountLabel.text

    return cell
  }
}
```

Looks almost identical to initial version, but with significant difference in that the data is now bonded to user interface and any change to it will be automatically reflected in views.

There is one problem though - it works great for scalar types, but what about arrays? How do we update the table view when new posts arrive?

<h2>Arrays are special</h2>
When dealing with arrays, we are usually not interested in the change of the array object as whole, rather in changes that occurred within the array itself like insertions, deletions or updates.

Bond object allows us to observe only value changes, and value is in this case an array as whole. In order to observe fine-grain changes, we need a special kind of Bond. For example a Bond defined in following way:

```swift
class ArrayBond<T>: Bond<Array<T>> {
  var insertListener: (([Int]) -> Void)?
  var removeListener: (([Int], [T]) -> Void)?
  var updateListener: (([Int]) -> Void)?
}
```

It would allow us to register different listeners for different events. Each listener is a closure that accepts an array of indices of elements that have been changed in bonded Dynamic. Removal listener also receives elements that are removed.

Our Dynamic doesn't know when and how to call these new listeners, so we would need a special kind of Dynamic. One with additional methods for array manipulation. We would need something that would work as following example:

```swift
let posts = DynamicArray<Post>()
let myBond = ArrayBond<Post>()

myBond.insertListener = { indices in
  println("Inserted posts at indices \(indices)")
}

myBond.updateListener = { indices in
  println("Updated posts at indices \(indices)")
}

posts ->> myBond

posts.insert(Post(...), atIndex: 0)
// prints: Inserted posts at indices [0]

posts[4] = Post(...)
// prints: Updated posts at indices [4]
```

What's interesting with that is that it would allow us to extend a table view with a Bond that acts as its data source. The Bond would accept Dynamics of UITableViewCell type and we could use map function to convert posts to cells, like this:

```swift
class Thread {
  let name: Dynamic<String>
  let posts: DynamicArray<Post>
}

class Post {
  let body: Dynamic<String>
  let likeCount: Dynamic<Int>
}

class ThreadViewController: UIViewController, UITableViewDataSource {
  var thread: Thread
  var threadView: ThreadView

  override func viewDidLoad() {
    super.viewDidLoad()

    thread.text ->> threadView.nameLabel.text
    thread.posts.map { [unowned self] (post: Post) -> PostCell in
      let cell = self.tableView.dequeueReusableCellWithIdentifier("PostCell") as PostCell

      post.body ->> cell.bodyTextView.text
      post.likeCount.map { "\($0)" } ->> cell.likeCountLabel.text
      
      return cell
    } ->> self.tableView
  }
}
```

It would allow us to populate and bind whole screen with few simple binding operators. Going through the implementation details of how it could be achieved is beyond this post, but concepts are very similar to ones presented for scalar values.

<h2>Conclusion</h2>
Presented solution boils binding problem down to just one operator. It's simple, powerful, type-safe and multi-paradigm - just like Swift. It allows you to think differently when coupling data with the view and makes the whole process smooth.

You can find implementation of presented concepts on GitHub in a framework named Bond. It includes both implementation of scalar Dynamic and Bond classes, as well as implementation of vector DynamicArray and ArrayBond classes. You can find it at <a href="https://github.com/SwiftBond/Bond">https://github.com/SwiftBond/Bond</a>.