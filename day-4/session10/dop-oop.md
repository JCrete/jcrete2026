# Is Data Oriented Programming (DOP) the New OOP?

Pattern matching is an alternative to polymorphism. Consider a functional list (like in Scala or Lisp). It is either empty or nonempty. In the latter case, with an element and a tail—another list. In Java, it would look like this:

```
sealed interface List permits EmptyList, NonEmptyList {}
enum EmptyList implements List { INSTANCE; }
record NonEmptyList(Object element, List tail) implements List {}
```
Huh? No methods? That's where pattern matching comes in:
```
static int length(List l) {
    return switch (l) {
        case EmptyList _ -> 0;
        case NonEmptyList(var _, var tail) -> 1 + length(tail);
    };
}
```
In OO, you would use a polymorphic method instead:
```
sealed interface List permits EmptyList, NonEmptyList {
    int length();
}
enum EmptyList implements List {
    INSTANCE;
    int length() { return 0; }

}
record NonEmptyList(Object element, List tail) implements List {
    int length() { return 1 + tail.length(); }
}
```

What is better? For an _open-ended hierarchy_, the OO approach wins. Imagine adding another class to the hierarchy. In the list case, a plausible candidate is a fixed-size list backed by an array. With pattern matching, you would have to locate every pattern and add a new case. That clearly doesn't scale. Polymorphism is the right choice in that situation.

But with a _sealed hierarchy_, you know all implementations. Consider adding new functionality. With OO, you have to add a method to every implementing class—if you control them, which is a big if. With pattern matching, you just write a static method and match the known cases.

![](https://horstmann.com/unblog/2026-08-10/oop-vs-dop.webp)

Are sealed hierarchies common? There is JSON, of course. Chris Kiehl's [book on data-oriented programming](https://www.manning.com/books/data-oriented-programming-in-java) has intriguing examples. But perhaps not enough to dethrone OOP and replace it with DOP.

TL;DR: If it's sealed, go forth and match patterns.

But stay away from the weird parts of pattern matching! Java `switch` is a ~~mess~~mélange of old and new features that don't interact well with each other. I could give an entire presentation on that. ([And I have.](https://horstmann.com/presentations/2025/jcrete/)) 

 Here is a [particularly troubling example](https://mail.openjdk.org/archives/list/amber-dev@openjdk.org/message/SZ7YYU4RBVZL6MKBUVP3VK3QU7BY4LM5/). 
