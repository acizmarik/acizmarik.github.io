+++
title = "Error Handling and Exceptions in .NET"
description = "Bla"
date = "2022-11-14"
+++

Over the past few months, I participated multiple times in discussions on how to do error handling in .NET properly. It always felt like the same questions kept arising over and over again. How to signalize errors? How to pass additional information? When to use exceptions and when to use different mechanisms?

<!--more-->

## Errors Classification

As a general rule, it is common to distinguish between **recoverable errors** and **unrecoverable errors**. An example of a recoverable error can be input validation. In most cases, when user provides an invalid number or a date outside of a requested range, we do not need to terminate the entire application. Instead, we can display a validation error and disable subsequent operations until the error is resolved by user. Unrecoverable errors are more serious -- for example, not having enough memory to allocate requested object is under most circumstances an acceptable situation for an almost immediate termination.

## Exceptions

Exceptions are really powerful mechanism that provide structured and uniform information about errors. The ability to capture full stack traces is great -- although certainly not for free considering execution time and consumed memory. Furhermore, in complex codebases, it might not be obvious where exceptions get handled. 

>When an exception occurs, the system searches for the nearest catch clause that can handle the exception, as determined by the run-time type of the exception. First, the current method is searched for a lexically enclosing try statement, and the associated catch clauses of the try statement are considered in order. If that fails, the method that called the current method is searched for a lexically enclosing try statement that encloses the point of the call to the current method. This search continues until a catch clause is found that can handle the current exception, by naming an exception class that is of the same class, or a base class, of the run-time type of the exception being thrown. A catch clause that doesn’t name an exception class can handle any exception.
> 
> *Source: https://github.com/dotnet/csharpstandard/blob/721ef2964efeb5b4b803e856a00094dc4fe21bb7/standard/exceptions.md*

While this is very useful when gracefully terminating programs after an *exceptional* situation, it should not be used to guide normal control flow. I have a couple of reasons for this (aside from the quite obvious performance hit):

* It creates a code that (sometimes) unpredictably jumps around the codebase
* It makes debugging harder because the debugger will break on exceptions that are considered *normal*
* I would argue it is even worse than using `goto` to guide the control flow. At least when we write something like `goto myLabel`, we know that the next instruction is determined by the `myLabel`. Exception handling performs stack unwinding, exceptions might be rethrown, also within an exception we might throw another exception...

{{< figure src="https://imgs.xkcd.com/comics/goto.png" title="Source: xkcd comics" >}}

Sadly, there are many places where people overuse exceptions. Does it create more readable code? Probably not. Is it an easier way as compared to rewriting the code properly? Maybe, but at the end of the day the resulting code is certainly harder to maintain.

## The Standard no-Throwing Solution

From early .NET versions this pattern was introduced to not rely on throwing exceptions for simple cases. For example, we should call `int.TryParse(input, out var result)` in comparison to `int.Parse(input)`, if the `input` can be invalid. Result of the operation, a boolean, is indicated by the return value and if successful, the parsed value is assigned to the `value` variable. This pattern can be commonly found in the BCL as well as in many other projects.

It is an elegant pattern to avoid throwing exceptions. However, it lacks an option to pass additional error information. Of course, we could return maybe a value tuple, for example something like `(bool result, string? errorMessage)` but it is not commonly used, there are other variants of this approach and it does not work well with some nullable attributes. So the question is -- can we do better?

## Monad `Result<T, TError>`

There is a great library called [dotNext](https://github.com/dotnet/dotNext) which extends standard .NET API with various new features. Apart from other great stuff, it introduces `IResultMonad<T, TError>`. For the purposes of error handling, it can be used in the following way:

```csharp
public static Result<double, MathError> Divide(double arg1, double arg2)
{
    if (arg2 == 0)
        return new(MathError.DivideByZeroError);
    else
        return arg1 / arg2;
}

public static void Main(string[] args)
{
    var result1 = Divide(4, 2); // 2
    var result2 = Divide(4, 0); // MathError.DivideByZeroError
    var result3 = result1.OrNull(); // 2
    var result4 = result2.OrNull(); // null
    var result5 = result2.Value; // throws UndefinedResultException
}

public enum MathError { None, DivideByZeroError }
```

{{< rawhtml >}}
</br>
{{< /rawhtml >}}

Each `Result` either has `Value=<double>` or `Error=MathError.DivideByZeroError`. Moreover, there are also different variants -- for example, `Result<T>` that can be constructed either from a value or from an exception (there is no need to use error enums). At the end, even if we wanted to throw an exception, we can do something like `result.OrThrow(e => new Exception(e.ToString()))`. Most importantly, nothing is forcing us to throw, especially if the error is recoverable. Similar monads are common in some other languages, for example [F#'s Result](https://learn.microsoft.com/en-us/dotnet/fsharp/language-reference/results) or [Rust's Result](https://doc.rust-lang.org/std/result/). These languages also tend to support stronger pattern matching techniques as compared to what is at the time

While this solution might not be perfect, it resolves two main important points -- (1) we can write an **exception free error handling** system and (2) we can **pass additional information** (error enum or an exception instance). Therefore, it is certainly a viable option to other error handling contructs.