---
layout: post
title:  "JavaScript MVVM vanilla flavour"
author: "Nabil"
categories: Design
---
“Fake it ‘till you make it” doesn’t work when developing in JavaScript. If you want to use any framework in JavaScript, you have to understand how it works. Ok, you don’t HAVE to, but it is preferable to invest time understanding; then you will be able to code more efficiently and grasp the documentation of any new framework quickly.

MVVM is a design pattern, which means that it is a way of designing a software in order to make it more reusable, extendable and maintainable. MVVM stands for Model View ViewModel. As the name describes correctly, it has 3 components, that we will briefly describe before getting our hands dirty.

* *Model.* The model is where our data live. The model is just a placeholder for our data in our application. It doesn’t act on the data, it just hold it.

* *View.* The view is the presentation of our data. It is responsible for controlling the appearance of the data.

* *ViewModel.* The ViewModel is the glue between the view and the model. It manages the binding between these two entities.

I know you are not convinced yet, and it’s perfectly fine. I prefer to describe software with code. In order to get your head around MVVM, we will work on an example with a tiny project.

The idea of the project is the following: we have a model with two entries: a name and a lastname. We want the UI and the model to always be in sync at any point in time. Here is a diagram (personally handwritten, please!):

![Image to explain mvvm]({{ "/assets/1_8PlGdX-dtssXfWDn3FEntA.png" }})
