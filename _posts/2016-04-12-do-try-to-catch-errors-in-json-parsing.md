---
layout: post
title: Do Try to Catch Errors in JSON Parsing
date: 2016-04-12 20:20
categories: Swift
disqus: true
uuid: DC55EF3C-FE60-4990-A4BF-7F3C00A66463
---

There are 2 pain points in our parser code that we've written in our previous posts:

- No error communication mechanism. Caller won't know why the parsing failed.
- Custom operators. This is not really a huge pain, but let's see if we can reduce the count.

In this post, I'll try to address both the problems by rewritting the parser code a bit.

<blockquote>
  <em>Posts in this series:</em>
  <ul>
    <li><em><a href="http://bhargavg.com/swift/2016/03/29/functional-json-parsing-in-swift.html">Functional JSON Parsing in Swift</a></em></li>
    <li><em><a href="http://bhargavg.com/swift/2016/03/30/json-parsing-with-value-transformers.html">JSON Parsing With Value Transformers</a></em></li>
    <li><em><a href="http://bhargavg.com/swift/2016/04/05/parsing-arrays-in-json.html">Parsing Arrays in JSON</a></em></li>
    <li><em><a href="http://bhargavg.com/swift/2016/04/07/handling-optional-properties-in-json.html">Handling Optional Properties in JSON</a></em></li>
    <li><em><a href="http://bhargavg.com/swift/2016/04/08/supporting-keypaths-json-parsing.html">Supporting keypaths</a></em></li>
    <li><em><strong><a href="http://bhargavg.com/swift/2016/04/12/do-try-to-catch-errors-in-json-parsing.html">Error Handling</a></strong></em></li>
  </ul>
</blockquote>

## Custom Operators

Till now, we have defined four custom operators: `<*>`, `<?>`, `<~~`, `<<~`.  Couple of days back, I was trying to minimize our custom operators count and `<*>`, `<?>` looked like possible candidates. Thanks @mikeash for suggesting this.

The only use for `<*>` and `<?>` operators is to inject the values in to our curried initalizers. So, if we remove currying altogether, we can get rid of these two operators.

How you ask?

## Failable Initializers
As the name suggests, Failable Initalizers (`init?`), are a special class of init methods that can fail. A failable initializer is not guaranteed to return an object every time it is invoked. Incase an object couldn't be created, it returns a `nil`.

{% highlight swift %}
struct Foo {
  let x: String
  let y: String?

  init?(data: [String: AnyObject]) {
    let maybeX = get(data, key: "x")
    let maybeY = get(data, key: "y")

    // x is not optional, so make sure
    // it is not nil
    guard let x = maybeX else {
        print("Couldn't read x from JSON")
        return nil
    }

    self.x = x

    // y can be nil, so even if `get` method
    // fails to retrieve the value, its fine
    self.y = maybeY
  }
}
{% endhighlight %}

This approach looks fine at first glance, but it has one major flaw. The failures are getting consumed inside the `guard` statements. There is no way for the caller to know why the parsing failed. This will be problematic if we want to debug.

Why are we even using `guard`s in the first place?

To handle _failure_ right? I mean, `get` does 2 things:

1. Retrieve the value from the dictionary. This can fail if key is not present or it has `null` value.
2. Cast the value (of type `AnyObject`) to the required type. This can also fail if the value cannot be casted successfully.

Right now, we are handling these failure by making `get` return an optional, requiring us to use optional binding via `guard` statements. 

Is there a better way?

## Throwing errors
Swift has a `try-catch` error handling mechanism that perfectly suits this situation. 

Basic control flow using errors is:

- If a function can throw error, it should be marked as such by using `throws` keyword
- Inside a function, errors can be thrown using `throw <OUR_ERROR>`. The `<OUR_ERROR>` is one of the cases for our custom enum that is of `ErrorType`
- For calling a function that can `throw` an error, the caller should prepend the function call with `try` and the whole statement should be inside a `do` block followed by a `catch` block
- If the function throws error, control directly jumps from `do` block to the corresponding `catch` block

So, to adopt this, our first step would be to create an enum of `ErrorType` for representing possible failure conditions:

- A `nil` value case
- An invalid type case

{% highlight swift %}
typealias JSON = [String: AnyObject]

enum ParseError<T, U>: ErrorType {
    case InvalidType(T, expected: U)
    case NilValue(T)
}

extension ParseError: CustomStringConvertible {
    var description: String {
        switch self {
        case .InvalidType(let aThing, let expectedType):
            return "\(aThing) is not of type: \(expectedType)"
        case .NilValue(let key):
            return "\(key) is nil"
        }
    }
}
{% endhighlight %}

The `get` and `keyPath` methods need to be modified to throw `ParseError` instead of returning Optionals.

{% highlight swift %}
import Foundation

func get<T>(box: JSON, key: String) throws -> T {
    
    typealias ParseErrorType = ParseError<String, T.Type>
    
    guard let value = box[key] else {
        throw ParseErrorType.NilValue(key)
    }
    
    guard let typedValue = value as? T else {
        throw ParseErrorType.InvalidType(key, expected: T.self)
    }
    
    return typedValue
}

func get<T>(item: AnyObject) throws -> T {
    guard let typedItem = item as? T else {
        throw ParseError.InvalidType(item, expected: T.self)
    }
    
    return typedItem
}


func keyPath<T>(path: String) -> (box: JSON) throws -> T {
    
    typealias ParseErrorType = ParseError<String, T.Type>
    
    return { box in
        guard let value = (box as NSDictionary).valueForKeyPath(path) else {
            throw ParseErrorType.NilValue(path)
        }
        
        guard let typedValue = value as? T else {
            throw ParseErrorType.InvalidType(path, expected: T.self)
        }
        
        return typedValue
    }
}
{% endhighlight %}

It might look like we've done a lot of things, but all we did is:

- Mark our methods with `throws` indicating that this method can throw an error
- Identify possible failures and throw corresponding errors using a combination of `guard` and `throw` statements

The `typealias` thingy in each function might look a bit, I don't know, _odd?_. We need that because of the generics. Please let me know if you can suggest a better approach.

Let's modify our `init` method to use our new error handling mechanism.

{% highlight swift %}
struct Foo {
  let x: String
  let y: String?

  init(data: JSON) throws {
    x = try  get(json, key: "x")
    y = try? get(json, key: "y")
  }
}
{% endhighlight %}

We've done few things here:

- `init` is no longer Failable. Instead, incase of failure, it just forwards the error to the caller.
- The Optional value is handled by turning the error into an optional by using `try?` 

## Value Transformers

All looks good, we've successfully parsed a JSON into a struct with out using currying and our `<*>`, `<?>` operators. Finally, we can delete those.

We are left with our `<~~` and `<<~~` transformation operators. Let's modify these too.

{% highlight swift %}
infix operator <~~ {
    associativity left
    precedence 110
}

infix operator <<~ {
    associativity left
    precedence 110
}

func <~~<A, B>(x: A, f: (A throws -> B)) throws -> B {
    return try f(x)
}

func <<~<A, B>(x: [A], f: (A throws -> B)) throws -> [B] {
    return try x.map(f)
}
{% endhighlight %}


That's it, we've modified our code to use our new error handling mechanism.

## Zero operators

The remaining custom operators, `<~~` and `<<~`, are not necessary for our parser to work. All they do is help you to write:

{% highlight swift %}
x <~~ g <~~ f
x <<~ g <<~ f
{% endhighlight %}

instead of equivalent longer version:

{% highlight swift %}
f(g(x))
x.map(g).map(f)
{% endhighlight %}

If you are comfortable with longer version, even these operators can be deleted. 

Wohoo..! zero custom operators.

You can find the whole source code [as Gist on GitHub](https://gist.github.com/bhargavg/9fc4189b696ead1c44c97854fe9ca02c)

## Library
I've made this into a library called [Banana](https://github.com/bhargavg/Banana). Contributions are welcome...
