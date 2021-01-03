---
layout: default
title: 7 Things to Consider Before Modifying Existing Code
---
The thing we do most often as software engineers is change existing software.  The most obvious way to change software is directly modifying the existing code without doing any refactoring.  This can be a good approach in some situations, but the way software is changed has important implications both for the stability of the change we are making and for how easy it will be to make changes in the future.

For this reason, there are several things to consider before directly modifying existing code.  The SOLID design principles form the foundation for these considerations, but this post will approach the topic by focusing on the following questions and bring SOLID design principles into the discussion as needed:

- How many classes will need to change?
- Would the change expand an existing class's responsibilities?
- Does the class being changed already have too many responsibilities?
- Does the code being modified have good tests?
- Is it necessary to preserve old behavior?
- Can the functionality be written as a new implementation of an interface?
- Can the changes be split into multiple pull requests to the main branch?

## How many classes will need to change?
If more than one class needs to change it suggests that one of the following may be true:
- The change being considered should be broken down into smaller changes and each of the items described here should be considered independently for each change
- The change may not fit well within the current design and some refactoring should be considered to first to make a place in the design for it.

If only one class needs to change it suggests that the change being made may not need to be further broken down and that directly modifying the class may be a good approach.  However, there are still many other important things to consider (described below) before making that decision.

## Would the change expand an existing class's responsibilities?
When considering a class's responsibilities it is important to think about the Single Responsibility Principle (the S in SOLID).  The Single Responsibility Principle is often stated as "A class should only have one reason to change."

That definition can be a little difficult to apply.  I didn't really have a good sense for it myself until I read "Clean Architecture" by Robert C. Martin.  In the section on Single Responsibility Principle he clarifies that the principle is about people and says to "Separate the code that different actors depend on".  He uses an example where the same class is used to implement requirements from two different departments (different "actors") within a company.  The class initially does the right thing for both use-cases are, but then the requirements diverge.  If code like this is not separated as those requirements diverge (or up front in anticipation of the requirements diverging), then it will end up being likely that a change in that code due to a change in one department's requirements could break the functionality the other department depends on.

Martin also describes another symptom of violating the Single Responsibility Principle: merge conflicts.  He gives an example of two different developers modifying the same class at the same time due to changes in requirements from two different departments.  The need to merge their changes puts the functionality used by both departments at risk.

We don't need to be talking about different departments within a company either.  I often think of different use-cases as different "actors" even if the same people are involved in both use cases.  If I anticipate those use-cases changing at different times and for different reasons then I will try to separate the code.

When considering implementing new behavior by modifying an existing class, consider what future changes in requirements or use cases would cause the class to need to be modified again.  In particular think about if the list of requirements and use cases that could affect the class would grow longer once you add the new behavior to the class.  If the list would grow longer, it may be worth considering if it would be better to put the new code somewhere else (in a different class or a new class).  Sometimes refactoring will be necessary to allow the code to be put somewhere else.  The effort involved in refactoring is worth it because it leads to the current change and future changes being easier to implement and test without breaking existing functionality and without colliding with work being done by other developers.

## Does the class being changed already have too many responsibilities?
Sometimes modifying an existing class to add new behavior would not expand its responsibilities because it has too many responsibilities already.  In this case it may make sense to refactor the existing class and split out the responsibility related to the new behavior into a new class.

This will make the existing code a little bit better because after refactoring there will be fewer reasons to change the already-too-big class than there were before.  This means that future changes to the already-too-big class will be less likely to break the functionality related to the responsibility that now has its own independent class.

Often when a class is already too big, the unit test coverage of it is lacking or the tests are difficult to understand.  Another benefit of splitting a responsibility out into its own class is that it will be easier to thoroughly unit test the behavior related to that responsibility.

As a part of refactoring, it will likely be necessary to update the tests for the "already-too-big" class as well.  If the existing tests already provide good coverage of the responsibility that is being split out a good process to follow is the following:

1. Determine the interface for the new class.
2. Write the new class.
3. Modify the existing class to accept the interface for the new class as a constructor argument.
4. Update the existing unit tests for the existing class to provide the actual implementation of the new class as the constructor argument to the existing class.
5. Ensure the tests pass.
6. Write unit tests for the new class.
7. Modify the existing tests for the existing class to pass a mock or stub implementation of the interface instead of the actual implementation.
8. Removing tests or assertions that are no longer necessary because the new class's test suite covers them.

The order of some of the steps can be rearranged a bit based on preference, but the goal is to first leverage the current unit tests to prove nothing was broken, but to still end up with two independent classes with two independent test suites in the end.  The goal of initially using the actual implementation of the new class with the existing unit tests, is to temporarily use them as integration tests between the two classes to prove that nothing was broken.  The goal of replacing the actual implementation of the class with a mock or stub (in step 7) and the removal of duplicate testing (in step 8) is so that future modifications to only one of the classes will cause only one the test suites to need changed.

In cases where there is not already good unit test coverage of the existing class it likely makes sense to get some level of test coverage in place before refactoring.  These tests should at least cover the responsibility that is being split out, but it is also good to consider covering related functionality that looks like it could accidentally be broken during the refactor.  The goal here is not to have perfect unit tests for the existing class.  The goal is just to make an incremental improvement to the tests and to be confident in being able to refactor the code without breaking anything.

## Does the code being modified have good tests?
Even if it makes sense to implement new behavior by modifying an existing class (because it clearly fits within the responsibility of the class), it is still a good idea to review the tests before making changes.  If the tests are difficult to understand or do not cover the behavior of the class well, it is a good idea to improve the tests before making changes.

If the new behavior being implemented is to fix a bug in the class, it is a good idea to write a failing test that exposes the bug, before implementing the fix to make the test pass.

## Is it necessary to preserve old behavior?
In some cases when new behavior needs to be implemented it is also necessary to preserve the old behavior.  One example of this is that maybe the software currently only supports one type of persistence technology, but the new version needs to support deployments where a different persistence technology is used.  A concrete example of this would be be an application deployed in AWS that uses DynamoDB that now needs to support a new Azure deployment using CosmosDB.

Another situation where old behavior needs to be preserved is for backwards compatibility, maybe the format of some persisted data is changing, but the software still needs to support reading data persisted in the old format.

A third example is a situation where the new behavior will be slow-rolled using a per-account feature flag.

In all of these situations there is an additional new responsibility being introduced on top of whatever responsibilities are involved in the change itself.  That responsibility is the decision to invoke the old behavior or the new behavior.  It is important to think about what part of the software should be responsible for this decision because it is likely different than the part of the software where the new behavior is implemented.  It also may be the case that the part of the software making the decision will be different from the part of the software acting on the decision.  Making the decision involves taking some inputs and producing an output that indicates which behavior to invoke.  Acting on the decision involves looking at the output of the decision and invoking the correct behavior.  Ideally changes that need to preserve old behavior only introduce one `if` statement to act on the decision.  This usually takes the form of deciding which implementation of an interface to instantiate.

When changes need made that require the old behavior to be preserved it is a good idea to look at the interfaces that already exist in the software and think about if the new behavior can be introduced as a new implementation of one of those interfaces.  The advantage of this is that you can be very confident that the new behavior will not change the old behavior because the code implementing the old behavior will not be touched.

If no interfaces exist that allow the new behavior to be implemented like this, it would be a good idea to consider refactoring the code to introduce such an interface prior to making the changes.

## Can the functionality be written as a new implementation of an interface?
Even in cases where the old behavior does not need preserved long term there can be advantages to adding new behavior by writing a new implementation of an existing interface.  Being able to add new functionality in this way is what the Open-Closed Principle (the O in SOLID) is all about.  The main advantage of this is that when writing new code you can be very confident you do not break existing code because you are not touching it.

Another advantage of this approach is that it can allow you to get feedback from your teammates earlier.  This is because you can write the new implementation and its unit tests and put up a pull request for it without having any production code call the new implementation.  Later a separate pull request can be done to "wire up" the new implementation so that it is actually used.  This can also make it easy to disable the new behavior if something goes wrong.

One thing to consider when taking this approach is if the new implementation would duplicate any code from an existing implementation.  If code will be duplicated, it is important to consider if it should be duplicated.  If the duplicated code is likely to be changed at different times for different reasons and evolve in different directions over time, it may be fine to duplicate it.  If both implementations want the same thing from the duplicated code and will continue to want the same thing in the future, then it may make sense to consider doing a refactor to extract the responsibility that both implementations will now share.  

## Can the changes be split into multiple pull requests to the main branch?
Smaller pull requests are easier to review well, which means it is more likely that issues can be caught before the code gets to production.  Smaller pull requests are also the result of shorter-lived branches and a shorter-lived branch is much less likely to introduce merge conflicts than a longer-lived branch.  Making small pull requests frequently to the main branch can also limit the assumptions that can build up during the development of new functionality.  I am assuming here that the main branch is being deployed frequently to production.  This means these small pull requests I am suggesting are not allowed to break anything.

One example of how a larger change can be broken up into small pull requests to the main branch is if refactoring is being done.  By definition, refactoring is changing the organization of the code without changing the functionality.  This means any refactoring you do can be broken down into small steps that can be put in separate pull requests to the main branch.  By breaking a large refactoring into small steps that go into the main branch you will run into way fewer merge conflicts with changes being made by other developers.

Another example is this: Assume you decided that you need to add three new classes to implement some new behavior.  One class is a new implementation of an existing interface and the other two are new implementations of new interfaces that the first depends on.  Each of these classes can be put in its own pull request.  Pull requests for each of the new classes can be safely merged into the main branch one at a time (along with their unit tests) even though the represent just part of the new functionality.  This is because no production code needs to actually call the new class yet.  Then, once the pull requests for all three the classes have been merged, they can be wired in to the production code to be used.

Another advantage of breaking down work like this is that it may be possible to parallelize the work and have different team members own the implementation of the different classes.

## Conclusion
Although modifying existing code directly is the most obvious way to add new functionality to software, there is a lot to think about before deciding that is the right approach.  Sometimes a direct modification will be the right answer, but often there are advantages to investing some effort into refactoring so that you can be confident that the new behavior works and doesn't break anything else.  This investment also pays off by making future changes easier.

The SOLID design principles are the foundation for the considerations discussed above.  I like to think of the Open-Closed Principle as suggesting that software should be designed with interfaces to keep the functionality decoupled and provide opportunities for implementing new functionality by writing new implementations of existing interfaces.

When it doesn't make sense to implement functionality by writing a new implementation of an existing interface, I like to think of the Single Responsibility Principle as providing a guide for that situation.  If some new behavior falls within the single responsibility of an existing class, then modifying that class is often the right approach.  When the new behavior does not fit in so well, the Single Responsibility Principle can be used to guide the refactoring effort.  I like to think of the remaining SOLID design principles (Liskov Substitution Principle, Interface Segregation Principle and Dependency Inversion Principle) as providing guidance on how to design the interfaces that decouple the software.
