+++
title = "IL Emitting and Verification for .NET"
description = "Approaches on how to verify generated or rewritten IL"
date = "2022-07-26"
+++

Nowadays, we have a couple of options when it comes to the area of code generation. Output of tools, such as Roslyn and Source Generators, that generate source code is easily verifiable -- we can compile it and see if it succeeds. However, generating IL can be a bit harder. Moreover, verifying the generated IL is more complicated, especially if it was generated and emitted during runtime.

<!--more-->

## What is Verifiable Code?

According to the specification (see: [ECMA 335, 6th edition, June 2012, II.3 Validation and verification](https://www.ecma-international.org/publications-and-standards/standards/ecma-335/)), in order for IL to be verifiable, it must be both **valid (memory safe)** and **type safe**. These terms should pretty self-explanatory but in the next section we can see some examples for them.

## Why is Verifiability Important?

Let's assume that we emitted the following method body:

{{< highlight csharp "linenos=table,hl_lines=1,linenostart=1" >}}
IL_0000: pop // Remove top element from evaluation stack <-- invalid operation (empty stack)
IL_0001: ret // Return control flow to the caller
{{< / highlight >}}</br>

Once JIT compiler tries to compile the method above, an `InvalidProgramException` gets thrown. Since the first instruction of the method is `pop`, we are attempting to remove an element from an empty evaluation stack. Therefore, if executed this would result in a stack underflow - which is by no means valid, nor memory safe. This exception generally reveals the signature of the violating method, but not the exact reason (stack underflow on offset *IL_0000*). Regardless, it at least narrows the problem down to a single method.

That was not that bad right? It seems like we can trust JIT with identifying invalid (memory unsafe) IL constructs and preventing them from being executed. Now, let's assume a bit more complex example where we will take a look at JIT identifying type unsafe IL:

{{< highlight csharp "linenos=table,hl_lines=2,linenostart=1" >}}
IL_0000: newobj    instance void [System.Private.CoreLib]System.Object::.ctor() // Create new instance of type "Object" and push it onto stack
IL_0005: call      void [System.Console]System.Console::WriteLine(string) // Method call <-- invalid operation (types mismatch)
IL_000A: ret
{{< / highlight >}}</br>

JIT compiler would not have any issue with compiling the method above. However, we can clearly see that the code is not type safe. On the first line we create a new instance of `object` and then call the `System.Console::WriteLine(string)` overload. Therefore, we are passing a reference to an `object` instead of a reference to a `string` as expected by the called method. What's more interesting is that the code even executes and (probably) does not even throw any exceptions.

Since JIT does not perform type safety checks, we could potentially execute some pretty unsafe IL. What is problematic is that the issue might not get noticed immediately. We could, for example break some internal structures - a problem that would get picked up by the runtime at some point later or not at all. These issues, once identified, tend to end with an `ExecutionEngineException`. This exception does not tell much apart from the fact that, at some point of the execution, something went horribly wrong.

## ILVerify Tool

From early .NET versions, there has been a tool called PEVerify. The purpose of the tool was to verify both metadata and IL within dotnet assemblies. However, it has certain flaws or missing features -- mainly the fact that it was meant only for .NET Framework and could not verify the core library (mscorlib). These issues were resolved by an introduction of the tool ILVerify.

ILVerify can be installed as a dotnet global tool. Just to quickly showcase its capabilities, let's try to verify the methods presented in the previous section. We should see a similar result to the following output:

{{< highlight txt "linenos=table,linenostart=1" >}}
[IL]: Error [StackUnderflow]: [<path-to-assembly.dll> : P::M1()][offset 0x00000000] Stack underflow.
[IL]: Error [StackUnexpected]: [<path-to-assembly.dll> : P::M2()][offset 0x00000005][found ref 'object'] Unexpected type on the stack.
{{< / highlight >}}</br>

We can see from the output above that ILVerify was able to identify both problems without any issues. Therefore, if we have the modified assembly available, we can just give it to ILVerify and check whether it contains any unverifiable IL. ILVerify does not need to be as fast as JIT compiler -- it can spend more time verifying the code (at the end of the day it is its sole purpose). As a result, we can easily identify also violations of the type safety. Therefore, it is a good idea to to use ILVerify or PEVerify whenever generating or rewriting IL.

## What about Runtime IL Emitting?

What are the options when we do not have the modified assembly available? After all, we could use `System.Reflection.Emit` or the Profiling API to emit or even rewrite existing IL during runtime. Therefore, it would be great if we could save the assembly from memory to a file and analyze it offline. This would effectively transform this problem into the previous one, which as we already know can be pretty effectively solved using tools like ILVerify. However, this is exactly the point where it gets a bit complicated.

### Missing Support in CoreCLR to Dump Assemblies

It turns out it is actually not supported since the introduction on .NET Core. And to this day also .NET 5, nor .NET 6 support this. Furthermore, this feature was actually available in .NET Framework and many people would like to see it available in newer frameworks too. Luckily, it seems that it could be finally available in .NET 8 and there is already a prototype implementation (see: [github.com/dotnet/runtime/issues/15704](https://github.com/dotnet/runtime/issues/15704#issuecomment-1175353695)). But the questions remains -- what can we do in the meantime?

### (Partial) Workarounds

In the following subsections, I will try to outline two possible workarounds that could be used until a proper support is implemented by dotnet team. While the following solutions are definitely not perfect, I consider them a good help when investigating mysterious bugs in connection with wrong IL.

#### Option 1: Manual IL Inspections

This approach is mainly usable if you are debugging an `InvalidProgramException`. If we want to inspect IL of a specific method, we can try the following approach to retrieve it from memory:

* Create a memory dump of the analyzed process ```dotnet-dump collect <pid>```
* Analyze the dump using either `dotnet-dump analyze` or something like WinDbg
* SOS commands to retrieve IL of a method:
   * Find declaring type `!name2ee *!MyNamespace.MyType` and get the method table address
   * Dump method table `!dumpMT -md <address>`
   * Find the analyzed method and get its address
   * Dump method body `!dumpIL <address>`

After that we can inspect the IL and corresponding method metadata for any issues. Generally, it is a good idea to try and write generated code first in C#, then compile it and after that try to recreate same IL as Roslyn generated.

#### Option 2: Usage of Native CoreCLR API

While working on a custom profiler, I discovered an interesting discussion: [github.com/dotnet/runtime/issues/37389](https://github.com/dotnet/runtime/issues/37389#issuecomment-511847798). In one of the comments, there is a described way on how to dump an assembly. Even though I was not able to successfully implement the mentioned steps (yet), I feel like it is a nice solution to obtain assemblies and be able to feed them to ILVerify. Hopefully, someone will be able to implement it and share a working snippet :).