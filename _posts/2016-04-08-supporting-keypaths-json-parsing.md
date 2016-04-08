---
layout: post
title: Supporting Keypaths - JSON Parsing
date: 2016-04-08 13:13
categories: Swift
disqus: true
uuid: C83743C1-C01E-4ECB-B0A8-2284059EAC18
---

Can we handle key paths (like the ones we're used to in NSDictionary) while parsing JSON? That would be cool right? So, why wait, let's implement it.

<blockquote>
  <em>Posts in this series:</em>
  <ul>
    <li><em><a href="http://bhargavg.com/swift/2016/03/29/functional-json-parsing-in-swift.html">Functional JSON Parsing in Swift</a></em></li>
    <li><em><a href="http://bhargavg.com/swift/2016/03/30/json-parsing-with-value-transformers.html">JSON Parsing With Value Transformers</a></em></li>
    <li><em><a href="http://bhargavg.com/swift/2016/04/05/parsing-arrays-in-json.html">Parsing Arrays in JSON</a></em></li>
    <li><em><a href="http://bhargavg.com/swift/2016/04/07/handling-optional-properties-in-json.html">Handling Optional Properties in JSON</a></em></li>
    <li><em><strong><a href="http://bhargavg.com/swift/2016/04/08/supporting-keypaths-json-parsing.html">Supporting keypaths</a></strong></em></li>
  </ul>
</blockquote>


{% highlight json %}
// JSON we want to parse
{
  "status": 200,
  "response": {
    "request_id": 1386,
    "result": {
      "department_name": "National Savings",
      "members": [
        {
          "member_id": 0,
          "name": "Essie Ferrell"
        },
        {
          "member_id": 1,
          "name": "Jaclyn Oneill"
        },
        {
          "member_id": 2,
          "name": "Forbes Tillman"
        },
        {
          "member_id": 3,
          "name": "Fry Odonnell"
        },
        {
          "member_id": 4,
          "name": "Ball Frazier"
        }
      ]
    }
  }
}
{% endhighlight %}

{% highlight swift %}
// Model
struct Member {
    let memberID: Int
    let name: String
}

// Curried initializer
func makeMember(memberID: Int) -> (name: String) -> Member {
    return { name in
        return Member(memberID: memberID, name: name)
    }
}

// Parser
// Nothing fancy, just pass a dictionary and 
// if it can, returns a Member, else a nil
func parseMember(data: [String: AnyObject]) -> Member? {
    return makeMember <*> get(data, key: "member_id")
                      <*> get(data, key: "name")
}
{% endhighlight %}

We defined a method, `parseMember`, method takes a Dictionary, parses it and returns a `Member`. This method can't deal with whole JSON response that we can see above. It can only handle dictionary objects under `members` key.

So, the question now is, how do we extract only the value of `members` key? 

Using key paths.

Currently, there is no support for it in Swift Dictionaries. Our only option is to cast the Swift Dictionary to `NSDictionary` and use its `valueForKeyPath` method.

{% highlight swift %}
func keyPath<T>(box: [String: AnyObject], path: String) -> T? {
    return (box as NSDictionary).valueForKeyPath(path) as? T
}
{% endhighlight %}

Because values in `box` are of type `AnyObject` we used generics to cast the value property when returning.

Where do we use this method?

The ideal place to use this `keyPath` function is during transformations ie., while using `<~~` or `<<~`, depending on whether we are talking about a single dictionary or array of dictionaries.

The problem is, `keyPath` function that we wrote above is not compatible with any of those transformation operators. 

Why?

If you look at the definition of `<~~` operator, the function `f` (`keyPath` in our case now) is of type `A -> B?`. But our `keyPath` is of type `([String: AnyObject], String) -> T?` which is wrong. It should be `[String: AnyObject] -> T?`, because that is what we get from `get` method.

That means, our `keyPath` method should only take one parameter, `[String: AnyObject]`, to be compatible with our operators. 

Then how can we pass the `key.path.info`?

Currying

{% highlight swift %}
func keyPath<T>(path: String) -> (box: [String: AnyObject]) -> T? {
    return { box in
        return (box as NSDictionary).valueForKeyPath(path) as? T
    }
}
{% endhighlight %}

Saw what I did there? We can invoke `keyPath("our.key.path")` which returns us a partially applied function that takes a parameter of type `[String: AnyObect]`. Exactly what we need.

For testing, let's try to extract the `department_name` value:

{% highlight swift %}
let department: String? = get(json) <~~ keyPath("response.result.department_name")
print(department)
{% endhighlight %}

The above code should print the value `Optional("National Savings")`. Sweet...

Let's start the parsing:

{% highlight swift %}
let members = get(json) <~~ keyPath("response.result.members") <<~ parseMember
print(members)
{% endhighlight %}

First `<~~` is used to get the `json` as `[String: AnyObject]` and passed to `keyPath` which extracts the value of key path. This extracted value is then casted to an array of `[String: AnyObject]` and each dictionary is passed to `parseMember`, results are collected and returned.

There you go, we've successfully retrieved values through key path and parsed it. Woohoh..!

You can find the whole source code [as Gist on GitHub](https://gist.github.com/bhargavg/78ea0893df7ef5352b22261b1ecf024b)
