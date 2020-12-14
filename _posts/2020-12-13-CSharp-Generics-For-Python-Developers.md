---
layout: default
title: C# Generics For Python Developers
---
One of the more difficult features to understand in statically typed languages like Java and C# is Generics.  This is especially true for folks whose experience is mostly with dynamically typed languages like Javascript, Python, Ruby and PHP.  The first time you encounter generics is almost certainly an instance of a generic collection like `List<string>`.

In this post I'll start with an example of how dynamically typed languages handle collections.  Then, I'll show how some collections in statically typed languages are very similar.  And finally, I will show how Generic collections can help catch mistakes at compile-time instead of run-time.

## Collections in Python
In Python if we want a list of strings it looks like this:
```python
myStrings = ["a", "b", "c"]
```

If we want a list of objects of some user-defined type it looks like this:
```python
class Contact:
    def __init__(self, name, phone):
        self.Name = name
        self.Phone = phone
    def __repr__(self):
        return "Contact("+self.Name+", "+self.Phone+")"

contactList = [
    Contact("Alice", "111-555-1111"),
    Contact("Bob", "222-555-2222"),
    Contact("Eve", "333-555-3333")
]
```

All the items in each of the lists above happen to contain the same type of item, and often that is what you want.  However, Python itself does nothing to enforce this.  It is perfectly happy for you to do this:
```python
contactList = [
    Contact("Alice", "111-555-1111"),
    ("Bob", "222-555-2222"),
    Contact("Eve", "333-555-3333"),
    42,
    False
]
```

## ArrayList in C#
C# 1 didn't have Generics, but of course it still had collections.  One example is `ArrayList` (Note: the example does several other things that are not valid C# 1).  Using `ArrayList` looks a lot like using a Python list:
```c#
var contactList = new ArrayList
{
    new Contact("Alice", "111-555-1111"),
    ("Bob", "222-555-2222"),
    new Contact("Eve", "333-555-3333"),
    42,
    false
};

class Contact
{
    public string Name { get; }
    public string Phone { get; }
    public Contact(string name, string phone)
    {
        Name = name;
        Phone = phone;
    }
    public override string ToString()
    {
        return $"Contact({Name}, {Phone})";
    }
}
```

*Wait, that works?  I thought C# was statically typed?*

It *does* work and it *is* statically typed.  `ArrayList` is a collection of `object` instances.  We are passing in `Contact` instances, but *every* `class` in C# implicitly inherits from `object`.  And if a class inherits from a base class, it can be used anywhere the base class can be used.  A `Contact` *is* an `object`.

As a side note, `42` and `false` are not classes but they get implicitly "boxed" into instances of `System.Int32` and `System.Boolean` respectively which *are* classes and therefore *are* `object`s.

## Trying to access `Contact` properties
When we access the members of the collection, they are all just `object` instances, so we *only* have access to the members of the `object` class (at least without casting).

So this compiles and runs:
```c#
foreach (var contact in contactList)
{
    Console.WriteLine($"{contact.ToString()} is type {contact.GetType()}");
}
```

But this fails to compile:
```c#
foreach (var contact in contactList)
{
    // Compile Error: object has no Name property
    Console.WriteLine($"{contact.Name}");
}
```

Why? Because `object` does not have a property `Name`, and the `contact` variable in the loop is statically typed as an `object`.

As another side note, just because I used `var` instead of explicitly writing the type name doesn't change the fact that the variable is statically typed.  `var` just means we don't have to write out the type in scenarios where the compiler can infer it from the code.

## A similar error in Python
The equivalent loop in Python will also fail, but at run-time.  It will actually not fail until after the 1st item in the list has already been printed successfully.
```python
for contact in contactList:
    # Run-time Exception: Contact instance has no attribute Name
    print(contact.Name)
```

## Compile-time errors versus Run-time errors
The difference between failing at compile-time and run-time may not initially seem like a big deal, especially with this trivial example.  However, in a larger project, it makes a big difference.  If this list is intended to be a list of only contacts, there may be an obscure piece of code somewhere that *sometimes* puts the wrong type of value into the list.  With only run-time checks, problems like this can be very difficult to track down.  The exception isn't thrown until it is accessed, and that may occur long after the offending item was placed there.  This delay can make it difficult to track down the buggy code that put the item there.

In C#, this mistake gets caught immediately before the code can even run.  If you are using a good IDE the problem will get caught as you type it.

However, with `ArrayList` the compiler errors are generated when we try to treat the items we pull *out* of the collection as `Contact` instances, *not* when we try to put a different type *in*.  This makes sense because we know `ArrayList` holds `object` items, but how can we prevent items other than `Contact`s from ever getting *in* to the list?

We will get to that soon, but first let's answer another question: We know we put `Contact` items *in* so how do we get them *out*?

## Getting our `Contact`s back
The only way to get access to the `Contact` properties of the items we get out of `ArrayList` is to cast them.  Here is an example:
```c#
foreach (var obj in contactList)
{
    var contact = (Contact)obj;
    Console.WriteLine($"{contact.Name}");
}   
```
In this example, if the actual run-time type of the item is *not* a `Contact`, we will get an `InvalidCastException` thrown at run-time.  For our example list, the exception will be thrown when the first non-`Contact` item is encountered.  

This is *exactly* the same behavior as the Python list.  We are really not getting much benefit from static typing!

So, there are two main problems with `ArrayList`:
1. There is no enforcement that the objects we put *in* are of a particular type.
2. When we pull the items *out* of the collection we have to cast them to get back the type we put in.  This means catching errors at run-time, not compile-time.

## A strongly typed ArrayList wrapper
If we want compile-time enforcement that an ArrayList only contains items of a particular type we can write a wrapper class.  A simple example would be:
```c#
class ListOfContacts
{
    private ArrayList _list;
    public ListOfContacts()
    {
        _list = new ArrayList();
    }
    public void Add(Contact contact) => _list.Add(contact);
    public Contact this[int i] => (Contact)_list[i];
    public int Count => _list.Count;
}
```
The cast is still there, but since the only way to add items to the list is through the `Add` method that only accepts `Contact` items, we can be confident that the cast will always succeed at runtime.

If we use that class to encapsulate our `ArrayList` then the compiler can prevent the wrong type of item from being inserted:
```c#
var contactList = new ListOfContacts();
contactList.Add(new Contact("Alice", "111-555-1111")); // works
contactList.Add(("Bob", "222-555-2222")); // Won't compile
contactList.Add(new Contact("Eve", "333-555-3333")); // works
contactList.Add(42); // Won't compile
contactList.Add(false); // Won't compile
```

In the following example, when we loop over the wrapper we are able to access the `Name` property because the indexer returns a `Contact`.
```c#
for (var i = 0; i < contactList.Count; i++)
{
    Console.WriteLine($"{contactList[i].Name}");
}
```

(Note that the fact that this loop does not use `foreach` is not important, that is just a side-effect of me keeping the example class simple by not implementing `IEnumerable<T>`)

This gives you a type-safe collection, but imagine writing (and testing) that boilerplate code for every different type you wanted a collection for.  I also used the modern C# 7 `=>` ("expression bodied member") syntax to make that more concise, so it would have been even worse back in the C# 1 days!

## Generics To The Rescue!
Essentially, generics make the C# compiler and the Common Language Runtime (CLR) work together to generate all that boilerplate code that was necessary for the type-safe solution so you don't have to!  Of course the code it generates will be more efficient and it doesn't really wrap `ArrayList`.  Also, note that when I say it "generates code" I mean that it is generated on the fly, it is not code that you will see as C# source code in your project.

Using a generic list looks like this:
```c#
var contactList = new List<Contact>();
contactList.Add(new Contact("Alice", "111-555-1111")); // works
contactList.Add(("Bob", "222-555-2222")); // Won't compile
contactList.Add(new Contact("Eve", "333-555-3333")); // works
contactList.Add(42); // Won't compile
contactList.Add(false); // Won't compile
```

And you can `foreach` over it because it implements `IEnumerable<T>`:
```c#
foreach (var contact in contactList)
{
    Console.WriteLine($"{contact.Name}");
}
```

## Implementing your own generic collection
In order to demonstrate what a generic implementation looks like, let's see what our ArrayList wrapper would look like with Generics.  Of course, you wouldn't really use this code (because you'd just use `List<T>`), but this is a good example of what a generic class looks like.
```c#
class ExampleList<T>
{
    private ArrayList _list;
    public ExampleList()
    {
        _list = new ArrayList();
    }
    public void Add(T item) => _list.Add(item);
    public T this[int i] => (T)_list[i];
    public int Count => _list.Count;
}
```

Since the class is generic, we put the type parameter on the class name.  In this case we named the type parameter `T`.  You can technically name it anything you like, but it is idiomatic to give type parameters names that begin with `T`.  You can see that everywhere the type `Contact` appeared in the original is replaced with `T`.  The actual type that gets "plugged in" for all those `T`s is determined when you create an instance of the generic type and supply the type parameter.

## Summary
- Writing type safe code allows errors to be caught earlier in the development cycle at compile time instead of runtime.
- Non-generic collections in C# (`ArrayList`) can behave very much like collections in Python.
- Non-generic type safe collections are possible to write, but doing so requires a lot of boilerplate code.
- Generics make it simple to write type-safe code that stores objects in collections without writing repetitive boilerplate code.

## Further Reading
Classes are not the only thing that can be generic either, interfaces, methods a and delegates can be generic as well.  Discussing these is beyond the scope of this post, but if you want to learn more about this or generics in general, I'd recommend reading through the [Generics section](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/generics/generic-type-parameters) in Microsoft's C# Programming Guide.
