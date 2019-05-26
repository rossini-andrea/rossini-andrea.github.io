---
layout: post
title: Parallel processing in C# with `Vector<T>`
snippet: Modern CPUs can perform multiple operations at a time, even in single threaded applications...
---

Modern CPUs can perform multiple operations at a time, even in single threaded applications. This is achieved using SIMD instructions. This post was inspired by a [series](https://www.youtube.com/watch?v=Pc8DfEyAxzg) on YouTube about C++; at a certain point I asked myself: "and what about .NET?"

## What is SIMD?

SIMD stands for Single Instruction, Multiple Data. Let's give it a brief explanation: SIMD is an instruction set which allows your CPU to perform faster array operations, even in a single threaded application. It works performing the desired operations simultaneously on multiple elements of an array. SIMD instructions run at the same speed of single element operations, accelerating loops on large data batches.

SIMD needs special support from a compiler, today I want to see how C# approaches the problem.

## Let's code

First, let's create a new console project, and open it in Visual Studio Code. At the time of writing I am using **.Net Core 3** which is still in preview.

```txt
PS K:\devel\C#> dotnet new console -o Rand0081.SimdTutorial
PS K:\devel\C#> cd .\Rand0081.SimdTutorial\
PS K:\devel\C#\Rand0081.SimdTutorial> code .
```

We want to keep the example simple, so we will create an app to fill an array with the elementwise sum of two similar arrays.

```C#
static void Main(string[] args)
{
    const int testsize = 45;
    var a = Enumerable.Range(0, testsize).ToArray();
    var b = Enumerable.Range(0, testsize).ToArray();
    var c = new int[testsize];

    for (int i = 0; i < testsize; ++i)
    {
        c[i] = a[i] + b[i];
    }

    Console.WriteLine($"c[21] = {c[21]}");
}
```

This code is straightforward; it creates 3 arrays, and loops on all elements a pair at once, and sums it to store the result in the third array, an element at once. Let's try it!

```txt
PS K:\devel\C#\Rand0081.SimdTutorial> dotnet build .\Rand0081.SimdTutorial.csproj

PS K:\devel\C#\Rand0081.SimdTutorial> bin\Debug\netcoreapp3.0\Rand0081.SimdTutorial.exe
c[21] = 42
```

This program is so simple that it already runs at light speed. But, can we make it run *faster*? The answer is yes! As we already know we can leverage SIMD instructions and perform multiple sums at a time.

## Introducing `Vector<T>`

`Vector<T>` (from now, simply vector) is a structure which holds a fixed length series of elements of the specified type **T**. This generic is geared towards primitive numeric types and allows to benefit of hardware acceleration for arithmetic and bit wise operations.

A vector can be obtained in different ways, but we are interested in iterating an array at large steps, so this is the constructor we need:

```C#
public Vector (T[] values, int index);
```

This constructor will make a copy of `Count` items from the specified index of the array `values` into a new vector. Take note that `Count` is a static property, which means that even if vector is a struct, its size will be known only at runtime and it will be the same across every instance.

And here the magic begins: the JIT compiler is configured to know at runtime that a vector should not be stored in a variable somewhere in RAM, rather, if hardware acceleration is available it will be stored in special registers (SSE and AVX) inside the CPU.

So, with this knowledge, let's modify our for loop to use vectors!

## Vectorized implementation

So, I'm going to loop over the array, with `Count` wide steps at a time.

```C#
for (int i = 0; i < testsize; i += Vector<int>.Count)
{
    Console.WriteLine($"Vectorized implementation i = {i}");
    var avec = new Vector<int>(a, i);
    var bvec = new Vector<int>(b, i);
    (avec + bvec).CopyTo(c, i);
}
```

And let's compare the underlying JIT-generated x86 assembly. I used [x64dbg](https://x64dbg.com/#start) to peek at the JIT generated code, and with some intuition I was able to isolate the Read-Sum-Write pattern from the assembly mess.

With **simple for loop**:

```asm
lea rax,qword ptr ds:[rax+rcx*4+10]
mov eax,dword ptr ds:[rax]
mov rdx,qword ptr ss:[rbp-20]
mov ecx,dword ptr ss:[rbp-2C]
cmp ecx,dword ptr ds:[rdx+8]
jb 7FFA549B6CB3
call coreclr.7FFAB45FADA0
mov r8d,ecx
lea rdx,qword ptr ds:[rdx+r8*4+10]
add eax,dword ptr ds:[rdx]
mov dword ptr ss:[rbp-5C],eax
```

With **vector**:

```asm
mov rcx,qword ptr ss:[rbp-20]
mov eax,dword ptr ss:[rbp-2C]
movups xmm0,xmmword ptr ds:[rcx+rax*4+10]
movaps xmmword ptr ss:[rbp-50],xmm0
movaps xmm0,xmmword ptr ss:[rbp-40]
movaps xmm1,xmmword ptr ss:[rbp-50]
paddd xmm0,xmm1
movaps xmmword ptr ss:[rbp-B0],xmm0
movaps xmm0,xmmword ptr ss:[rbp-B0]
movaps xmmword ptr ss:[rbp-60],xmm0
```

Well, I'm not an expert, but I can safely state that code is now using SIMD instructions, Hooray! Let's run it...

```txt
PS K:\devel\C#\Rand0081.SimdTutorial> bin\Debug\netcoreapp3.0\Rand0081.SimdTutorial.exe
Vectorized implementation i = 0
Vectorized implementation i = 4
Vectorized implementation i = 8
Vectorized implementation i = 12
Vectorized implementation i = 16
Vectorized implementation i = 20
Vectorized implementation i = 24
Vectorized implementation i = 28
Vectorized implementation i = 32
Vectorized implementation i = 36
Vectorized implementation i = 40

Unhandled Exception: System.IndexOutOfRangeException: Index was outside the bounds of the array.
   at Rand0081.SimdTutorial.Program.Main(String[] args) in K:\devel\C#\Rand0081.SimdTutorial\Program.cs:line 19
```

It seems to run fine, the algorithm is working on 4 elements simultaneously but... **What?** Apparently we cannot load element 44 into a Vector. This is because the vector must be used to its full extent, so the constructor pretends to pull elements 44, 45, 46, 47 from the source array, resulting in an out of range error.

We need a trick. We must manually loop on the last elements. So let's create a little helper function:

```C#
static int GetSplitPoint<T>(int desiredSize) where T : struct
{
    var mask = ~(Vector<T>.Count - 1);
    return desiredSize & mask;
}
```

This function caps the upper bound of the first loop to the highest multiple of Count, according to the base type of the array, telling us where to split the array (*we count on Count to always be a power of 2, no pun intended*). To use it we must split our code and implement the algorithm twice, like so:

```C#
int splitpoint = GetSplitPoint<int>(testsize);

for (i = 0; i < splitpoint; i += Vector<int>.Count)
{
    Console.WriteLine($"Vectorized implementation i = {i}");
    var avec = new Vector<int>(a, i);
    var bvec = new Vector<int>(b, i);
    (avec + bvec).CopyTo(c, i);
}
for (; i < testsize; ++i)
{
    Console.WriteLine($"Simple loop i = {i}");
    c[i] = a[i] + b[i];
}
```

Now we can safely run the app.

```txt
PS K:\devel\C#\Rand0081.SimdTutorial> bin\Debug\netcoreapp3.0\Rand0081.SimdTutorial.exe
Vectorized implementation i = 0
Vectorized implementation i = 4
Vectorized implementation i = 8
Vectorized implementation i = 12
Vectorized implementation i = 16
Vectorized implementation i = 20
Vectorized implementation i = 24
Vectorized implementation i = 28
Vectorized implementation i = 32
Vectorized implementation i = 36
Vectorized implementation i = 40
Simple loop i = 44
c[21] = 42
```

Finally this simple program is vectorized and works as expected.

## Conclusion

In this writing I skipped a lot of details to jump at the fun of coding. I also forgot to say I'm running my code on a Intel i7 860 CPU which, by documentation, only supports SSE instructions. This is why the loop is running over 4 x 32bits integer at a time. On modern CPUs `Count` may be 8 or even 16.

What if we want to run *even faster*? In a next blog I may use parallel loops.

## Links

* Get the [source](https://github.com/rossini-andrea/Rand0081.SimdTutorial) for this article.
* SIMD article on [Wikipedia](https://en.wikipedia.org/wiki/SIMD).
* [MSDN page](https://docs.microsoft.com/en-us/dotnet/api/system.numerics.vector-1) about `Vector<T>`.
