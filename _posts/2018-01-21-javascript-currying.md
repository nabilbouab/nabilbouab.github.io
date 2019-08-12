---
layout: post
title:  "JavaScript Currying"
author: "Nabil"
categories: JavaScript
---

Have you ever come across a JavaScript code where a function was called like this `foo(1)(2)(3)` and you were not sure what was happening? If this is the case, you are reading the right blog post.

Currying is a functional programming technique that enables us to create a function with less arguments from a function that takes a lot of arguments. For example, let’s take the classic example of adding 3 numbers a, b and c.

```javascript
function add(a,b,c){
  return a+b+c;
}

function add(a){
  return function(b){
    return function(c){
      return a+b+c;
    }
  }
}
```
The first version of the add function must be straight forward. The function takes three arguments and returns the sum of the three. Easy.

In the curried version, each function takes one argument and returns a new function. If we call `var add1 = add(1)`, this will return a function that closes over 1 as value for the variable a . Since it returned a function that we saved in variable add1, we can write `var add12 = add1(2)`. add12 function closes over a=1 and b=2. We can do the same: `var res = add12(3)` . res will now be equal to 6. We can avoid saving intermediate functions in variables and just write `var res=add(1)(2)(3)` .

```javascript

var add1 = add(1)
var add12 = add1(2)
var res = add12(3)

var res = add(1)(2)(3)
```

That's it! Thanks for reading.
