---
layout: post
title: Parsing Arrays in JSON
date: 2016-04-05 22:48
categories: Swift
---

[Till]({{site.url}}/swift/2016/03/29/functional-json-parsing-in-swift.html) [now]({{site.url}}/swift/2016/03/30/json-parsing-with-value-transformers.html) we've successfully parsed simple JSON dictionaries along with value transformations. In this post, we'll see how to parse array of objects.

{% highlight json %}
[
  {
    "name": {
      "first_name": "Lex",
      "last_name": "Luthor"
    },
    "age": 28,
    "gender": "male",
    "address": {
      "street": "244 Clifton Place",
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
    "name": {
      "first_name": "Ada",
      "last_name": "Stuart"
    },
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
{% endhighlight %}

{% highlight swift %}
// Our Models
struct Customer {
    let name: String // first_name, last_name
    let age: Int
    let gender: Gender
    let address: Address
    let skills: [String]
}

enum Gender {
    case Unknown, Male, Female
}

struct Address {
    let street: String
    let city: String
    let state: String
}


// Curried initializers
func makeAddress(street: String) -> (city: String) -> (state: String) -> Address {
    return { city in { state in
        return Address(street: street, city: city, state: state)
    }}
}

func makeCustomer(name: String) -> (age: Int) -> (gender: Gender) -> (address: Address) -> (skills: [String]) -> Customer {
    return { age in { gender in { address in { skills in
        return Customer(name: name, age: age, gender: gender, address: address, skills: skills)
        
    }}}}
}
{% endhighlight %}

We've already defined a custom operator `<~~` that transforms the given value using a transformation function. Now, let's define another operator `<<~` that does almost the same thing, except it now applies the given transformation on each value of provided array. Because the transformation may fail, we can use `flatMap` to remove `nil` values that resulted due to failed transformations.

{% highlight swift %}
infix operator <<~ {
    associativity left
    precedence 110
}

// An operator that applies f on each value of x
// and return the values that came out through
// successfull transformation
func <<~<A, B>(x: [A]?, f: (A -> B?)) -> [B]? {
    guard let x = x else {
        return nil
    }
    
    // apply f on each value of x
    // and remove the nil values
    // (values that couldn't be
    // transformed by f)
    return x.map(f).flatMap{ $0 }
}
{% endhighlight %}

`NSJSONSerialization.JSONObjectWithData` returns an `AnyObject`. But, to parse our above JSON array, we need to cast it to `[AnyObject]`. 

We can simply write:

{% highlight swift %}
let x = NSJSONSerialization.JSONObjectWithData(...) as? [AnyObject]
{% endhighlight %}

Because, we are following functional approach, let's overload the `get` function that does this casting for us in more functional way:

{% highlight swift %}
// A function that takes AnyObject and tries to
// cast it to a specific type T (can fail, that's why it is Optional)
func get<T>(item: AnyObject) -> T? {
    return item as? T
}
{% endhighlight %}

Now, we got everything we need:

 - A function, `get`, that converts our `AnyObject` we got from `NSJSONSerialization.JSONObjectWithData(...)` into an array `[AnyObject]`
 - A custom operator, `<<~`, that takes this `[AnyObject]` and applies the transformation function

So, let's start the parsing

{% highlight swift %}
let customers = get(jsonObject) <<~ parseCustomer

guard let customers = customers  else {
    print("Couldn't parse customers")
    exit(1)
}

print(customers)

// Transform the given String to a Gender Enum
func parseGender(gender: String) -> Gender {
    switch gender {
    case "male":
        return .Male
    case "female":
        return .Female
    default:
        return .Unknown
    }
}

// transform the given Dictionary of "first_name" and "last_name"
// into a name of the form "first_name, last_name"
func parseName(data: [String: AnyObject]) -> String? {
    return ["first_name", "last_name"].map{ data[$0] as? String }
                                      .flatMap{ $0 }
                                      .joinWithSeparator(", ")
}

func parseAddress(data: [String: AnyObject]) -> Address? {
    return makeAddress <*> get(data, key: "street")
                       <*> get(data, key: "city")
                       <*> get(data, key: "state")
}

func parseCustomer(data: [String: AnyObject]) -> Customer? {
    return makeCustomer <*> get(data, key: "name")     <~~ parseName
                        <*> get(data, key: "age")
                        <*> get(data, key: "gender")   <~~ parseGender
                        <*> get(data, key: "address")  <~~ parseAddress
                        <*> get(data, key: "skills")
}
{% endhighlight %}

You can find the whole source code [as Gist on GitHub](https://gist.github.com/bhargavg/7e74850ab7a042b5d012a5ffb32b670b)
