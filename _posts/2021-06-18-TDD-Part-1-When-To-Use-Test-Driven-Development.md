---
layout: default
title: TDD Part 1 - When To Use Test Driven Development
---
This is the first of three posts discussing my thoughts on getting started with TDD.  In this post I will discuss what TDD is, as well as when and why I think it is useful.

Posts in this series:
- TDD Part 1 - When To Use Test Driven Development (you are here)
- [TDD Part 2 - Writing The First Test]({% post_url 2021-06-19-TDD-Part-2-Writing-The-First-Test %}).
- [TDD Part 3 - Writing The Simplest Code To Pass The Test]({% post_url 2021-06-20-TDD-Part-3-Writing-The-Simplest-Code-To-Pass-The-Test %})

## What is Test-Driven Development (TDD)?
You may already be familiar with the concept of Test-Driven Development (TDD), but in case you aren’t I wanted to give a quick summary of the process.  The traditional way of developing software is to write the production code and then to test it after the fact.  Test-Driven Development aims to flip that approach where tests are written first and then the implementation is written.

However, this does _not_ mean that _all_ the tests are written first and then _all_ the implementation is written.  TDD means following these steps:
1. Write a simple test that fails.
2. Write the simplest implementation code you can to make that test pass.
3. Repeat

## TDD is best applied after you know the design
I find it difficult to apply TDD if I have not already decided the “shape” of what I am building.  Since unit tests are written to cover individual classes or functions, it is difficult to write them until you have decided how the feature you are building will be split into classes and functions.

Sometimes I will start to “sketch” what I am about to build by writing bits and pieces of the code in a non-TDD way just to make sure I am not missing something in the design.  The goal of this is not to build a fully functional prototype, but just to figure out how a feature fits into the existing design.  The result to aim for in this activity is a good enough understanding of the work required that you can express it as a list of small tasks that will take at most half a work day to implement.  Once the design is at that level, there is a very good chance that TDD can be used on each of those tasks.

## Can TDD be applied at a higher level than unit tests?
One idea for doing TDD at a high level, would be to write a high-level integration test for some feature before fully designing or implementing it.  My current thought is that this approach would mostly be useful for looking for gaps in the requirements by thinking about them in a different way, but that it would not help with true TDD.  A high-level test is not something we are going to be able to make pass by just writing a small amount of code, so it doesn’t really fit the definition of TDD.  A high-level test written up front would need to sit around for a while (until the full implementation is done) before it can be run with the expectation that it would pass.  I definitely think higher level tests should be written, but I am not sure there is a lot of value in trying to write them first.

The best thought I have for applying TDD with a less of a design up front would be to do TDD to write the unit tests for the high level business logic for some feature first.  Then, after that has taken shape, to use TDD to implement the input/output details that the business logic ended up requiring.  I don’t think I have tried this technique, so perhaps when I do it would be a good topic for a future blog post.

One situation where I have written higher level tests first is while fixing bugs.  If only one part of the code is responsible for a bug, I will write a unit test.  However, if a bug is caused by the interaction of multiple parts of code, I will write a higher level test first.  Being able to reproduce the bug with any kind of automated test is very useful for verifying the fix and preventing regressions in the future.

## Is TDD good?  Why?
I always like to know the benefits (and costs) for different techniques and best practices so that I can explain to others why they should consider following them.  So, what are the benefits of TDD?

I think one benefit is test coverage.  While doing TDD if you really only write the minimal code to pass a test you know that there is some test that requires all of the implementation code to be there.  Even if you don’t use TDD all the time, doing it as an exercise can help you learn to be able to think about the relationship between unit tests and implementation code in terms of “What test made me write this code?”.  I find that this question helps me evaluate test coverage.

I think another benefit is to see the limits of unit testing.  When you write your first test you can often come up with some silly hardcoded implementation to make it pass.  At least for me, this usually gets me thinking a bit more deeply about what my test is really asserting and how I need to add more tests or extend the existing test in order to assert the right things.  It is important to limit how silly you allow your implementation to be though, because things can get out of hand.  I find that drawing the line between what is too silly to do in an implementation gets me thinking about what unit tests can protect against and what they can't.

I think one additional benefit may be related to the observation I made earlier:  It is difficult to follow TDD before having a properly decomposed design described as small tasks.  I think that if trying to follow TDD is difficult for a task it may be a good indication that more high level design and tasking work needs to happen first.  This is an idea I want to keep in mind and see how well it applies to my own work going forward.

## Is TDD a learning exercise or an everyday approach?
For the reasons stated above, I think at minimum it is useful as a learning exercise.  However, I think it can be beneficial as an everyday approach.  I have done a little Test-Driven Development (TDD) in the past, but recently I have been doing it more consistently.  I intend to try to apply TDD as much as I can to learn more about it and find out where it works well for me and where it may not.

## Further Reading
If you liked this post, please check out the other posts in this 3 part series:
- TDD Part 1 - When To Use Test Driven Development (you are here)
- [TDD Part 2 - Writing The First Test]({% post_url 2021-06-19-TDD-Part-2-Writing-The-First-Test %})
- [TDD Part 3 - Writing The Simplest Code To Pass The Test]({% post_url 2021-06-20-TDD-Part-3-Writing-The-Simplest-Code-To-Pass-The-Test %})

If you are looking to read more about Unit Testing I highly recommend Roy Osherove’s book “The Art Of Unit Testing”.  This book helped shape my thinking on the subject.
