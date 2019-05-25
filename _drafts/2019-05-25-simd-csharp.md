# C# - SIMD and `Vector<T>`

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

Code is straightfoward; it creates 3 arrays, and loops on all elements a pair at once, and sums it to store the result in the third array, an element at once. Let's try it!

```txt
PS K:\devel\C#\Rand0081.SimdTutorial> dotnet build .\Rand0081.SimdTutorial.csproj

PS K:\devel\C#\Rand0081.SimdTutorial> bin\Debug\netcoreapp3.0\Rand0081.SimdTutorial.exe
c[21] = 42
```

This program is so simple that it already runs at light speed. But, can we make it run *faster*? The answer is yes! As we already know we can leverage SIMD instructions and perform multiple sums at a time.

## Introducing `Vector<T>`

`Vector<T>` is a structure

So, with this knowledge, let's modify our for loop to use `Vector<T>`!

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

And let's compare the underlying JIT-generated x86 assembly. I used [sharplab](https://sharplab.io) to disassemble JIT code, and with some intuition I was able to isolate the Read-Sum-Write pattern from the assembly mess.

**With linear for loop**

```asm
L00c8: lea rcx, [rcx+rdx*4+0x10]
L00cd: mov ecx, [rcx]
L00cf: mov rax, [rbp-0x20]
L00d3: mov edx, [rbp-0x2c]
L00d6: cmp edx, [rax+0x8]
[...]
L00e0: mov r8d, edx
L00e3: lea rax, [rax+r8*4+0x10]
L00e8: add ecx, [rax]
L00ea: mov [rbp-0x64], ecx
```

**With `Vector<T>`**

```asm
L00fe: vmovupd ymm0, [rcx+rax*4+0x10]
L0104: vmovupd [rbp-0x50], ymm0
[...]
L0135: vmovupd ymm0, [rcx+rax*4+0x10]
L013b: vmovupd [rbp-0x70], ymm0
L0140: vmovupd ymm0, [rbp-0x50]
L0145: vmovupd ymm1, [rbp-0x70]
L014a: vpaddd ymm0, ymm0, ymm1
L014e: vmovupd [rbp-0xf0], ymm0
L0156: vmovupd ymm0, [rbp-0xf0]
L015e: vmovupd [rbp-0x90], ymm0
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

It seems to run fine, the algorithm is working on 4 elements simultaneously but... **What?** Apparently we cannot load element 44 into a Vector. This is because the Vector must be used to its full extent, so the constructor pretends to pull elements 44, 45, 46, 47 from the source array, resulting in an out of range error.

We need a workaround. We must manually loop on the last elements. So let's create a little helper function:

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

But what if we want to run *even faster*? In the next blog I will try to use parallel loops.
