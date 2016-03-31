---
layout: post
title: JSON Parsing With Value Transformers
date: 2016-03-30 21:06
categories: Swift
---

In [the last tutorial](http://bhargavg.com/swift/2016/03/29/functional-json-parsing-in-swift.html) we've seen how to use functional concepts to parse JSON into Swift models. One of the main problems with that code is it doesn't support value transformations, ie., changing the values coming from JSON before assigning them to the model. Let's try to implement it.

{% highlight swift %}
// JSON to Parse
let json: [String: Any] = [
    "name": "Bill",
    "age": 47,
    "is_married": "s"
]

// Our model
struct Person {
    let name: String
    let age: Int
    let isMarried: Bool
}

// Modified curry function
func makePerson(name: String) -> (age: Int) -> (isMarried: Bool) -> Person {
    return { age in { isMarried in
        Person(name: name, age: age, isMarried: isMarried)
    }}
}
{% endhighlight %}

As you can see, the `is_married` flag from JSON is either `s` or `m` depending on whether the `Person` is married or not. These strings need to be mapped into `Bool` before assigning them to our `Person`.

The `parseIsMarried` does this mapping for us:
{% highlight swift %}
func parseIsMarried(value: String) -> Bool? {
    switch value {
    case "s":
        return false
    case "m":
        return true
    default:
        return .None
    }
}
{% endhighlight %}

For applying this transformation method, let's define one more operator that has a higher precedence than `<*>`

{% highlight swift %}
infix operator <~~ {
    associativity left
    precedence 110  // Precedence slightly higher than <*>
}

// Operator that takes
//      x: Optional value of type A
//      f: Transformation function that maps from 
//         A to an Optional B (because the transformation can fail)
//
// and returns f(x) if x is present (applies the transformation)
func <~~<A, B>(x: A?, f: (A -> B?)) -> B? {
    guard let x = x else {
        return .None
    }
    return f(x)
}
{% endhighlight %}

This newly defined operator can be used as follows:

{% highlight swift %}
makePerson <*> get(dict, key: "name")
           <*> get(dict, key: "age")
           <*> get(dict, key: "is_married")  <~~ parseIsMarried
{% endhighlight %}

The way it works is:

- `get` function will retrieve the `is_married` value from JSON as a `String`
- This value is passed to `parseIsMarried` function (because `<~~` is having higher precedence than `<*>`)
- The transformation function then maps the value to appropriate boolean value
- This mapped value will then be passed to our curried function.

Using this transformation operator, we can even map nested objects as well.

The whole source code (along with more usecases for transformation) can be found [in this gist](https://gist.github.com/bhargavg/66d77a8b162740bc70999de3a9376389)
