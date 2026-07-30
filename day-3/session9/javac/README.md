javac
===

This session was discussing things around javac. Topics covered:

javac is fast
---

It is fast especially compared to compilers for some other languages. But could presumably be improved. In some cases, there are bugs/problems, causing javac for be slow - e.g. during initial lambda implementation, there could be an exponential processing time. Current example would be exhaustiveness checking.

javac implementation language and launcher
---

We discussed that javac is implemented in Java, and that its launcher is basically a simple launcher calling the main javac class. This also works to start javac:

```
$ java com.sun.tools.javac.Main
```

"builder" lambda pattern
---

We discussed a pattern where builders are sent to lambdas. Advantages include that it is not possible to miss the "`build()`" method invocation.

improving javac error messages
---

We discussed that javac messages are sometimes not ideal, and if those could be improved. Possibly, they could be improved by just adding suggestions about the most common causes for the given error, and possibly only for the most common errors.

We discussed the problematic aspect of finding out which problems are most common. Basically getting representative broken code. We discussed possible relation to IDEs.

evaluating changes
---

We discussed what options there are for evaluating impact of changes.
