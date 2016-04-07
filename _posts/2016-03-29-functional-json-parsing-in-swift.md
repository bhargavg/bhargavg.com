---
layout: post
title: Functional JSON Parsing in Swift
date: 2016-03-29 18:33
categories: Swift
---

There are so many [good](http://chris.eidhof.nl/posts/json-parsing-in-swift.html) [tutorials](https://robots.thoughtbot.com/efficient-json-in-swift-with-functional-concepts-and-generics) written on parsing JSON in swift. This is my take on the same.

<blockquote>
  <em>Posts in this series:</em>
  <ul>
    <li><em><strong><a href="http://bhargavg.com/swift/2016/03/29/functional-json-parsing-in-swift.html">Functional JSON Parsing in Swift</a></strong></em></li>
    <li><em><a href="http://bhargavg.com/swift/2016/03/30/json-parsing-with-value-transformers.html">JSON Parsing With Value Transformers</a></em></li>
    <li><em><a href="http://bhargavg.com/swift/2016/04/05/parsing-arrays-in-json.html">Parsing Arrays in JSON</a></em></li>
    <li><em><a href="http://bhargavg.com/swift/2016/04/07/handling-optional-properties-in-json.html">Handling Optional Properties in JSON</a></em></li>
  </ul>
</blockquote>

We'll try to parse the following basic JSON object into a Swift model.

{% highlight swift %}
// Parse this JSON
let json: [String: Any] = [
    "name": "Bhargav",
    "age": 25
]
// into this Model object
struct Person {
    let name: String
    let age: Int
}
{% endhighlight %}

Before jumping into actual parsing, we need to understand what Currying is.

## Currying
> From Wikipedia: In mathematics and computer science, currying is the technique of translating the evaluation of a function that takes multiple arguments (or a tuple of arguments) into evaluating a sequence of functions, each with a single argument

Currying is a process splitting a function that takes `N` arguments into `N-1` smaller functions (or nested `closures` in this case) that take `1` argument each. Lets take the initializer function for our `Person` struct:

{% highlight swift %}
print(Person.init.dynamicType)
# Output: (String, Int) -> Person
{% endhighlight %}

So, to create a `Person` Object, we need to pass both `name` and `age` to `init` method. Here is how it's curried version looks like.

{% highlight swift %}
// Curried version of Person.init
func makePerson(name: String) -> (age: Int) -> Person {
    return { age in
        return Person(name: name, age: age)
    }
}

print(makePerson.dynamicType)
# Output: String -> Int -> Person
# ie., it's a function that takes a String 
# and returns a function of type Int -> Person

print(makePerson("Bill").dynamicType)
# Output: Int -> Person
# ie., it's a function that takes a String 
# and returns a Person object

print(makePerson("Bill")(age: 49).dynamicType)
# Output: Person
# ie., We got the Person object
{% endhighlight %}

If you can understand the above code snippet, that's all there is to Currying.

Now that we understand what Currying is, let's parse some JSON.

## Parsing - Attempt 1

{% highlight swift %}
func parse1(json: [String: Any]) -> Person? {
    guard let name = json["name"] as? String else {
        return .None
    }
    guard let age = json["age"] as? Int else {
        return .None
    }
    return Person(name: name, age: age)
}

print(parse1(json))
# Output: Optional(Person(name: "Bhargav", age: 25))
{% endhighlight %}

That is too much code just to parse a model with two properties. We can do better by identifying the repeated chunks of code and splitting them into functions.

## Parsing - Attempt 2

{% highlight swift %}
func int(box:[String: Any], key: String) -> Int? {
    return box[key] as? Int
}

func string(box:[String: Any], key: String) -> String? {
    return box[key] as? String
}

func parse2(json: [String: Any]) -> Person? {
    guard let name = string(json, key: "name") else {
        return .None
    }
    
    guard let age = int(json, key: "age") else {
        return .None
    }
    
    return Person(name: name, age: age)
}

print(parse2(json))
{% endhighlight %}

Code for "extracting values from a dictionary as a specific type" is converted in to individual functions `int(:key:)` and `string(:key:)`.

The code for those two functions is the same except the types. We can use Generics to make it into a single function.

## Parsing - Attempt 3

{% highlight swift %}
func get<T>(box:[String: Any], key: String) -> T? {
    return box[key] as? T
}
func parse3(json: [String: Any]) -> Person? {
    guard let name: String = get(json, key: "name") else {
        return .None
    }
    
    guard let age: Int = get(json, key: "age") else {
        return .None
    }
    
    return Person(name: name, age: age)
}

print(parse3(json))
{% endhighlight %}

We created `get` method using Generic type `T`, but where are we passing the type?

If you look closely, the type, `T`, is being inferred from the variable it is being assigned to. The type inferencing is an awesome thing in Swift.

We got the code into a good shape but, still there are two guard statements doing absolutely the same thing and we haven't used our curried function yet. So, let's rewrite the code with `if-let`s and `makePerson` function.


## Parsing - Attempt 4
{% highlight swift %}
func parse4(json: [String: Any]) -> Person? {
    
    let mayBeName: String? = get(json, key: "name")
    let mayBeAge: Int? = get(json, key: "age")
    
    let intermediate: (Int) -> Person
    
    if let name = mayBeName {
        intermediate = makePerson(name)
    } else {
        return .None
    }
    
    if let age = mayBeAge {
        return intermediate(age)
    } else {
        return .None
    }
}

print(parse4(json))
{% endhighlight %}

At first glance, it might look like we made it even worse than previous version. But we can observe a pattern now.

We have a function `makePerson` aka., `f`, an optional value aka., `x` and we perform `f(x)` only if the optional value is present.

Let's define a custom operator `<*>` that does this.

## Parsing - Attempt 5
{% highlight swift %}
infix operator <*> {
    associativity left
    precedence 100
}

// Operator that takes 
//     f: An optional function that takes A and returns B
//     x: An optional value of type A
//
// and returns f(x) if both f and x are present
func <*><A, B>(f: (A -> B)?, x: A?) -> B? {
    guard let f = f, x = x else {
        return .None
    }
    
    return f(x)
}

func parse5(json: [String: Any]) -> Person? {
    return makePerson <*> get(json, key: "name")
                      <*> get(json, key: "age")
}

print(parse5(json))
{% endhighlight %}

There you have it, `JSON` dictionary being decoded beautifully into a Swift model.

> Thanks [Chris Eidhof](http://chris.eidhof.nl/posts/json-parsing-in-swift.html) and [Thoughtbot](https://robots.thoughtbot.com/efficient-json-in-swift-with-functional-concepts-and-generics) for your awesome blog posts on this topic.
