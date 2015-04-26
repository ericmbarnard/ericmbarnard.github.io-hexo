title: "JavaScript Basics - Native Stuff"
date: 2011-11-01 08:29:23
categories:
    - Development
tags:
    - JavaScript
---

For a while I’ve been wanting to write about some core things that I’ve learned while creating complex JavaScript applications. Not necessarily ‘begginer’ things, but I guess someone could use these as so.

I also build on the .NET platform. Think what you will, but I like it… and I’ve written Ruby (liked it too), and love JavaScript (a dynamic language at that) development.

First off JavaScript does have some core types that should be understood. My attempt to describe them below is for someone coming from a strongly typed world:

| Type      | Description |
| ----      | ------ |
| `Boolean`   | true/false |
| `String`   | character array |
| `Number`    | float (can be used as integer, decimal, etc…) |
| `null`      | null |
| `undefined` | basically non-existent, values can’t be initialized to `undefined` |
| `Object`    | essentially a dictionary/hash table of key values |
| `Array`     | not really a true array, but an object with numerical keys, and native manipulation functions. Think more ‘special object’ than ordered sets in memory |
| `Function`  | objects that can be invoked, but still can exist as singular objects |

So lets address the first big question, what’s the difference between `null` and `undefined`? `null` and `undefined` are very similar, however `undefined` seems to be what most browsers use to indicate something hasn’t been declared or initialized. `null` is something that is used when you perhaps want to initialize an object or property, but don’t want it to have a value.

Ok, so how about this Array vs Object thing? First off, Arrays DO have the following properties/methods:
```javascript
   var arr = [];
   arr.length; // Number
   arr.push    // Function
   arr.shift;  // Function
   arr.unshift; // Function
   arr.splice; // Function
   arr.slice;  // Function
   arr.pop;    // Function
   arr.join;   // Function
   arr.concat; // Function
```
I won’t get into those here, especially when you have [this](http://www.w3schools.com/jsref/jsref_obj_array.asp)

However, depending on the browser that you are using, these are not usually the super-efficient data structures that we’re used to. They are still very useful, I just want you to be informed.

Lastly, what is this `Function` thing?

Functions, like I mentioned, can be invoked and have parameters passed to them (they can do a lot more, but I can’t put it all in one post, right?) Essentially a `Function` is an object that you can create and execute logic inside. You can hang it off of a regular object and make a method (like we’re normally used to) or some other fancy things we’ll get to.

Here’s a decent example of building out a basic object:
```javascript
   var car = {}; //a new object
   car.name = "Corvette"; //a string
   car.maxSpeed = 155; //a number
   car.currentSpeed = null; //initializing to null
   car.honk = function(){ //a function
       alert("Honk");
   };
```
A couple things you may have noticed so far… creating new objects and arrays.
```javascript
    var arr = [];

    //Dont' do
    var arr = new Array();

    var obj = {};

    //Don't do
    var obj = new Object();
```
Use the shorthand and not the “new” keyword for native objects. Its pointless and is extra keysstrokes for your fingers.