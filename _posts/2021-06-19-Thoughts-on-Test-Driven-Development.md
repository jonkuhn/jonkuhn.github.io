---
layout: default
title: Thoughts on Test Driven Development
---
I have done a little Test-Driven Development (TDD) in the past, but recently I have been doing it more consistently.  I have been wanting to write up some blog posts about Unit Testing in general so I recently created a repository on Github showing an example of TDD in C#.  I plan to use this example as the subject of a series of blog posts about Test-Driven Development and Unit Testing.  This post focuses on some thoughts I have had recently about TDD.  Future posts in this series will likely focus on particular unit testing techniques and best practices.

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
One idea for doing TDD at a high level, would be to write a high-level integration test for some feature before fully designing or implementing it.  My current thought is that this approach would mostly be useful for looking for gaps in the requirements by thinking about them in a different way, but that it would not help with true TDD.  A high-level test is not something we are going to be able to make pass by just writing a small amount of code, so it doesn’t really fit the definition of TDD.  A high-level test written up front would need to sit around for a while, until the full implementation is done before it can be run with the expectation that it would pass.  I definitely think higher level tests should be written, but I am not sure there is a lot of value in trying to write them first.

The best thought I have for applying TDD with a less of a design up front would be to do TDD to write the unit tests for the high level business logic for some feature first.  Then, after that has taken shape, to use TDD to implement the input/output details that the business logic ended up requiring.  I don’t think I have tried this technique, so perhaps when I do it would be a good topic for a future blog post.

One situation where I have written higher level tests first is while fixing bugs.  If only one part of the code is responsible for a bug, I will write a unit test.  However, if a bug is caused by the interaction of multiple parts of code, I will write a higher level test first.  Being able to reproduce the bug with any kind of automated test is very useful for verifying the fix and preventing regressions in the future.

## Is TDD good?  Why?
I always like to know the benefits (and costs) for different techniques and best practices so that I can explain to others why they should consider following them.  So, what are the benefits of TDD?

I think one benefit is test coverage.  While doing TDD if you really only write the minimal code to pass a test you know that there is some test that requires all of the implementation code to be there.  Even if you don’t use TDD all the time, doing it as an exercise can help you learn to be able to think about the relationship between unit tests and implementation code in terms of “What test made me write this code?”.  I find that this question helps me evaluate test coverage.

I think another benefit is to see the limits of unit testing.  When you write your first test you can often come up with some silly hardcoded implementation to make it pass.  At least for me, this usually gets me thinking a bit more deeply about what my test is really asserting and how I need to add more tests or extend the existing test in order to assert the right things.  It is important to limit how silly you allow your implementation to be though, because things can get out of hand (more on that later).  I find that drawing the line between what is too silly to do in an implementation gets me thinking about what unit tests can protect against and what they can't.

I think one additional benefit may be related to the observation I made earlier:  It is difficult to follow TDD before having a properly decomposed design described as small tasks.  I think that if trying to follow TDD is difficult for a task it may be a good indication that more high level design and tasking work needs to happen first.  This is and idea I want to keep in mind and see how well it applies to my own work going forward.

## Is TDD a learning exercise or an everyday approach?
For the reasons stated above, I think at minimum it is useful as a learning exercise.  However, I think it can be beneficial as an everyday approach.  I intend to try to apply TDD as much as I can to learn more about it and find out where it works well for me and where it may not.

# Practical Tips for TDD
Next I'd like to talk about a couple of practical tips for doing TDD.  The first tip involves how to get started with your first test and the second tip is about how to think about writing the simplest implementation to make a test pass.

## The first test you write is not set in stone
When doing TDD, the first test is often the hardest test to write.  My advice is to make it super simple and build on it.  The first test you write is not likely to survive unmodified into your final test suite.  This is okay.  TDD is an iterative process.  Once I get through a couple iterations of the write test, implement, repeat loop I find that starts to be easy to know what to do next.  The trick is just to get started.

For my TDD example repo I wrote the business logic for checking out a book from a library that has the following requirements:
- If checking out another book would put the member over the maximum number of checked out books, do not allow the book to be checked out.
- If the member has past due books, do not allow the book to be checked out.
- Record a copy of the book as checked out by the member.
- If no copy of the book is still available, the member cannot check out the book.
- If a copy of the book was successfully checked out, call out to a reminder service that will remind the member when the book is due.

With a list of requirements like that, it may be difficult to know where to start, but the important part is just to start with something.  The test I started with was this:
 
 ```c#
[Test]
public void TestCheckoutBookAsync_GivenMemberOverBookLimit_ThrowsTooManyCheckedOutBooksException()
{
    const string memberId = "test-member-id";
    const string isbn = "test-isbn";

    var bookLoanRepository = Substitute.For<IBookLoanRepository>();
    var bookLibrary = new BookLibrary(bookLoanRepository);

    // TODO: This test doesn't actually do what it says yet because it
    // doesn't actually set up the situation where the member is over
    // the book limit.   However, this is still a good time to try to
    // run it and get the classes built and compiling.

    Assert.ThrowsAsync<TooManyCheckedOutBooksException>(async () =>
        await bookLibrary.CheckoutBookAsync(memberId, isbn));
}
 ```

As you can see, this test isn’t even complete.  However, it gets the process started.  I wrote this test even before I created any of the interfaces or classes involved.  So to make this test compile and pass I created the classes and interfaces and hardcoded the `CheckoutBookAsync` method to throw.

The next iteration of this test looked like this:

```c#
[Test]
public void TestCheckoutBookAsync_GivenMemberOverBookLimit_ThrowsTooManyCheckedOutBooksException()
{
    const string memberId = "test-member-id";
    const string isbnOfBookToCheckOut = "test-isbn";

    var bookLoanRepository = Substitute.For<IBookLoanRepository>();

    var stubOutstandingLoans = new List<BookLoan>();

    // Stub GetOutstandingBookLoansForMember to return exactly
    // the maximum number of outstanding BookLoans
    for (var i = 0; i < BookLibrary.MaxOutstandingLoans; i++)
    {
        stubOutstandingLoans.Add(new BookLoan());
    }
    bookLoanRepository.GetOutstandingBookLoansForMemberAsync(memberId)
        .Returns(stubOutstandingLoans);

    var bookLibrary = new BookLibrary(bookLoanRepository);

    Assert.ThrowsAsync<TooManyCheckedOutBooksException>(async () =>
        await bookLibrary.CheckoutBookAsync(memberId, isbnOfBookToCheckOut));
}
```

As you can see, the test above adds the setup of the situation where too many books are already checked out by a member by stubbing out the response to the `GetOutstandingBookLoansForMemberAsync` method on the repository.

The final version of this test in my test repository looked like this:

```c#
[TestCase("memberId1", BookLibrary.MaxOutstandingLoans)]
[TestCase("memberId2", BookLibrary.MaxOutstandingLoans)]
public void TestCheckoutBookAsync_GivenMemberOverBookLimit_ThrowsTooManyCheckedOutBooksException(
    string memberId, int outstandingBookLoanCount)
{
    var futureDueDate = DateTime.UtcNow + TimeSpan.FromDays(1);

    var bookLoanRepository = Substitute.For<IBookLoanRepository>();
    SetupOutstandingBookLoansForMember(
        bookLoanRepository,
        memberId,
        CreateOutstandingBookLoans(memberId, outstandingBookLoanCount, futureDueDate));

    var bookLibrary = new BookLibrary(
        bookLoanRepository, Substitute.For<IBookLoanReminderService>());

    Assert.ThrowsAsync<TooManyCheckedOutBooksException>(async () =>
        await bookLibrary.CheckoutBookAsync(memberId, "test-isbn"));
}
```

You can see that as I worked through my example and wrote more tests, I updated this test to use some helper methods and also added a couple of `TestCase` attributes so it would be run with different arguments.  This is how I do TDD.  I start with something very simple and then enhance it.  I often build my next test by copying and pasting a previous test.  Then, after I have a number of tests implemented, I look for opportunities to extract helper functions that are carefully designed to keep each test readable by itself.  (Techniques for keeping tests readable while extracting test helpers is likely to be the topic of a future blog post.)

## Should I write the simplest code to make a test pass?
When doing TDD, it is often said to write the simplest code to make a test pass.  This can sometimes mean unconditionally returning a hardcoded value from a function because the tests do not yet require more.  This may seem silly, but I think that it is beneficial to do a little bit of this, because it can tell you a lot about your test coverage.

Let's take another look at one of the example tests from the previous section:

```c#
[Test]
public void TestCheckoutBookAsync_GivenMemberOverBookLimit_ThrowsTooManyCheckedOutBooksException()
{
    const string memberId = "test-member-id";
    const string isbnOfBookToCheckOut = "test-isbn";

    var bookLoanRepository = Substitute.For<IBookLoanRepository>();

    var stubOutstandingLoans = new List<BookLoan>();

    // Stub GetOutstandingBookLoansForMember to return exactly
    // the maximum number of outstanding BookLoans
    for (var i = 0; i < BookLibrary.MaxOutstandingLoans; i++)
    {
        stubOutstandingLoans.Add(new BookLoan());
    }
    bookLoanRepository.GetOutstandingBookLoansForMemberAsync(memberId)
        .Returns(stubOutstandingLoans);

    var bookLibrary = new BookLibrary(bookLoanRepository);

    Assert.ThrowsAsync<TooManyCheckedOutBooksException>(async () =>
        await bookLibrary.CheckoutBookAsync(memberId, isbnOfBookToCheckOut));
}
```

The code I'd initially write to make this pass is the following:
```c#
public async Task CheckoutBookAsync(string memberId, string isbn)
{
    await Task.CompletedTask;
    throw new TooManyCheckedOutBooksException();
}
```

Note for folks not familiar with NSubstitute: I am using NSubstitute in my examples and it will not fail the test if the setup `GetOutstandingBookLoansForMemberAsync` call is not received.  NSubstitute goes beyond the mocks simply being non-strict (ignoring calls that are not set up).  It will also not fail tests if a call that _was_ set up is not received.  This is because it is designed to be used with tests that follow the Arrange/Act/Assert pattern.  It only fails tests if it doesn't receive calls that you explicitly ask it to verify using `Received()` checks.  I like this approach because I think it makes it easier to write tests that are not brittle.  However, a deeper discussion of the pros and cons of this approach is yet another idea for a future blog post.  

I think it was useful for me to make the test pass with this "silly" hardcoded throw, because it demonstrates that my test suite is not yet proving anything about the exception ever _not_ being thrown.  This helps lead me to the next test to write.  TDD sessions generally get into a loop like this where it is obvious what test to write next.

Next, I may write a test like this:

```c#
public void TestCheckoutBookAsync_GivenMemberNotOverBookLimit_Succeeds()
{
    const string memberId = "test-member-id2";
    const string isbnOfBookToCheckOut = "test-isbn";

    var bookLoanRepository = Substitute.For<IBookLoanRepository>();
    bookLoanRepository.GetOutstandingBookLoansForMemberAsync(memberId)
         .Returns(new BookLoan[] {})

    var bookLibrary = new BookLibrary(bookLoanRepository);

    Assert.DoesNotThrowAsync(async () =>
        await bookLibrary.CheckoutBookAsync(memberId, isbnOfBookToCheckOut));
}
```

Now, it is certainly possible to take "silly" implementation code too far.  Since I made the memberId different in the second test I could technically pass the test using the following code.  However, I think it is clear that this is going too far:

```c#
public async Task CheckoutBookAsync(string memberId, string isbn)
{
    await Task.CompletedTask;
    if (memberId == "test-member-id")
    {
        throw new TooManyCheckedOutBooksException();
    }
}
```

I think a good guideline is to only do “unconditionally silly” implementations.  By this I mean to not wrap any silly hardcoded implementations inside of if/else conditionals that are not necessary for implementing the requirements.  The above code violates this rule.  Another way I sometimes think about it is that the unit tests are doing their job if they make incorrect implementations so ugly they would easily raise flags in code review.

However, I'd like to give one more example where doing something silly in the implementation demonstrates something interesting about the tests.  I could also write the following implementation code to pass the tests.  This code is wrong because of the hardcoded `"test-member-id"`, but the tests will pass.  Doing this does actually demonstrate something interesting about the test suite:  The test coverage around the argument being passed to `GetOutstandingBookLoansForMemberAsync` could still be better.

(If you are wondering why the tests pass even though the second test sets up the mock for `"test-member-id2"` and not `"test-member-id"`, read on.  I explain this later.)

```c#
public async Task CheckoutBookAsync(string memberId, string isbn)
{
    var numberOfOutstandingLoans =
        (await _bookLoanRepository.GetOutstandingBookLoansForMemberAsync("test-member-id"))
        .Count();

    if ((numberOfOutstandingLoans + 1) > MaxOutstandingLoans)
    {
        throw new TooManyCheckedOutBooksException();
    }
}
```

I think you could make a very valid case that the above code would not pass code review, but I think it demonstrates how "unconditionally silly" implementations tend to tell you something interesting about your test coverage.

One way to make the tests fail for this implementation is to add `TestCase` attributes to our tests to run them with multiple different `memberId` values.  When doing this we need to think carefully about the default behavior of the mocks.  If NSubstitute mocks receive a call that has not been set up (for example if the arguments don't match the set up) they automatically return mocks when possible.  This means the `IEnumerable<BookLoan>` result from `GetOutstandingBookLoansForMemberAsync` will be an automatically generated mock.  The behavior of an automatically generated mock for an `IEnumerable<>` is such that `Count()` will return 0.

This means that adding `TestCase` attributes that run the test with different memberId values to the "happy path" test case (where no exception is expected) will not make the tests fail for the hardcoded implementation given above.  This is because even if our mock setup will return an empty list for a given value of the memberId argument, that behavior is no different from the default behavior of the mock with no set up.  If you were reading along carefully above you may have wondered why the "happy path" test case would pass in the first place with the argument hardcoded to `"test-member-id"` because the test itself uses `test-member-id2`.  This is why.

The test that _can_ be made to fail by running it with multiple `memberId` values is the test that asserts that the exception is thrown.  If that test sets up the mock to return too many outstanding book loans when called with `"another-member-id"`, the test will fail with the hardcoded implementation that passes `"test-member-id"`.  This is because that call will get the default mock behavior (which is effectively an empty list of book loans) and the exception will not be thrown which will make the test fail.  Of course, even without trying multiple values for memberId, we could make the test fail by just making the memberId in the test different from the hardcoded one.  However, if the test doesn't try two different values the hardcoded value could just be changed to match the test.  If the test is run for two different values the implementation can no longer be "unconditionally silly" and still pass the test.

The two tests we end up with are as follows:

```c#
[TestCase("test-member-id")]
[TestCase("another-member-id")]
public void TestCheckoutBookAsync_GivenMemberOverBookLimit_ThrowsTooManyCheckedOutBooksException(string memberId)
{
    const string isbnOfBookToCheckOut = "test-isbn";

    var bookLoanRepository = Substitute.For<IBookLoanRepository>();

    var stubOutstandingLoans = new List<BookLoan>();

    // Stub GetOutstandingBookLoansForMember to return exactly
    // the maximum number of outstanding BookLoans
    for (var i = 0; i < BookLibrary.MaxOutstandingLoans; i++)
    {
        stubOutstandingLoans.Add(new BookLoan());
    }
    bookLoanRepository.GetOutstandingBookLoansForMemberAsync(memberId)
        .Returns(stubOutstandingLoans);

    var bookLibrary = new BookLibrary(bookLoanRepository);

    Assert.ThrowsAsync<TooManyCheckedOutBooksException>(async () =>
        await bookLibrary.CheckoutBookAsync(memberId, isbnOfBookToCheckOut));
}

[Test]
public void TestCheckoutBookAsync_GivenMemberNotOverBookLimit_Succeeds()
{
    const string memberId = "test-member-id";
    const string isbnOfBookToCheckOut = "test-isbn";

    var bookLoanRepository = Substitute.For<IBookLoanRepository>();

    bookLoanRepository.GetOutstandingBookLoansForMemberAsync(memberId)
         .Returns(new BookLoan[] {});

    var bookLibrary = new BookLibrary(bookLoanRepository);

    Assert.DoesNotThrowAsync(async () =>
        await bookLibrary.CheckoutBookAsync(memberId, isbnOfBookToCheckOut));
}
```

To pass these tests without just passing along `memberId`, the implementation would need to be pretty obnoxious.  Something like the following, which is clearly going too far:

```c#
public async Task CheckoutBookAsync(string memberId, string isbn)
{
    int numberOfOutstandingLoans = 0;

    // Ha Ha, behave correctly only if it is a test! -- NO! This has gone too far!
    if (memberId == "test-member-id" || memberId == "test-member-id2")
    {
        numberOfOutstandingLoans = 
            (await _bookLoanRepository.GetOutstandingBookLoansForMemberAsync(memberId))
            .Count();
    }

    if ((numberOfOutstandingLoans + 1) > MaxOutstandingLoans)
    {
        throw new TooManyCheckedOutBooksException();
    }
}
```

So far I like this "unconditionally silly" guideline that I came up with.  I plan to continue to use it and see how it works for me.  If I discover something that works better, perhaps I will write another blog post about it.

## Conclusion
I hope that you found these thoughts on TDD useful.  I am looking forward to sharing more of my thoughts about Unit Testing in the future.  If you are looking to read more about Unit Testing I highly recommend Roy Osherov’s book “The Art Of Unit Testing”.  This book helped shape my thinking on the subject.
