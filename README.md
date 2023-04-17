# Asynchronous JavaScript

## Learning Goals

- Establish an analogy for synchronous versus asynchronous work.
- Describe a synchronous code block.
- Describe an asynchronous code block.
- Compare and constrast the call stack and task queue.
- Define the event loop.

## Introduction

Thus far, when writing our code, we write it in the exact order we want things
to happen. If a later operation depends on two numbers being multiplied, we make
sure the multiplication happens first.

```js
const product = 6 * 13; // => 78
const halfProduct = product / 2; // => 39
```

In a lot of cases, this is exactly what we want. It follows what we would
logically expect to happen. One thing happens, _then_ once that's finished, the
next thing happens, _then_ another thing, and so on. Sometimes, however, this
isn't the behavior we want to happen.

This especially becomes a problem when we start to program applications for the
web. Browsers have to manage a lot. They're animating a `gif`, they're
displaying text, they're listening for clicks and scrolls, they're streaming a
SoundCloud demo in a background tab, _and_ they're running JavaScript programs.

To do all that work efficiently, browsers use an _asynchronous_ execution model.
That's a fancy way of saying "they do little bits of lots of tasks until the
tasks are done."

In this lesson we'll describe, compare, and contrast the synchronous and
asynchronous execution models of JavaScript.

## Establish an Analogy for Synchronous Versus Asynchronous Work

Let's imagine a chef in a kitchen preparing a big meal. There's only one chef in
this kitchen. The chef could prepare a turkey, then prepare some potatoes, then
prepare some bread, then prepare green beans, and then serve it.

Our diners would be treated to cold turkey, cold bread, cold green beans, and
cold potatoes! This is not the goal. This meal was prepared in a _synchronous_
model: one-thing-after-the-other. Whatever happened "blocked" the rest of things
that were waiting for work.

Instead, our chef should move between each of these tasks quickly. The chef
should use the _asynchronous_ execution model browsers use. They should stuff
the turkey, they should measure the ingredients for the bread, they should peel
the potatoes, etc. in a loop, _as fast as possible_ so that all the tasks _seem_
to be advancing at the same time. If the chef were to adopt this _asynchronous_
model of work, the diners would be treated to piping-hot turkey, steaming
potatoes, soft warm bread, and fresh warm green beans.

![synch/asynch
diagram](https://curriculum-content.s3.amazonaws.com/fewpjs/fewpjs-asynchrony/Image_42_AsynchronyIllustrations.png)

## Review: How JavaScript Executes

Before we can discuss how to do the same sort of juggling in our code, let's
review how the JavaScript engine actually executes code.

When running a JavaScript file, the engine will first look for any function
_declarations_ and store those in memory. Next, it will go back to the top of
the file and run the code in written order. If it comes across a variable, it
will store it in memory. If it comes across a function _call_, it gets added to
the **call stack** and is resolved before moving on.

<details><summary><b>Review: What is the call stack?</b></summary>
  <p>The call stack is how the JavaScript engine keeps track of multiple function calls. It follows the LIFO (Last In, First Out) principle, meaning the function <i>last</i> added to the call stack is the <i>first</i> to be returned.</p>
</details>

Remember, the LIFO principle applies to function calls _within_ function calls.
For example:

```js
function parentFn() {
  childFn();
  return "Parent";
}

function childFn() {
  return "Child";
}

parentFn();
```

The call stack would look like:

- `childFn()`
- `parentFn()`

Thus, the output would look like:

```bash
"Child"
"Parent"
```

However, if we called each function _separately_, they would not be part of the
same call stack.

```js
function parentFn() {
  return "Parent";
}

function childFn() {
  return "Child";
}

parentFn();
childFn();
```

The JavaScript engine would come across the `parentFn()` call first, whose call
stack would simply be:

- `parentFn()`

That call stack would resolve, then the engine would move on to the `childFn()`
call, whose call stack would similarly be:

- `childFn()`

> **Note**: Try pasting these examples into this [call stack visualizer] and
> step through the code to see the difference in action.

Understanding the call stack and how JavaScript normally runs is key to
understanding asynchronosity as the call stack is where the difference lies. We
will get into that difference soon! Before that, let's define synchronous code.

## Describe a Synchronous Code Block

The examples we wrote above are _synchronous_. In fact, in our JavaScript
journey so far, we've mostly written _synchronous_ code.

With synchronous behavior, each line of code has to fully resolve before the
engine will move on to the next line. So, for any function calls the engine
comes across, each function's call stack has to fully resolve before the
remaining code can run.

This is fine in some cases, but imagine if we had a loop that has to iterate
through thousands of lines of data. That would take some time.

For example, try running the following snippet:

```js
console.log("I'm the first thing that happens");

for (let i = 0; i <= 1500000000; i++) {
  if (i == 1500000000) {
    console.log("The loop is done!");
  }
}

console.log("Finally, it's my turn");
```

Notice how the console hangs for a little bit between the first log and the
last. Because our loop is so large, it takes a second or so to resolve. In this
case, it can be considered a "blocking" operation. All the code beneath the loop
has to wait before they can be executed. Our "blocking" operation _blocks_ the
rest of the code from running for an amount of time.

In arbitrary cases like our above example, you might think i's not too
detrimental. However, soon we will want to work with data from external sources.
Imagine we had a synchronous function called `synchronousFetch("URL STRING")`
that fetches data from some external source.

```js
const tooMuchData = synchronousFetch("http://genome.example.com/..."); // Line 1
console.log(tooMuchData); // Line 2
```

That work in Line 1 could take a long time (e.g. slow network), or might fail
(e.g. failed login), or might retrieve a **_huge_** amount of data (e.g. The
Human Genome).

With this synchronous approach, JavaScript won't continue to the next line of
code until `synchronousFetch` has finished executing, so it's possible that the
`console.log()` in Line 2 _will never execute_! Furthermore, while JavaScript is
executing `synchronousFetch` it will not be able to animate gifs, you won't be
able to open a new tab, it will stop streaming SoundCloud, it will appear
"locked up." Recall our chef metaphor: while the chef prepares the potatoes, the
green beans grow cold and the turkey congeals. Gross.

## Describe an Asynchronous Code Block

With asynchronous code, we can make those long operations _not_ block the main
thread of execution. With asynchronous code, we're essentially telling
JavaScript:

> Hey, do this thing. While you're waiting for that to finish, go do whatever
> maintenance you need: animate that gif, play some audio from SoundCloud,
> whatever. But when that first thing has an "I'm done" event, go **back** to it
> and _then_ do some work that I defined in a function when I called it.

How do we do this? Usually by way of **callback functions** provided by the [Web
API]. For example:

```js
function asyncLog() {
  console.log("I've been logged asynchronously!");
}

console.log("Start");

setTimeout(asyncLog);

console.log("End");
```

Assuming `setTimeout()` will _call back_ the `asyncLog()` function, what would
you usually expect the output to look like?

<details><summary>Click to see the answer</summary>
<p>
  If the above example was run <i>synchronously</i>, you might expect the output to be:
<pre>
Start
I've been logged asynchronously!
End
</pre>
</p>
</details>
<br/>

However, that is not the case. `setTimeout` is an _asynchronous function_. Let's
break down the series of events:

1. The `asyncLog()` function is saved in memory.
1. The `"Start"` string is logged to the console.
1. The asynchronous `setTimeout()` function is _started_ but not completely
   resolved.
1. The JavaScript engine continues through the code.
1. The `"End"` string is logged to the console.
1. The `setTimeout()` function resolves, and "I've been logged asynchronously!"
   is logged to the console.

<details><summary>Click to see the actual output</summary>
<p>
<pre>
Start
End
I've been logged asynchronously!
</pre>
</p>
</details>
<br/>

Most asynchronous functions in JavaScript have this quality of "being passed a
callback function." It's a helpful tool for spotting asynchronous code "in the
wild". We will learn more about the `setTimeout()` function and the Web API in
the next lesson.

For now, let's focus on _why_ this occurs. Why are asynchronous functions like
`setTimeout()` treated differently from any other function?

## The Task Queue and Event Loop

When the JavaScript engine comes across an asynchronous function, it does
**not** get immediately added to the call stack. Instead it gets added to
something called the **task queue**. Wait, two different queues? How does the
JavaScript engine juggle between them?

The **call stack** takes priority. Anything in the call stack gets resolved
first. Then, when it's empty, the engine will look into the **task queue** and
take the _oldest_ task there and push it into the call stack to be resolved.
This happens in a _loop_ until everything has been resolved.

This loop is called the **event loop**. Whenever there's an asynchronous task in
the task queue, the event loop is what checks the call stack for when it becomes
empty.

This separation of queues is how asynchronous functions are treated differently.
With the task queue, tasks that are not meant to occur immediately can wait
until it's their turn without blocking the rest of the code.

> **Note**: The task queue is sometimes known as the **callback queue**.
> Additionally, the task queue follows the FIFO principle - First In, First Out.

## Conclusion

Executing code in order is fine in a lot of cases. Sometimes, however, we need
to write code that we expect to take some time, or specifically want to delay.
When we don't want the rest of our code to wait for this delayed bit to finish,
we have to make it _asynchronous_.

JavaScript in the browser has an asynchronous execution model. This fact has
little impact when you're writing simple code, but when you start doing work
that might block the browser you'll need to leverage asynchronous functions. In
the following lesson, we will learn more about the browser environment and how
the previously mentioned [web API] allows us to write asynchronous code.

## Resources

- [Introducing Asynchronous JavaScript](https://developer.mozilla.org/en-US/docs/Learn/JavaScript/Asynchronous/Introducing)
- [Call Stack Visualizer]

[call stack visualizer]: (https://www.jsv9000.app/)
[web api]: (https://developer.mozilla.org/en-US/docs/Web/API)
