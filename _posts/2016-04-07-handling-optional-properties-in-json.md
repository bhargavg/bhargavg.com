---
layout: post
title: Handling Optional Properties in JSON
date: 2016-04-07 14:57
categories: Swift
disqus: true
uuid: 845BAB33-DC46-4492-BF29-8D83A8750DF4
---

In our [previous](http://bhargavg.com/swift/2016/04/05/parsing-arrays-in-json.html) [versions](http://bhargavg.com/swift/2016/03/30/json-parsing-with-value-transformers.html) of [parser](http://bhargavg.com/swift/2016/03/29/functional-json-parsing-in-swift.html), there is no support for optionals. If any key transformation fails or if any key is missing or if the response contains `nil` values for keys, the whole parsing fails. This is not desired in most of the situations. 

<blockquote>
  <em>Posts in this series:</em>
  <ul>
    <li><em><a href="http://bhargavg.com/swift/2016/03/29/functional-json-parsing-in-swift.html">Functional JSON Parsing in Swift</a></em></li>
    <li><em><a href="http://bhargavg.com/swift/2016/03/30/json-parsing-with-value-transformers.html">JSON Parsing With Value Transformers</a></em></li>
    <li><em><a href="http://bhargavg.com/swift/2016/04/05/parsing-arrays-in-json.html">Parsing Arrays in JSON</a></em></li>
    <li><em><strong><a href="http://bhargavg.com/swift/2016/04/07/handling-optional-properties-in-json.html">Handling Optional Properties in JSON</a></strong></em></li>
    <li><em><a href="http://bhargavg.com/swift/2016/04/08/supporting-keypaths-json-parsing.html">Supporting keypaths</a></em></li>
    <li><em><a href="http://bhargavg.com/swift/2016/04/12/do-try-to-catch-errors-in-json-parsing.html">Error Handling</a></em></li>
  </ul>
</blockquote>

To handle the above cases, we need to add support for Optionals.

Following is the JSON that we want to parse:

{% highlight json %}
// Our JSON to parse
[
  {
    "name": {
      "first_name": "Lex",
      "last_name": "Luthor"
    },
    "age": 28,
    "gender": "male",
    "address": {
      "city": "Edneyville",
      "state": "Maine"
    },
    "skills": [
      "laboris",
      "id",
      "consectetur",
      "duis",
      "incididunt",
      "eiusmod"
    ]
  },
  {
    "age": 29,
    "gender": "female",
    "address": {
      "street": "754 Marconi Place",
      "city": "Rosburg",
      "state": "Louisiana"
    },
    "skills": [
      "laborum",
      "et",
      "proident",
      "dolor",
      "exercitation",
      "nisi"
    ]
  }
]
{% endhighlight %}

If you observe carefully, there are 2 small (yet breaking) changes from our previous version:

1. For the 1st person, `address` doesn't have a `street` and as per our requirements, this is an invalid address
2. For the 2nd person, `name` key is missing, which again leads to an invalid person

In both the cases, parsing will fail by returning `nil` objects. Let's modify our `Customer` model to handle this `nil` value by making `address` property optional. Also, change the curried function so that it now takes an optional `Address`:

{% highlight swift %}
struct Customer {
    let name: String // first_name, last_name
    let age: Int
    let gender: Gender
    let address: Address?  // may be nil
    let skills: [String]
}

func makeCustomer(name: String) -> (age: Int) -> (gender: Gender) -> (address: Address?) -> (skills: [String]) -> Customer {
    return { age in { gender in { address in { skills in
        return Customer(name: name, age: age, gender: gender, address: address, skills: skills)
    }}}}
}
{% endhighlight %}




## `<?>` Operator
We've used our custom operator `<*>` till now to push values into our curried model initializers. Unfortunately, it doesn't support optional values (see the definition, it performs `f(x)` only if both `f` and `x` are present).

Overriding `<*>` with optional support didn't help, Swift compiler got stuck for a very long duration during compilation. So I had to define a new operator `<?>` for this. 

{% highlight swift %}
infix operator <?> {
    associativity left
    precedence 100
}

func <?><A, B>(f: (A? -> B)?, x: A?) -> B? {
    guard let f = f else {
        return nil
    }
    return f(x)
}
{% endhighlight %}

> Please let me know if you can get this to work without creating a new operator.

This new operator `<?>` does almost same job as `<*>` except `x` can be an optional now. Because our function `f` accepts an optional value, the operator just passes the optional value `x` directly to `f`.

## Parsing
Now that we've defined our new operator to handle Optional values, lets parse the model:
{% highlight swift %}
makeCustomer <*> get(data, key: "name")     <~~ parseName
             <*> get(data, key: "age")
             <*> get(data, key: "gender")   <~~ parseGender
             <?> get(data, key: "address")  <~~ parseAddress 
             <*> get(data, key: "skills")
{% endhighlight %}

The above code will parse the JSON into `Customer` model and if it couldn't create `Address` model, it simply assigns `nil` value to address property of `Customer`.


There is one slight problem with this approach. This will only work if the whole property is optional ie., `T?`. You cannot have a property of type `[T?]` ie., array of Optional values.  I'm experimenting few ideas (mostly not involving custom operators). 

Please feel free to comment below, if you have suggestions and questions.

You can find the whole source code [as Gist on GitHub](https://gist.github.com/bhargavg/d1d16d04d342d5804d8b2ee7337b840e)

