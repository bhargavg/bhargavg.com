---
layout: post
title: Error Handling Patterns In Swift
date: 2016-04-28 11:51
categories: Swift
disqus: true
uuid: E17E560B-09A8-4FDB-8876-760002C76B65
---

Error handling in Objective-C is mostly done through `NSError` object:

- For synchronous functions, we pass the double pointer of `NSError` to a function that can fail and if the function hits a error condition, it will set the error object and return
- For asynchronous functions, `NSError` is passed as an argument to the completion block

{% highlight objective-c %}
// Patterns of error handling in Objective-C
- (BOOL)methodThatCanFail:(NSError *__autoreleasing *)err
- (ReturnValue *)methodThatCanFail:(NSError *__autoreleasing*)err
- (void)methodThatCanFail:(void(^)(ReturnValue *, NSError *))block
{% endhighlight %}

We can do follow the same Pattern in Swift, but it is not advised. There are more swiftier ways of handling errors and in this post I'll be discussing them.

## Optionals
An `Optional` is a box that can contain a value or nothing. This is particularly suited when a function can fail, but we are not concerned about why it failed.

{% highlight swift %}
func methodThatCanFail() -> ReturnValue? {
 ...
}

if let value = methodThatCanFail() {
  // method successfully returned a value
} else {
  // method failed
}

{% endhighlight %}


## Throws
In some cases, ignoring errors might not be an option. We might need to do different things based on why a particular method failed. Using `try-catch` is best option in this case.

{% highlight swift %}
// All the error cases encapsulated as an Enum
enum FetchError: ErrorType {
    case UnknownError
    case UnAuthorized
}

// function is marked as throws to indicate that it can fail
func fetchPosts() throws -> [Post] {
    ...
}

do {
  // try to run the function
  try fetchPosts()
} catch FetchError.UnAuthorized {
  // handle UnAuthorized case
  showLoginScreen()
} catch {
  // handle all the other error cases
  showErrorAlertDialog()
}
{% endhighlight %}

## Result enum
Lately, I have seen lot of open source projects using this method to handle errors in more functional manner.

This is an extension of `Optional` pattern where in error information is not discarded. 

An Optional, in swift, is implemented as:

{% highlight swift %}
enum Optional<T> {
    case Some(T)
    case None
}
{% endhighlight %}

From the above definition, we can see that, `.Some` can carry set value, but `.None` has no associated data. Let's define another enum that has associated error type for `.None` case as well:

{% highlight swift %}
enum Result<T, U: ErrorType> {
  case Success(T)
  case Failure(U)
}
{% endhighlight %}

Now, failing functions can be made to return this `Result` box

{% highlight swift %}
func fetchPosts() -> Result<[Post], FetchError> {
  ...
}

let result = fetchPosts()

switch result {
  case let .Success(posts):
    display(posts: posts)
  case let .Failure(error):
    handle(error: error)
}

func handle(error: error) {
  switch error {
    case .UnknownError
      showErrorAlertDialog()
    case .UnAuthorized
      showLoginScreen()
  }
}
{% endhighlight %}

Even though above version might look like it has a lot of code, I really like the fact that every function does what it's supposed to do. Nothing more, nothing less.

This Result enum is already packed into a neat little library by [Antitypical](https://github.com/antitypical/Result) with all the bells and whistles.

I'm still digging into this pattern and will hopely write more about it in future...
