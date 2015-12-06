---
layout: post
title:  "Reactive Form Validation with Bond, Swift Bond"
date:   2015-02-21 15:17:58
categories: ios swift
permalink: reactive-form-validation-with-bond-swift-bond
disqus: true
---

<em>Note: This article is now outdated. I'll be updating it soon with Bond 4 syntax.</em>

Writing a form validator doesn’t require above-average IQ, but a mere thought about it summons uneasiness. There is so much boilerplate work to it that makes you hate your profession. You have to implement delegates for multiple text fields and other controls, pull out their values and write conditional code so that you could finally make a button reflect validity of the form. Fortunately, there is a way to avoid all that. We'll do it by implementing a form validator using <a href="https://github.com/SwiftBond/Bond">Bond - a Swift binding framework</a>.

Bond solves the problem of binding dynamic values to views, controls or other interested objects, but there is another aspect to it that might not be obvious at first. It allows us to create new dynamic values out of existing ones through functional concepts like <em>map</em>, <em>filter</em> and <em>reduce</em>. With those concepts we can build hierarchies of dynamic values that transform source into something useful for our needs.

Let's start by defining a simple <em>Sign up</em> screen with a text field for username, two text fields for password and a sign up button.

```swift
class ViewController: UIViewController {
  @IBOutlet weak var userNameTextField: UITextField!
  @IBOutlet weak var passwordTextField: UITextField!
  @IBOutlet weak var confirmPasswordTextField: UITextField!
  @IBOutlet weak var signUpButton: UIButton!
  
  override func viewDidLoad() {
    super.viewDidLoad()
    // we'll put rest of the code here
  }
}
```

Our goal is to make sign up button disabled as long as user hasn't input valid data. What is valid data? We can define it with these three rules:

<ul>
	<li>Username has at least three characters</li>
	<li>Password has at least eight characters</li>
	<li>Confirmation password is equal to password</li>
</ul>

Let's start with the first rule. Whenever text in username text field changes we need to enable sign up button if text has three or more characters or disable it if it doesn't have. When you think in terms of functional programming about that problem you'll see that we need to transform string value to boolean value by applying some rule. Type transformation like that can be done by <em>map</em> function. Finally, we need to update button's <em>enabled</em> property with transformed value. With Bond, all that can be done like this:

```swift
let username: Dynamic<String> = userNameTextField.dynText

let usernameLengthRule: Dynamic<Bool> = username.map { countElements($0) >= 3 }

usernameLengthRule ->> signUpButton.dynEnabled
```

First we get dynamic representation of username text field's <em>text</em> property. Then we apply map to it in order to transform it to boolean by applying given rule. Finally, we bind it to sign up button's <em>enabled</em> property through corresponding <em>dynamic</em> (provided by Bond framework). I've intentionally written it in three lines with explicit types to make clear what we're doing, but the snippet could have easily been one-liner.

Second rule is pretty much same as the first one.

```swift
let password = passwordTextField.dynText

let passwordLengthRule = password.map { countElements($0) >= 8 }
```

You might be asking yourself were is the binding line. Problem is that binding another dynamic to <em>dynEnabled</em> would not produce desired effect. Those two dynamics would drive button's enabled state independently of one another. What we actually want is to somehow model conjunction of our two rules. We want button to be visible if and only if both of the rules are satisfied. Say hi to <em>reduce</em>. It's a function that takes multiple dynamics and reduces them to just one by applying given transformation - boolean <em>and</em> in our case. Let's see how that looks like in code.

```swift
let username = userNameTextField.dynText
let password = passwordTextField.dynText

let usernameLengthRule = username.map { countElements($0) >= 3 }
let passwordLengthRule = password.map { countElements($0) >= 8 }

let formValidRule = reduce(usernameLengthRule, passwordLengthRule) { $0 && $1 }

formValidRule ->> signUpButton.dynEnabled
```

Smooth ride so far. Final rule says that the values of password text field and confirm password text field must match. We have two dynamics that should be transformed to one so we'll use <em>reduce</em> once again.

```swift
let confirmPassword = confirmPasswordTextField.dynText

let passwordsMatchRule = reduce(password, confirmPassword, ==)
```

We can witness Swift's awesomeness here. For final argument of <em>reduce</em> we needed a function that accepts two Strings and returns a Bool. Because infix operators in Swift are treated as binary functions and because operator == is overloaded to compare two Strings, we were able to pass just the operator itself.

To wrap things up, here is the final code of our form validator.

```swift
class ViewController: UIViewController {
  @IBOutlet weak var userNameTextField: UITextField!
  @IBOutlet weak var passwordTextField: UITextField!
  @IBOutlet weak var confirmPasswordTextField: UITextField!
  @IBOutlet weak var signUpButton: UIButton!
  
  override func viewDidLoad() {
    super.viewDidLoad()
    
    let username = userNameTextField.dynText
    let password = passwordTextField.dynText
    let confirmPassword = confirmPasswordTextField.dynText
    
    let usernameLengthRule = username.map { countElements($0) >= 3 }
    let passwordLengthRule = password.map { countElements($0) >= 8 }
    let passwordsMatchRule = reduce(password, confirmPassword, ==)
    
    let formValidRule = reduce(usernameLengthRule, passwordLengthRule, passwordsMatchRule) {
      $0 && $1 && $2
    }
    
    formValidRule ->> signUpButton
  }
}
```

Notice that we've dropped <em>.dynEnabled</em> in last line. That's because that Dynamic is treated as <em>designated dynamic</em>. You can find more info on that on <a href="https://github.com/SwiftBond/Bond">Bond's GitHub</a>.

If you've not encountered it previously, you might have not been aware that presented solution is implemented using <a href="http://en.wikipedia.org/wiki/Functional_reactive_programming">Functional reactive programming</a> paradigm. We can think of Dynamics as objects that produce signals that we later transform and act upon. It's a declarative paradigm that allows one to focus on what needs to be done, rather than on how to do it. Bond brings some aspects of FRP to iOS development in Swift. If you are developing in Objective-C, check out <a href="https://github.com/ReactiveCocoa/ReactiveCocoa">ReactiveCocoa</a> - a FRP framework for iOS.