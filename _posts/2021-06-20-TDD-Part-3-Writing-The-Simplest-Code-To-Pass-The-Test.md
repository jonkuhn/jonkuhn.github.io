---
layout: default
title: TDD Part 3 - Writing The Simplest Code To Pass The Test
---
This is the third of three posts discussing my thoughts on getting started with TDD.  In this post I will discuss how writing very simple, even "silly", implementations to make a test pass can help you learn about your test coverage and decide what test to write next. 

Posts in this series:
- [TDD Part 1 - When To Use Test Driven Development]({% post_url 2021-06-18-TDD-Part-1-When-To-Use-Test-Driven-Development %})
- [TDD Part 2 - Writing The First Test]({% post_url 2021-06-19-TDD-Part-2-Writing-The-First-Test %}).
- TDD Part 3 - Writing The Simplest Code To Pass The Test (you are here)

## Should I write the simplest code to make a test pass?
When doing TDD, it is often said to write the simplest code to make a test pass.  This can sometimes mean unconditionally returning a hardcoded value from a function because the tests do not yet require more.  This seems silly at first, but as I have done more TDD I have found that it is beneficial to do some of these "silly" implementations because it can tell you a lot about your test coverage.  Of course, it is possible to take these "silly" implementations too far.  In this post I will discuss where I think the line is between a "silly" implementation that tells you something about your test, and one that takes it too far.

Let's take another look at one of the example tests from the [previous post in this series]({% post_url 2021-06-19-TDD-Part-2-Writing-The-First-Test %}):

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

The code I initially wrote to make this pass is the following:
```c#
public async Task CheckoutBookAsync(string memberId, string isbn)
{
    await Task.CompletedTask;
    throw new TooManyCheckedOutBooksException();
}
```

**Note for folks not familiar with NSubstitute:** I am using NSubstitute in my examples and it will not fail the test if the setup `GetOutstandingBookLoansForMemberAsync` call is not received.  NSubstitute goes beyond the mocks simply being non-strict by ignoring calls that are not set up.  It will also not fail tests if a call that _was_ set up is not received.  This is because it is designed to be used with tests that follow the Arrange/Act/Assert pattern.  It only fails tests if it doesn't receive calls that you explicitly ask it to verify using `Received()` checks.  I like this approach because I think it makes it easier to write tests that are not brittle.  However, a deeper discussion of the pros and cons of this approach is yet another idea for a future blog post.  

I think it was useful for me to make the test pass with this "silly" hardcoded throw, because it demonstrates that my test suite is not yet proving anything about the exception ever _not_ being thrown.  This helps lead me to the next test to write.  TDD sessions generally get into a loop like this where it is obvious what test to write next.

Next, I would write a test like this:

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

Now, it is certainly possible to take "silly" implementation code too far to make this pass.  Since I made the memberId different in the second test I could technically pass the test using the following code.  However, I think it is clear that this is going too far:

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

## Guidelines for "silly" implementations
I think a good guideline is to only do “unconditionally silly” implementations.  By this I mean to not wrap any silly hardcoded implementations inside of if/else conditionals that are not necessary for implementing the requirements.  The above code violates this rule.  Another way I sometimes think about it is that the unit tests are doing their job if they make incorrect implementations so ugly they would easily raise flags in code review.

## Another interesting lesson from a silly implementation
I'd like to give one more example where doing something silly in the implementation demonstrates something interesting about the tests.  I could also write the following implementation code to pass the tests.  This code is wrong because of the hardcoded `"test-member-id"`, but the tests will pass.  Doing this does actually demonstrate something interesting about the test suite:  The test coverage around the argument being passed to `GetOutstandingBookLoansForMemberAsync` could still be better.

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

So far, I like this "unconditionally silly" guideline.  I plan to continue to use it and see how it works for me.  If I discover something that works better, perhaps I will write another blog post about it.

## Further Reading
If you liked this post, please check out the other posts in this 3 part series:
- [TDD Part 1 - When To Use Test Driven Development]({% post_url 2021-06-18-TDD-Part-1-When-To-Use-Test-Driven-Development %})
- [TDD Part 2 - Writing The First Test]({% post_url 2021-06-19-TDD-Part-2-Writing-The-First-Test %}).
- TDD Part 3 - Writing The Simplest Code To Pass The Test (you are here)

If you are looking to read more about Unit Testing I highly recommend Roy Osherov’s book “The Art Of Unit Testing”.  This book helped shape my thinking on the subject.
