---
layout: post
title:  "async-await"
author: "Nabil"
categories: JavaScript
---

This post is about asynchronous programming in JavaScript with async/await. Let’s say we have a function that enables us to get data from a server located in another country which is reachable via this url https://jsonplaceholder.typicode.com which can give us the data regarding some blog posts. Let’s do this! So we use the promise based api that our browser gives us in order to fetch data from the web which is the fetch function. this returns us a Promise that will be fulfilled with the data that the server returns (aka the posts that we are looking for). So let’s put this into a function 

```javascript
function getPosts(){
fetch('https://jsonplaceholder.typicode.com').then(function(res){
	console.log(res);
})
}
```

Let’s stop and explain this part of the code. As we said, `fetch` returns a Promise, so we can call the `then` function which attaches a handler that will be executed when the promise resolves. This means that when the data comes back from the server, we will execute the function that is given to `then`. 
Conceptually, we can model the Promise as a progress bar. We send a pigeon to our server far away to our server, it gets the message and comes back. When the pigeon comes back, the progress bar is filled and the promise is resolved with a result that the pigeon brings back. When the pigeon is away, the progress bar is not full yet. We say that the Promise is pending.
When the pigeon comes back, the function given to `then` gets executed with the result brought by the pigeon as a parameter. In a nutshell, we give the Promise a function that it needs to execute when it is resolved and we get as parameter the result of the resolution. That's all about`then`. 

Something important to keep in mind is that `then` executes the function we give it and returns a promise which is resolved when the function finished executing. This enables us to chain `then` functions. 
Let’s say we need to get the first post of the series of posts of the user with id equal to 2.

```javascript
function getPosts(){
fetch('https://jsonplaceholder.typicode.com').then(function(res){
	return res.filter(function(user){user.id==2});
}).then(function(usertwoPosts){
	return usertwoPosts[0];
})
}
```

