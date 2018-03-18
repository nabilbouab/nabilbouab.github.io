---
layout: post
title:  "JavaScript Promises"
author: "Nabil"
---

Javascript is a single-threaded programming language . If you have already programmed in JavaScript, you must have come across callback. If we perform a costly operation like I/O, or HTTP request that reaches a server to the other part of the world, we do not want to wait for it to finish before executing the next instruction. Instead, we use asynchronous programming.

```javascript
function printUserIdAsync() {
  makeRequest(`posts?userId=2`, (err, results) => {
    console.log(results[0].userId);
  });
}
```

This is a piece of code that illustrates how async programming can be done. It might not seem obvious for those who are not familiar with async programming, but the makeRequest function takes two parameters. The first one is the URL where the request should be routed and the second is what we call a callback function. The callback function is the code that should be executed when the request comes back to us.

What is the problem?

This piece of code is fairly simple. However, you can very easily get into what we call a callback hell. Here is an example:

```javascript
fs.readdir(source, function (err, files) {
  if (err) {
    console.log('Error finding files: ' + err)
  } else {
    files.forEach(function (filename, fileIndex) {
      console.log(filename)
      gm(source + filename).size(function (err, values) {
        if (err) {
          console.log('Error identifying file size: ' + err)
        } else {
          console.log(filename + ' : ' + values)
          aspect = (values.width / values.height)
          widths.forEach(function (width, widthIndex) {
            height = Math.round(width / aspect)
            console.log('resizing ' + filename + 'to ' + height + 'x' + height)
            this.resize(width, height).write(dest + 'w' + width + '_' + filename, function(err) {
              if (err) console.log('Error writing file: ' + err)
            })
          }.bind(this))
        }
      })
    })
  }
})
```

This example is taken from callbackhell.com. This piece of code is not composable, not maintainable and not easily debuggable. In an ideal world, we would like to keep the asynchronous aspect of this code without the drawback of nested callbacks. This is when promises come into the play.



How Promises are a solution to this problem?

In order to answer this question, let’s explain what is a promise in a very concrete way. Let’s begin with an example of a promise code:

```javascript

import loadUsersPromise from ./load_user_promise

loadUsersPromise('https://jsonplaceholder.typicode.com/users')
  .then((users) => {
    var elem = document.createElement("div");
    for(user in users){
       var elem_user = document.createElement("div");
       elem_user.appendChild(user.name);
    }
  });
```

In the first line, we import a library. This library consists of one function that takes a URL as a parameter and returns a promise for this url. What does it mean? It means that we have an object which encapsulate a value that we do not have yet, but on which we can reason. It is a sort of blackbox to which we attach a callbacks when the promise resolves. This is what we do thanks to `then` function which takes two parameters — one callback if the promise is *fullfiled* and another callback is the promise is *rejected*. We tell the promise “when you finish, execute this function and give me the result in the parameter”. Thus, we can apply any logic within the callback. In this example, we are printing to the screen the name of all the users we receive from the call to the server.


Composability of promises.

Promises have an interesting property: they are composable. In fact, we can chain them as we want like this:

```javascript
getUserByName('nolan').then(function (user) {
  if (inMemoryCache[user.id]) {
    return inMemoryCache[user.id];    // returning a synchronous value!
  }
  return getUserAccountById(user.id); // returning a promise!
}).then(function (userAccount) {
  // I got a user account!
});

// see https://pouchdb.com/2015/05/18/we-have-a-problem-with-promises.html
```

Let’s talk about how .then(...) function makes chaining possible and then talk about the gotchas of chaining.

In the code above, we can see that we return a value line 3 `return inMemoryCache[user.id]` . But how come, we are still able to call `then` on it?This method is available only on promises not on other variable types? In fact, `then` always creates a promise and returns this promise and this is why we can chain promises together.

What are the gotchas of chaining? I found a very interesting github gist about that here. At the end of the article, the author gave a puzzle that we have to resolve: here they are, with an explanation for each one of them.

Explanation of Puzzle 1:

Since the first `then` returns a promise, the second `then` is called on a promise and wait for it to resolve in order to execute it’s own callback.

Explanation of Puzzle 2:

Since the first `then` returns `undefined` the `then` is executed with `undefined` as `this` and executes immediately, because it doesn’t have any promise to wait for to resolve.

Explanation of Puzzle 3:

The parameter of the `then` function is not a function, but the result of the function which is a promise. Instead, what we should give to `then` is a callback function which take as parameter the result of the previous promise in the chain, and this callback function will be executed. Here, we give a function that is immediately executed. Meanwhile, the first promise resolved, so the value is passed to the default callback function that was constructed to the second `then`. The second `then`, construct a promise around that value and returns the value, which is caught and passed as parameter to the last then.

Explanation of Puzzle 4:

In this example, the second `then` takes as parameter a function. So the callback to the first call of `doSomething()` is now a function. This function will be executed when the first promise resolves. Same applies to the second `then` .

I hope this article was useful!
