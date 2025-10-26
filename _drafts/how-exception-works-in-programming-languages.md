## C++

Questions

1. How stack unwind works?
2. How state is cleaned up especially destructors?
3. Exception hiarchy? How is it used?
4. Does it complicate multi-threading?
5. What is the performance impact?
6. The implementation detail of `__cxa_allocate_exception`.
7. When you throw an exception, what you are really throwing?
8. How is `_Unwind_Exception.private_2` and private_1 used ?
9. Difference between Itanium C++ ABI, ARM EHABI and Windows SEH (Structured
   Exception Handling)?

Exception is allocated in heap.

```
Low Address ───────────────────────────────────────────> High Address

+-------------------------------------+------------------------+
| __cxa_exception (runtime metadata)  | user object (MyError)  |
+-------------------------------------+------------------------+
^                                     ^
|                                     |
|                                     +-- pointer returned to compiler (__cxa_exception*)
|
+-- header, found at a negative offset
```

C++ runtime hands control to the platform’s unwind library
