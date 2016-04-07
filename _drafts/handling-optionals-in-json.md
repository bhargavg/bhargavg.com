---
layout: post
title: Handling Optionals in JSON
date: 2016-04-07 14:57
categories:
---


In our previous versions, there is no support for optionals. If any key transformation fails or if any key is missing or if the response contains `nil` values for keys, the whole parsing fails. This is not desired in most of the situations. 

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

In above both cases parsing will fail returning `nil` objects. Let's modify our `Customer` model to handle the `nil` address by making it an optional. Also, change the curried function so that it now takes an optional `Address`:


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

The above code will parse the `Customer` model and if it couldn't create a `Address` object from JSON, it simply assigns `nil` value to address property of `Customer`.


## Optionals in arrays

So, we've added support for Optional values with an operator, we're done right?

Not exactly. There is one important use case with using Optionals that we've missed, ie., Optionals in an array.

If you parse the above json with our newly defined opertor, we'll only get an array of size 1, whereas we have 2 customers in our JSON.

{% highlight swift %}
let customers: [Customer]? = get(jsonData) <<~ parseCustomer
print(customers!)
#Output: [CustomerObject]
{% endhighlight %}

This is because, as previously stated, second person is invalid because of missing `name` key in JSON. So, this value will be removed from the final `Customer`s array (because of `flatMap{ $0 }` in `<<~` operator definition)

There will be situations, where you don't this behaviour. Instead you want that `nil` to be present in final parsed array.

We need to understand difference between `[T]?` and `[T?]?`

- `[T]?` - Optional array containing `T` type elements
- `[T?]?` - Optional array containing optional `T` type elements

The following code helps to digest this better:

{% highlight swift %}
# Array of optional integers
var mayBeInts: [Int?] = [1, 2, nil, 4, nil]
print(mayBeInts)
#Output: [Optional(1), Optional(2), nil, Optional(4), nil]

# Compile Error: Cannot assign nil to non optional
mayBeInts = nil

# Optional array of optional integers
var mayBeIntsInMayBeArray: [Int?]? = [1, 2, nil, 4, nil]
print(mayBeIntsInMayBeArray)
#Output: Optional([Optional(1), Optional(2), nil, Optional(4), nil])

# No compile time error
mayBeIntsInMayBeArray = nil
print(mayBeIntsInMayBeArray)
#Output: nil
{% endhighlight %}

