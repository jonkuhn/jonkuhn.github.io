---
layout: default
title: TDD Part 2 - Writing The First Test
---
This is the second of three posts discussing my thoughts on getting started with TDD.  In this post I will discuss what I think is one of the most difficult parts of getting started with TDD: writing the first test.

Posts in this series:
- [TDD Part 1 - When To Use Test Driven Development]({% post_url 2021-06-18-TDD-Part-1-When-To-Use-Test-Driven-Development %})
- TDD Part 2 - Writing The First Test (you are here)
- [TDD Part 3 - Writing The Simplest Code To Pass The Test]({% post_url 2021-06-20-TDD-Part-3-Writing-The-Simplest-Code-To-Pass-The-Test %})

## The first test you write is not set in stone
When doing TDD, the first test is often the hardest test to write.  My advice is to make it super simple and build on it.  The first test you write is not likely to survive unmodified into your final test suite.  This is okay.  TDD is an iterative process.  Once I get through a couple iterations of the write test, implement, repeat loop I find that starts to be easy to know what to do next.  The trick is just to get started.

I recently created a [TDD Example repo](https://github.com/jonkuhn/TddExample) on Github to draw examples from for my blog posts on TDD and Unit testing.

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

As you can see, this test isn’t even complete.  However, it gets the process started.  I wrote this test even before I created any of the interfaces or classes involved.  So to make this test compile I created the classes and interfaces.  Then I hardcoded the `CheckoutBookAsync` method to throw, like this:

```c#
public async Task CheckoutBookAsync(string memberId, string isbn)
{
    await Task.CompletedTask;
    throw new TooManyCheckedOutBooksException();
}
```

It may seems silly to write such a simple implementation, but as I will discuss in the [next post in this series]({% post_url 2021-06-20-TDD-Part-3-Writing-The-Simplest-Code-To-Pass-The-Test %}), I think there are important benefits do doing this.

In the next iteration of this test, I began writing the test setup code related to how the business logic should determine if too many books are checked out.  The next iteration looked like this:

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

As you can see, the test above adds the setup of the situation where too many books are already checked out by a member by stubbing out the response to the `GetOutstandingBookLoansForMemberAsync` method on the repository.  Given the way NSubstitute mocks work, this test will still pass if the implementation unconditionally throws.  How to make the tests fail for such an implementation is the subject of the [next post in this series]({% post_url 2021-06-20-TDD-Part-3-Writing-The-Simplest-Code-To-Pass-The-Test %}).  For now the point is just to see how slowly a test case can be built up when getting started with TDD.

The final version of this test in my TDD example repository looked like this:

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

You can see that as I worked through my example and wrote more tests, I updated this test to use some helper methods and also added a couple of `TestCase` attributes so it would be run with different arguments.  This is how I do TDD.  I start with something very simple and then enhance it.  I often build my next test by copying and pasting a previous test.  Then, after I have a number of tests implemented, I look for opportunities to extract helper functions that are carefully designed to keep each test readable by itself.  (Techniques for keeping tests readable while extracting test helpers is likely to be the topic of a future blog post at some point.)

## Further Reading
If you liked this post, please check out the other posts in this 3 part series:
- [TDD Part 1 - When To Use Test Driven Development]({% post_url 2021-06-18-TDD-Part-1-When-To-Use-Test-Driven-Development %})
- TDD Part 2 - Writing The First Test (you are here)
- [TDD Part 3 - Writing The Simplest Code To Pass The Test]({% post_url 2021-06-20-TDD-Part-3-Writing-The-Simplest-Code-To-Pass-The-Test %})

If you are looking to read more about Unit Testing I highly recommend Roy Osherov’s book “The Art Of Unit Testing”.  This book helped shape my thinking on the subject.
