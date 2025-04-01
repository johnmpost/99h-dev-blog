---
title: "On-The-Job Learning: Unit Tests"
---

One of the things I was tasked with making while interning at Workshop was a wrapper for Canva's new [Connect API](https://www.canva.dev/docs/connect/). We just needed to use 3 endpoints: list designs, create a design export, and get a design export. Notice how there are both creation and query endpoints for design exports. If you want to export a design from Canva's new API, you have to kick off the export with one API call, and then poll the query endpoint until you get a completed result.
I'm going to walk through some of my process in making this API wrapper and extract some insights we can learn from it about unit testing.

## Good unit testing starts with good code organization

Once again, we were using 3 endpoints from Canva's API. But I also wanted to make a convenience wrapper for the process of creating an export job and polling for a completed one. So, in my first pass at the wrapper API, I ended up with 4 functions:

1. List all designs
2. Start a design export
3. Get a design export (which might be in progress or finished)
4. Export a design (which essentially does 2 and 3 together)

I figured that organization made sense. I was debating whether number 4 should be included along with the first 3, but I decided it should because that's functionality we will want. Theoretically, Canva could have structured their API such that number 4 was provided, so it makes sense from the perspective of someone using my API wrapper that number 4 is part of it.

However, I then thought about what it was going to look like to unit test this API wrapper. Obviously, you can't unit test a function that does a side effect of making a network request to a third-party resource. (We'll talk about that more later.) So, I had to think about what I _was_ going to test.

What I realized is that the first three functions are fundamentally different than the fourth. Not in what results they produce, but in how they are implemented. The first three _only_ make a network request to Canva and have a small amount of logic to support that. But the fourth function, the one that essentially wraps 2 and 3 together into a nicer interface, has non-trivial logic that I wrote. And it was non-trivial enough that it was probably worth testing.

Let's glance again at my previous argument: "Theoretically, Canva could have structured their API such that number 4 was provided, therefore I should bundle number 4 with the others." While it may be true that Canva could have done this, the truth is they did not, and that difference actually does matter here. It matters because I can (and maybe should) unit test the code that _I_ write, but I cannot (and shouldn't need to) unit test the code that the _Canva_ team writes.

So, even though Canva could have implemented number 4 themselves and the functions available to us would remain the same, the fact that they did not _should_ change the way we structure our code.

So, I changed how I originally structured my API wrapper. I only included the first 3 functions, those that simply use API endpoints provided by Canva. What that allows me to do is to construct function 4 separately and _specify the API wrapper as an injected dependency_. So now I have an API wrapper that accurately represents an interface to a third-party resource (Canva's API) and I have another function. This other function does some special logic and also happens to _consume_ the interface to the third-party resource and use some of its functions. We can also specify which instance of the interface we provide to it. This includes a real implementation for a production app, but also a mock implementation to make testing easier.

So, by working through this particular problem in my internship, we can extract a piece of knowledge that we can apply to all our code moving forward.

> When creating interfaces to third-party resources, model them as closely as possible. This creates more correct semantics and allows better testability when consuming these interfaces or creating derivative functions.

## Effectful dependencies and mocking

Before, I mentioned that you can't unit test functions that make network requests to third-party resources. Let's talk about what that means.

Obviously, you can't _unit_ test a piece of code that literally does an external network request. (You may be able to test it some other way, which we'll also talk about later.) One option is to consider the function or service you use to make a network request (fetch, axios.get, etc) an explicit _dependency_. This would allow you to mock that dependency. Let's take a look at what that might do for you.

Let's say we have a function that does these things:

1. accepts some parameters
2. constructs a network request based on those parameters
3. sends the request
4. receives a response
5. parses the response and returns it

Our network request dependency is responsible for 3 and 4. So, if we mock that dependency, our mock dependency has two facets, it can do something based on its inputs, and it can return something as a "network response." Let's examine each of those cases.

If we create a mock dependency that actually performs computations based on the inputs to it, we are attempting to mimic the third-party resource. We've created our own business logic (that, frankly, should probably have its own tests) and created a (probably incomplete) copy of the resource that will certainly become out of date. So, we should not do this.

That leaves the other option: when mocking the network request dependency, we just tell it to return a given value (or set of values) regardless of what the input is. So then what does the function look like as it's being tested?

1. accepts some parameters
2. constructs a network request based on those parameters
3. sends the request (which is ignored by our mock)
4. receives a response (which is not based on anything that happens in 1-3)
5. parses the response and returns it

Well, now it's clear that we're only testing part 5. That's somewhat useful, but not very. Let's see if there's another way we can look at this. We can essentially think of functions like this as 3 parts:

1. a pure function of (input parameters) -> (network request)
2. an impure function of (network request) -> (network response)
3. a pure function of (network response) -> (parsed response as final result)

But if this is true implicitly, why don't we just extract out 1 and 3 into their own functions explicitly. If we do that, it becomes very clear _how_ to test our code. We can just test the pure functions we created from 1 and 3 by passing some input and expecting the correct response. This is another piece of general knowledge we can take from this example.

> When writing unit tests, don't mock your dependencies. Simply decompose the function into a set of pure functions that you can test without any mocking. This will grant you the same confidence in your function with much less complexity. In other words: "Only unit test pure functions."

## _Should_ we test these functions? How?

However, we're not done with this example. Let's take a deeper look at those pure functions we extracted from 1 and 3.

Function 1 takes input parameters and constructs a network request. In reality, we aren't _actually_ outputting an http request, but we're creating a good representation of one that our network request dependency can understand. Function 3 takes the response from the network and parses it to a usable result format.

I'm going to make the claim that these two functions are "trivial," and should not be unit tested at all. Let me explain why, and what I mean by "trivial."
There are two things to notice about both of these functions.

First, they are probably pretty simple, not including a lot of complex logic. While I believe that should be a consideration as to whether it's worth writing unit tests for these functions, complexity is not what makes a function trivial.

What makes a function trivial is that the implementation of the function itself is what defines its correct behavior. If you were to write a unit test for this function, you could only test that "the code is doing what the code is doing" and not that "the code is doing the correct thing." Writing this unit test would also basically just include copying the logic of the function. Adding a test does not give us any extra confidence that our code is actually "correct," and now we just have a bunch more tech debt because we have to adjust this test if we ever adjust the code itself. Again, this is because _the function itself is what defines its own correct behavior._ There is no external definition of what it looks like for this function to be correct.

Functions that are not trivial are often algorithmic or complex, and have an external definition of correctness. For example, a function that produces a fibonacci sequence is non-trivial. Before we ever implement the logic of the function, we know what a correct version of the function should produce given any set of parameters.

But our network request functions are not like this. They define themselves and are trivial as a result. We can make a function that constructs a network request from some parameters. But to truly test "does this code do the correct thing?" and not just "does this code do what it is doing?", we would have to actually send that network request to the real third-party service, and see if it works correctly. So, testing this code in isolation provides nothing to us, and only adds more useless code to maintain.

> Don't unit test "trivial" functions. And remember, triviality is not about complexity, but about how correctness is defined.

But we do want to make sure the code we wrote is working how we expect, right? Of course. But remember, unit testing is only one tool available to us. When our functions are good candidates for unit testing, we should unit test them, because unit testing is simple and provides us many assurances. But, for functions that are not good candidates for unit testing, we can do integration tests. Whether manual or automated, integration tests allow us to test whether our trivial functions (which shouldn't be unit tested) actually work correctly when integrated with other systems.

> The right way to test functions that aren't pure and non-trivial is via integration tests.

## Lessons learned

One of the most valuable things I do as a software engineer is work on new problems. Not just to gain experience with particular technologies or patterns, but to critically work through a problem and discover deep, new insights that can be applied more generally. You never know when a task you'll think is simple will have one small aspect that makes you rethink your approach considerably.

By working through this one small part of a project I did at my internship this summer, I was able to solidify my ideas about an important part of software engineering: testing.

I called out the lessons I learned throughout the article, but I'll summarize them here so you can easily reference them:

- Model effectful resource dependencies as closely as possible.
- Only unit test pure functions.
- Only unit test non-trivial functions.
- If unit testing isn't right for a function, integration testing may be.

Happy testing!
