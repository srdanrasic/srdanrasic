---
layout: post
title:  "From MVC to MVVM in Swift"
date:   2014-12-07 15:17:58
categories: ios swift
permalink: from-mvc-to-mvvm-in-swift
disqus: true
---


For the past year and a half I've been working on a project that's grown from a simple news reading phone app to a full-blown virtual newspaper for both phones and tablets. Following Apple's <a href="https://developer.apple.com/library/ios/documentation/General/Conceptual/DevPedia-CocoaCore/MVC.html">advice</a> and sticking with Model-View-Controller (MVC) design pattern seemed like a good idea at first, but as the app continued to grow, logic that drove some parts of it started to became complex to the point where making changes was accompanied with the feeling of unease and fixing issues in one part of the code with the fear of causing new bugs in some other parts. It would not be fair to blame only MVC for that - of course that some problems are caused by bad programming, lack of experience and the pressure of hard deadlines - but MVC is not well-wisher either. Thinking only in the terms of model, view and controller, one can miss some useful possibilities of how responsibilities can be further separated and dependency graph simplified. In following paragraphs I'll present a situation where MVC can be improved by expanding it to MVVM (I'll explain what that is later).

Note that this post is not an advocate of MVVM nor does it present MVVM as an alternative to MVC, rather it shows a way to improve MVC in certain scenarios.

To avoid any confusion let me define what I mean by 'view' in this post. View would be a screen or part of it that presents a set of data. For example, a collection view cell that's made up of an image view and a title label. Or a story screen made up of a title label, a date label and a body text view. In other words, by the view I don't mean UIView.

Say we need a view (screen) that displays a news article. It should show article body, its title and publication date. As views in UIKit are tightly coupled with controllers, hence the name <em>view controller</em>, and since we're not interested in view's layout let's start by defining our view as a view controller.

```swift
class ArticleViewController: UIViewController {
  var bodyTextView: UITextView
  var titleLabel: UILabel
  var dateLabel: UILabel
}
```

Next, we need some data to populate the view. Let's define article model as a class with few properties. You can ignore thumbnail and saved properties for now - we'll use those later.

```swift
class Article {
  var title: String
  var body: String
  var date: NSDate
  var thumbnail: NSURL
  var saved: Bool
}
```

What's left is to populate the view, i.e. body text view and title and date label, with the model. In MVC architecture that's a job for the controller so let's do it. It's quite simple.

```swift
class ArticleViewController: UIViewController {
  var bodyTextView: UITextView
  var titleLabel: UILabel
  var dateLabel: UILabel

  var article: Article {
    didSet {
      titleLabel.text = article.title
      bodyTextView.text = article.body

      let dateFormatter = NSDateFormatter()
      dateFormatter.dateStyle = NSDateFormatterStyle.ShortStyle
      dateLabel.text = dateFormatter.stringFromDate(article.date)
    }
  }
}
```

We've defined a property for the model object, <em>article</em>, and used a neat Swift feature - <a href="https://developer.apple.com/library/ios/documentation/swift/conceptual/Swift_Programming_Language/Properties.html#//apple_ref/doc/uid/TP40014097-CH14-XID_390">Property observer</a> - to execute code whenever the property is set. (Be careful: property observers are not called when setting property from the constructor!). Whenever someone sets the property, we'll populate the view with newly set Article object. Notice how we have to process the model to get proper data. Date label wants string on its input, but our model has NSDate object.

Now, consider a situation where you have a fully implemented article view controller like the one we've just defined, and the requirement arises that you need to support some other kind of the article model, let's say, one that's defined as a dictionary. (This is just an example to show the point. Having model as dictionary is bad practice.)

```js
{
  "title": "<string>",
  "body": "<string>",
  "date": "<json_date>",
  "thumbnail": "<url>"
}
```

How do you proceed? A straightforward approach could be to define another property, say <em>articleAsDictionary</em> that populates the view with that dictionary.

```swift
class ArticleViewController: UIViewController {
  // now we must change article to optional type
  var article: Article? {
    didSet {
      if let article = article {
        // ...
      }
    }
  }

  // additional model
  var articleAsDictionary: [String:AnyObject]? {
    didSet {
      if let article = articleAsDictionary {
        titleLabel.text = article["title"] as String?
        bodyTextView.text = article["body"] as String?

        if let jsonDate = article["date"] as String? {
          dateLabel.text = /* parse and format date from jsonDate - ouch! */
        }
      }
    }
  }

  // ...
  // any subsequent interaction with the model would result in:
  //
  // if let article = article {
  //  proceed as article as Article
  // } else if let article = articleAsDictionary {
  //  proceed as article as Dictionary
  // } else {
  //   handle no article case
  // } 
  // ...

}
```

Don't ever do this! Any interaction with your model will have to be conditionally handled - a lot of if-s that check what model you're actually working with as explained in the comment above - and that results in messy and overly complex code that's hard to read, hard to improve and susceptible to bugs.

A better solution that might have come to your mind it to convert dictionaries to Article objects. But what if Article is a managed object in Core data? We cannot instantiate it without having a managed object context. Also, defining a common protocol that all article models conform to would not work well for our dictionary and would not support a feature we'll introduce later. We need something different that will still allow us to be independent of a kind of model we might get.

Before continuing to the solution, consider one more thing. Our example of a controller is very simple and in practice it can be more complicated. Remember how we had to process NSDate to get the date in some user-friendlier string format? Not a big deal we might say, but what if we also needed the article view to display that photo thumbnail we've defined in the model earlier. Because our model doesn't have an image, rather a URL to the image, we'd have to additionally burden the controller with networking and downloading logic and maybe even some image processing logic. Think about what else does the controller usually do. To mention a few roles: it listens for orientation changes and adjusts views as necessary, handles events like button taps or view scrolls, acts on gesture recognizer actions, worries about state preservation, controls status and navigation bars and can manage memory. Burdening it with model processing should be avoided when possible. This is a second issue we'll try to solve with MVVM pattern.

In the architecture design one should always aim for simple components (classes). Achieving that often comes in the form of restricting their roles and inputs. That's exactly what we'll do to our view controller. We'll take all the model processing logic out and explicitly define what's expected on the input. The most elegant way to do this is to define a protocol that specifies what exactly view (controller) displays. That'll be our view model.

```swift
protocol ArticleViewViewModel {
  var title: String { get }
  var body: String { get }
  var date: String { get }
}
```

As you can see, the view model protocol defines input exactly in the types the view needs. All properties are of String type. Date is String because date label expects String on its input. Also note that we're requiring only getters for the properties - we'll not be setting any of them from the view controller.

MVVM stands for Model-View-ViewModel. What we're building here might make more sense to be called Model-View-ViewController-ViewModel, but as view and controller are so closely tied into view controller, from the perspective of MVVM we can think of the view controller as the 'view' part of it. We'll stick with MVVM for the sake of simplicity.

Let's refactor our article view controller so is accepts view model instead of various models.

```swift
class ArticleViewController: UIViewController {
  var bodyTextView: UITextView
  var titleLabel: UILabel
  var dateLabel: UILabel

  var viewModel: ArticleViewViewModel {
    didSet {
      titleLabel.text = viewModel.title
      bodyTextView.text = viewModel.body
      dateLabel.text = viewModel.date
    }
  }
}
```

Isn't that just beautiful. All we had to do is map view model properties to our labels. We've made the view controller independent of the model and we threw out all ugly stuff like date formatting or checking what model type do we actually work with. That's great but we still need to get actual data somehow. As you've already guessed it, it'll be done in a concrete implementation of the view model.

There are few ways to make concrete view model. If you're familiar with Swift <a href="https://developer.apple.com/library/ios/documentation/Swift/Conceptual/Swift_Programming_Language/Extensions.html">extensions</a> you could extend Article to conform to ArticleViewViewModel protocol. That's great solution and would work for our Article example, but not for the dictionary and for the reason which will become obvious in next post. Instead, we'll use different approach. We'll create a new class that adopts ArticleViewViewModel protocol and needs an Article object on the input.

```swift
class ArticleViewViewModelFromArticle: ArticleViewViewModel {
  let article: Article
  let title: String
  let body: String
  let date: String
  
  init(_ article: Article) {
    self.article = article
    
    self.title = article.title
    self.body = article.body
    
    let dateFormatter = NSDateFormatter()
    dateFormatter.dateStyle = NSDateFormatterStyle.ShortStyle
    self.date = dateFormatter.stringFromDate(article.date)
  }
}
```

I sympathize, name might not be perfect and it would be nice if Swift would allow nested types, however it is descriptive. And that's our view model! You create it with some article object which is then being processed to fit the view. We would make view model out of a dictionary in same fashion. Populating the view (controller) is straightforward.

```swift
let article: Article = /* some article */
let viewModel = ArticleViewViewModelFromArticle(article)
let viewController = ArticleViewController()
viewController.viewModel = viewModel
```

I hope it's clear what we did here. The controller doesn't know anything about the model any more, all it accepts is some type that conforms to its view model protocol. All model processing is done in view model and view controller supports any number of view model object as long as they conform to its view model protocol.

The solution we've implemented will work only in certain scenarios. Why? You've probably noticed a lot of <code>let</code>-s in our view model which means that properties are constants, not variables, so they're immutable. If we were to make any of them mutable it would not help. Let's clarify that by adding thumbnail to our article view. Thumbnail is an image so we need an image view to display it. Image view wants an image on its input, but our model has a URL to the thumbnail, so how do we proceed? We do not care. We need the image and that's exactly how we'll model our view model. Let's extend the view model protocol so it can provide thumbnail and the view controller so it can display that thumbnail.

```swift
protocol ArticleViewViewModel {
  var title: String { get }
  var body: String { get }
  var date: String { get }
  var thumbnail: UIImage? { get }
}
```


```swift
class ArticleViewController: UIViewController {
  var bodyTextView: UITextView
  var titleLabel: UILabel
  var dateLabel: UILabel
  var thumbnailImageView: UIImageView

  var viewModel: ArticleViewViewModel {
    didSet {
      titleLabel.text = viewModel.title
      bodyTextView.text = viewModel.body
      dateLabel.text = viewModel.date
      thumbnailImageView.image = viewModel.thumbnail
    }
  }
}
```

That was simple. Notice that we've declared image as an Optional type. Reason will become apparent in a bit. (You would also do that if you expect some articles not to have the thumbnail, but that's not our case.)

Who provides the thumbnail? Right, a concrete view model. Let's extend our view model so it downloads the thumbnail from the URL provided by Article model.

```swift
class ArticleViewViewModelFromArticle {
  let article: Article
  let title: String
  let body: String
  let date: String
  var thumbnail: UIImage?
  
  init(_ article: Article) {
    self.article = article
    
    self.title = article.title
    self.body = article.body
    
    let dateFormatter = NSDateFormatter()
    dateFormatter.dateStyle = NSDateFormatterStyle.ShortStyle
    self.date = dateFormatter.stringFromDate(article.date)
    
    let downloadTask = NSURLSession.sharedSession().downloadTaskWithURL(article.thumbnail) {
      [weak self] location, response, error in
      if let data = NSData(contentsOfURL: location) {
        if let image = UIImage(data: data) {
          self?.thumbnail = image
        }
      }
    }

    downloadTask.resume()
  }
}
```

Super! Does it work? No. Downloading of a thumbnail is done asynchronously and it takes time. By the time thumbnail is downloaded our view controller has already assigned all values from the view model to its views (image view, text view and labels) so changing image property of view model some time later does not change image in image view. We need a way to propagate that change.

We could post a notification using NSNotificationCenter and have our view controller observe it, but that is not scalable, makes code difficult to read or follow and requires extra care when registering and unregistering the observer. Another way to inform the controller would be to define a delegate protocol for the view model and have the controller conform to it. That's a bit safer, but it introduces unnecessary dependency and it is even less scalable than notifications - imagine the pain of defining a notify method for each of the view model properties and then implementing those in view controller. Of course, we will not use any of those approaches. What we need is some kind of binding mechanism on property level. We need a way to somehow attach ourselves on a property and get notified each time its value changes.

Mechanisms that do that are called <em>bindings</em> and there is a couple of ways to make them. You could use <a href="https://developer.apple.com/library/ios/documentation/Swift/Conceptual/BuildingCocoaApps/AdoptingCocoaDesignPatterns.html#//apple_ref/doc/uid/TP40014216-CH7-XID_8">Key-Value Observing</a> mechanism but it's not really native to Swift, there is a lot of boilerplate code with it and it can be hard to debug. View model classes would have to inherit NSObject and it would require a lot of vigilance to properly register observers and unregister them when they're not needed any more.

Another solution would be to use <a href="https://github.com/ReactiveCocoa/ReactiveCocoa">ReactiveCocoa</a>. It's a framework inspired by functional reactive programming paradigm and bindings are in heart of it. But it's the framework built with Objective-C in mind (although it can <a href="http://napora.org/a-swift-reaction/">work with Swift</a>) and unless you'll be also using it for some other things, introducing a whole paradigm into your code might be to much.

To allow view model immutability we'll try something more 'Swiftly'. With the power of generics and closures we'll make our simple binding mechanism. In the <a href="http://rasic.info/bindings-generics-swift-and-mvvm/">next post</a>.