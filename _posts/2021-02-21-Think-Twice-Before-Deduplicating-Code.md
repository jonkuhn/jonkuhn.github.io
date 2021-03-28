---
layout: default
title: Think Twice Before Deduplicating Code
---
As software engineers when we see duplicate code we tend to immediately have an impulse to de-duplicate the code by extracting a function and then calling that function from all the places where the code was duplicated.  This is often a good idea, but there are important trade-offs to consider.  Removing code duplication can sometimes lead to more problems in the future if the trade offs are not carefully considered and if the responsibility of the shared code is not clearly communicated.

## Different Kinds of Code Duplication
The key thing to consider when deciding if code duplication should really be removed is the Single Responsibility Principle (the S in SOLID).  A few years ago I read Robert C. Martin's book Clean Architecture.  This book played a very important role in shaping how I think about designing software.  In the section on Single Responsibility Principle he discusses the different kinds of code duplication in terms of "accidental duplication" versus "true duplication".

"True duplication" is when code is duplicated and we know that every time one of the duplicate instances of the code is changed, all the other instances will need to be changed in the same way as well.  In these situations our impulse to remove code duplication is correct and beneficial.  It is beneficial because in the future rather than needing to remember to update every instance of the code, the change will only need to be made in one place.  This removes the risk of introducing bugs by forgetting to update one of the duplicates.

"Accidental duplication" is when code is duplicated now, but over time the different instances of the code will evolve in different directions.  This tends to happen when different use-cases need the same thing right now, but the requirements for those use-cases will change at different times and for different reasons in the future.  It can be beneficial to leave this code duplicated.  In the future, when requirements change for *only one* of these use cases we need to modify *only one* of the duplicates.   We don't need to worry about breaking the other (unrelated) use case.  If the code for both use-cases called the same function, we would need to pause and understand what both use cases were looking for from the function and then decide how to proceed.

In effect, when you decide to share code or duplicate code you are deciding what you want to happen "by default" in the future as the code evolves:
- When you share code everything that uses the shared code receives updates to the code by default.
- When you duplicate code then the duplicate instances diverge by default.

## Code Duplication Example
As an example, suppose we have an e-commerce site with a function that returns the top ten best selling products according to revenue over the past month.  The initial use-case for this logic may be an internal report about the products generating the most revenue.  Suppose we later add a "popular products" section to the site.  Initially we want this to also return the top ten best selling products according to revenue over the past month.  What happens if the implementation for the new "popular products" section ends up calling the same function that is used for the internal report?  Initially everything works fine.

However, what happens if the people responsible for the "popular products" section decide that it should now show the top 10 products by units sold?  If the code is shared with the internal report and we don't realize it, we may just modify the shared code to implement this new requirement.  However, by doing this we would also break the internal report that we were not asked to change.  If the code was not shared, we would not risk breaking the internal report.

## Communicating Intentions Through Code
In this example, I don't think it is necessarily wrong for the internal report and the initial implementation of the "popular products" page to share code at some level.  If code is shared, I think what is important is communicating what the shared code is responsible for.

One way to communicate the responsibility of shared code is to think carefully about the name given to that shared code.  Let's consider the simplest case of shared code: a single shared function.  If a function is named very clearly based on what the caller can expect it to do it makes it easier for future developers (including ourselves) working on the code to understand the responsibility of the function.  This means that the function is less likely to be modified in ways that would violate caller's expectations.

In our example, the internal report existed first, and then we implemented a "popular products" page that initially used the same logic.  Lets consider a few different names that the function for the logic of the internal report could have had:
- `BestSellingProducts`
- `TopProductsByRevenue`
- `SalesReportTopProductsByRevenue`

Of these, I think `BestSellingProducts` is the most vague.  Let's assume the initial implementation of the "popular products" page called this function directly.  My concern with a vague name like `BestSellingProducts` is that when we go to update the "popular products" page there is a significant risk that we would edit that function in place to return the top selling products by units sold instead of by revenue.  This would break the internal report.

`TopProductsByRevenue` is not named according to how it is used, but it very clearly states what the caller can expect.  To me, it communicates the idea of "Call this function from wherever you want, but it is always going to return the top products by revenue".  I would expect it's parameters would allow the caller to specify the time range over which to determine the top products by revenue.   Let's assume the initial implementation of the "popular products" page called this function directly.  With a very clear name like this, when the requirements for the "popular products" page changes, it is unlikely that we would modify this function to return the top selling products by units sold instead of by revenue.  Doing so would be in direct conflict with the name, therefore I think:
- The developer implementing the new requirement is unlikely to modify the function to do something that doesn't agree with the name.
- If they were to modify the function, they would likely update the name as well.  Updating the name would break compilation for other use-cases which would at least draw attention to the fact that it was used elsewhere.
- Even if the developer implementing the change modified the function directly and put up a pull request, I think with a clear name, reviewers are more likely to question the change.

`SalesReportTopProductsByRevenue` is the most verbose of the names and it clearly communicates what the function does and what it is for.  If we were to call that from the initial implementation for the new "popular products" page I think it would be very clear that we were "borrowing" that functionality from the internal report, but that the internal report owned it.  "Borrowing" the functionality in this way would be risky because the public facing "popular products" page could change unexpectedly when the internal report's requirements change.  Therefore, I think it is unlikely that the initial implementation of the "popular products" page would call a function like `SalesReportTopProductsByRevenue`.  What is likely to happen instead is one of two things:
- Duplicating the code in a `PopularProductsByRevenue` function.
- Extracting a shared `TopProductsByRevenue` function and calling it from `SalesReportTopProductsByRevenue` and from the code for the "popular products" page.

Of course, names alone are not the only tool for communicating how a function is intended to be used.  The class, namespace and project the function is in can help to clearly communicate its intent as well.  The function `SalesReportTopProductsByRevenue` could actually be a `TopProductsByRevenue` method in a `SalesReport` class.

Using appropriate access modifiers is important as well.  If the `TopProductsByRevenue` method is `private` to the `SalesReport` class, there is no risk of it being directly called by some unrelated code like the "popular products" page.  Of course the access modifier can be changed, but changing access modifiers to be more permissive should raise flags in the mind of the developer considering such a change as well as in the minds of reviewers of the code.

Another important way to defend against changes to one feature from breaking another feature is to write good unit tests as well as higher level functional and integration tests.  However, if the code clearly communicates the intentions potential problems can be caught even before running the tests.

## Summary
While removing code duplication is often a good decision, there are times when it can actually cause problems.  There are always trade-offs to consider:
- When you share code everything that uses the shared code receives updates to the code by default.
- When you duplicate code then the duplicate instances diverge by default.

Whether you make the decision to share code or leave it duplicated, it is important to think carefully about how you communicate your intentions through the code.  A well-named function can make a big difference in how well future developers (including ourselves) understand the code.  This level of understanding can be the difference between quickly implementing a new feature without breaking anything and struggling to implement a new feature while causing regressions.

## Further Reading
If you found this post interesting and want to learn more about software architecture and design, I highly recommend Robert C. Martin's book "Clean Architecture".  This post is inspired by the ideas about duplication discussed in chapter 7 ("SRP: The Single Responsibility Principle") and also the discussion of duplication on page 154 within chapter 16 ("Independence").

If you found this post interesting you may also like my post about [7 Things To Consider Before Modifying Existing Code]({% post_url 2021-01-03-7-Things-To-Consider-Before-Modifying-Code %}).
