---
layout: post
title:  "Activator.CreateInstance Alternatives with Benchmarks"
date:   2018-08-16 15:52:48 -0400
categories: .net csharp
---
.NET provides multiple ways of generating objects at runtime. Each of these options offer their own trades offs in terms of performance. I will demonstrate that not only are there great performance gains to be had over Activator.CreateInstance(...), but I will also show that there are nuances to how you set up these classes that can dramatically effect performance.

## Why is this important?

While there may be more applications, two examples that I can think of that rely on generating objects at runtime are game development frameworks and IOC (inversion of control) containers. Many game develoment environments are set up to create and manipulate objects that are dynamically loaded in. Being able to do that quickly is super important especially with the recent boom of game development in C# as of late.

IOC containers also need to be able to generate objects quickly because applications that use them will typically rely on the framework to generate most of the objects in the application.

I decided to put together this analysis because I recently started working on a legacy codebase that relies a great deal on Activater.CreateInstance for instantiating objects. Swapping out the implementation of Activater.CreateInstance could load to some better performance. I may report back with my findings.

## Show me the code!

I want you to be able to verify my results if you wish. As such, I've put together a github repo with all the code I used to do these benchmarks: https://github.com/holymoo/DynamicObjectCreationBenchmark

In addition to this, I'm going to break down each segment that I tested in these benchmarks.

### the 'new' operator
The new operator is the normal way of creating new objects in C#.

    public static class NewBuilder
    {
        public static TestObject Build()
        {
            return new TestObject();
        }
    }

This is included because it will serve as our baseline to show us how much performance we are losing by dynamically generating our objects at runtime.

### Activator.CreateInstance
This is the simplest and most common way of generating objects at runtime. It's simple to write and is generally performant enough for most things, **but we can do better.**

**Create Instance without a Template**

    public static class ActivatorCreateBuilderWithoutGeneric
    {
        public static TestObject Build()
        {
            return (TestObject)Activator.CreateInstance(typeof(TestObject));
        }
    }

**Create Instance with a Template**

    public static class ActivatorCreateBuilderWithGeneric
    {
        public static TestObject Build()
        {
            return Activator.CreateInstance<TestObject>();
        }
    }

I'm including both the templated and the non-templated version in this test because there can be performance implications demonstrated in the other methods.

### Serialization

These methods rely on combination of the System.Reflection and System.Runtime.Serialization namespaces. I believe these methods rely primarily on serialization to generate the objects; however, I didn't look at the source code.

**Without a template, without caching**

    public static class FormatterServicesBuilderWithoutCachingWithoutGeneric
    {
        public static object Build(Type t)
        {
            var o = FormatterServices.GetUninitializedObject(t);
            var ctor = t.GetConstructors()[0];
            return ctor.Invoke(o, new object[] { });
        }
    }

**Without a template, with caching**

    public static class FormatterServicesBuilderWithCachingWithoutGeneric
    {
        private static readonly Dictionary<Type, ConstructorInfo> cache = 
            new Dictionary<Type, ConstructorInfo>();

        public static object Build(Type t)
        {
            if (cache.TryGetValue(t, out ConstructorInfo ctor))
            {
                var o = FormatterServices.GetUninitializedObject(t);
                return ctor.Invoke(o, new object[] { });
            }
            else
            {
                var o = FormatterServices.GetUninitializedObject(t);
                ctor = t.GetConstructors()[0];
                cache.Add(t, ctor);
                return ctor.Invoke(o, new object[] { });
            }
        }
    }

**With a template, without caching**

    public static class FormatterServicesBuilderWithoutCachingWithGeneric<T>
    {
        public static T Build()
        {
            var t = typeof(T);
            var o = FormatterServices.GetUninitializedObject(t);
            var ctor = t.GetConstructors()[0];
            return (T)ctor.Invoke(o, new object[] { });
        }
    }

**With a template, with caching**

    public static class FormatterServicesBuilderWithCachingWithGeneric<T>
    {
        private static readonly Type t = typeof(T); 
        private static readonly ConstructorInfo ctor = t.GetConstructors()[0];

        public static T Build()
        {
            var o = FormatterServices.GetUninitializedObject(t);
            return (T)ctor.Invoke(o, new object[] { });
        }
    }

### Emitting IL

These methods are often referenced as being the fastest way to generate objects at runtime. To a certain degree, that is true; however, there are some nuances to the topic that we will see when it comes to the benchmarks.

**Without a template, without caching**

    public static class MsilBuilderWithoutCachingWithoutGeneric
    {
        public delegate object DynamicObjectActivator();

        public static object Build(Type t)
        {
            var dynamicMethod = new DynamicMethod("CreateInstance", t, Type.EmptyTypes, true);
            var ilGenerator = dynamicMethod.GetILGenerator();
            ilGenerator.Emit(OpCodes.Nop);
            ConstructorInfo emptyConstructor = t.GetConstructor(Type.EmptyTypes);
            ilGenerator.Emit(OpCodes.Newobj, emptyConstructor);
            ilGenerator.Emit(OpCodes.Ret);
            var del = (DynamicObjectActivator)dynamicMethod.CreateDelegate(typeof(DynamicObjectActivator));
            return del();
        }
    }

**Without a template, with caching**

    public static class MsilBuilderWithCachingWithoutGeneric
    {
        public delegate object DynamicObjectActivator();
        private static readonly Dictionary<Type, DynamicObjectActivator> cache = new Dictionary<Type, DynamicObjectActivator>();

        public static object Build(Type t)
        {
            if(cache.TryGetValue(t, out DynamicObjectActivator value))
            {
                return value();
            }

            var dynamicMethod = new DynamicMethod("CreateInstance", t, Type.EmptyTypes, true);
            var ilGenerator = dynamicMethod.GetILGenerator();
            ilGenerator.Emit(OpCodes.Nop);
            ConstructorInfo emptyConstructor = t.GetConstructor(Type.EmptyTypes);
            ilGenerator.Emit(OpCodes.Newobj, emptyConstructor);
            ilGenerator.Emit(OpCodes.Ret);

            var del = (DynamicObjectActivator)dynamicMethod.CreateDelegate(typeof(DynamicObjectActivator));
            cache.Add(t, del);

            return del();
        }
    }

**With a template, without caching**

    public static class MsilBuilderWithoutCachingWithoutGeneric
    {
        public delegate object DynamicObjectActivator();

        public static object Build(Type t)
        {
            var dynamicMethod = new DynamicMethod("CreateInstance", t, Type.EmptyTypes, true);
            var ilGenerator = dynamicMethod.GetILGenerator();
            ilGenerator.Emit(OpCodes.Nop);
            ConstructorInfo emptyConstructor = t.GetConstructor(Type.EmptyTypes);
            ilGenerator.Emit(OpCodes.Newobj, emptyConstructor);
            ilGenerator.Emit(OpCodes.Ret);
            var del = (DynamicObjectActivator)dynamicMethod.CreateDelegate(typeof(DynamicObjectActivator));
            return del();
        }
    }

**With a template, with caching**

    public static class MsilBuilderWithCachingWithGeneric<T>
    {
        private static Type t = typeof(T);
        private static Func<T> func;

        public static T Build()
        {
            if(func != null)
            {
                return func();
            }

            var dynamicMethod = new DynamicMethod("CreateInstance", t, Type.EmptyTypes, true);
            var ilGenerator = dynamicMethod.GetILGenerator();
            ilGenerator.Emit(OpCodes.Nop);
            ConstructorInfo emptyConstructor = t.GetConstructor(Type.EmptyTypes);
            ilGenerator.Emit(OpCodes.Newobj, emptyConstructor);
            ilGenerator.Emit(OpCodes.Ret);

            func = (Func<T>)dynamicMethod.CreateDelegate(typeof(Func<T>));
            return func();
        }
    }

### Compiled Linq method

This is the final benchmark that we'll be doing for this post. This is also often referred to the fastest way to generate an object dynamically at runtime. While this may be true, there are some nuances to this much like generating IL at runtime.

**With a template, without caching**

    public static class LinqBuilderWithoutCachingWithGeneric
    {
        public static T Build<T>()
        {
            var t = typeof(T);
            var ex = new Expression[] { Expression.New(t) };
            var block = Expression.Block(t, ex);
            var builder = Expression.Lambda<Func<T>>(block).Compile();
            return builder();
        }
    }

**With a template, with caching**

    public static class LinqBuilderWithCachingWithGeneric<T>
    {
        private static readonly Type t = typeof(T);
        private static readonly Expression[] ex = new Expression[] { Expression.New(t) };
        private static readonly BlockExpression block = Expression.Block(t, ex);
        private static readonly Func<T> builder = Expression.Lambda<Func<T>>(block).Compile();

        public static T Build()
        {
            return builder();
        }
    }

### The actual testing code

Since I am a noob to writing benchmark code, I decided to use the framework benchmark .net for doing this performance analysis. The framework was really nice for providing easy testing and making sure the environment was set up to not introduce side effects that could skew the results.

The code was compiled in x64 with the /optimize flag. In addition to this the benchmarks were run without the debugger attached.

    [ClrJob]
    [RPlotExporter, RankColumn]
    public class TheActualBenchmark
    {
        [Params(1000000)]
        public int N;

        [Benchmark]
        public TestObject StandardNew() => NewBuilder.Build();

        [Benchmark]
        public TestObject ActivatorCreateBuilderWithoutGenericTest() => 
            ActivatorCreateBuilderWithoutGeneric.Build();

        [Benchmark]
        public TestObject ActivatorCreateBuilderWithGenericTest() => 
            ActivatorCreateBuilderWithGeneric.Build();

        [Benchmark]
        public TestObject FormatterServicesBuilderWithoutCachingWithoutGenericTest() => 
            (TestObject)FormatterServicesBuilderWithoutCachingWithoutGeneric
                .Build(typeof(TestObject));

        [Benchmark]
        public TestObject FormatterServicesBuilderWithCachingWithoutGenericTest() =>
            (TestObject)FormatterServicesBuilderWithoutCachingWithoutGeneric
                .Build(typeof(TestObject));

        [Benchmark]
        public TestObject FormatterServicesBuilderWithoutCachingWithGenericTest() =>
            FormatterServicesBuilderWithoutCachingWithGeneric<TestObject>
                .Build();

        [Benchmark]
        public TestObject FormatterServicesBuilderWithCachingWithGenericTest() =>
            FormatterServicesBuilderWithCachingWithGeneric<TestObject>.Build();

        [Benchmark]
        public TestObject MsilBuilderWithoutCachingWithoutGenericTest() =>
            (TestObject)MsilBuilderWithoutCachingWithoutGeneric
                .Build(typeof(TestObject));

        [Benchmark]
        public TestObject MsilBuilderWithCachingWithoutGenericTest() =>
            (TestObject)MsilBuilderWithCachingWithoutGeneric
                .Build(typeof(TestObject));

        [Benchmark]
        public TestObject MsilBuilderWithoutCachingWithGenericTest() =>
            MsilBuilderWithoutCachingWithGeneric<TestObject>.Build();

        [Benchmark]
        public TestObject MsilBuilderWithCachingWithGeneric() =>
            MsilBuilderWithCachingWithGeneric<TestObject>.Build();

        [Benchmark]
        public TestObject LinqBuilderWithoutCachingWithGenericTest() =>
            LinqBuilderWithoutCachingWithGeneric.Build<TestObject>();

        [Benchmark]
        public TestObject LinqBuilderWithCachingWithGenericTest() =>
            LinqBuilderWithCachingWithGeneric<TestObject>.Build();
    }

## Show me the results!

Rank |Method                                                    | Mean          |      StdDev |
----------|----------------------------------------------------------|---------------|-------------|
    **1** |                                              StandardNew |      2.057 ns |   0.0291 ns |
    **2** |                    LinqBuilderWithCachingWithGenericTest |     10.443 ns |   0.0219 ns |
    **3** |                        MsilBuilderWithCachingWithGeneric |     18.629 ns |   0.0363 ns |
    **4** |                 MsilBuilderWithCachingWithoutGenericTest |     32.084 ns |   0.1033 ns |
    **5** |                 ActivatorCreateBuilderWithoutGenericTest |     37.118 ns |   0.1491 ns |
    **6** |                    ActivatorCreateBuilderWithGenericTest |     44.275 ns |   0.1224 ns |
    **7** |       FormatterServicesBuilderWithCachingWithGenericTest |    157.669 ns |   1.6577 ns |
    **8** | FormatterServicesBuilderWithoutCachingWithoutGenericTest |    203.001 ns |   0.4439 ns |
    **9** |    FormatterServicesBuilderWithCachingWithoutGenericTest |    205.826 ns |   0.3978 ns |
   **10** |    FormatterServicesBuilderWithoutCachingWithGenericTest |    206.732 ns |   0.3488 ns |
   **11** |                 MsilBuilderWithoutCachingWithGenericTest | 61,932.402 ns | 764.5614 ns |
   **12** |              MsilBuilderWithoutCachingWithoutGenericTest | 62,795.093 ns | 494.1148 ns |
   **13** |                 LinqBuilderWithoutCachingWithGenericTest | 76,461.628 ns | 998.5591 ns |


### Based of of these results we can draw some pretty solid conclusions

1. The fastest way by far to create an object is by using the new operator. Using the new operator yields performance this is about 18 times faster than Activater.CreateInstance and about 5 times faster than our fastest dynamic implementation.
2. If you can specify the class via a generic, your fastest implementation is going to be using the Linq builder implementation with caching. It's performance was about three times faster than Activator.CreateInstance.
3. If you have to specify your type via the typeof operator, than the fastest implementation is going to be with dynamically generating Msil code. Using this method produced objects in about half the time compared to Activater.CreateInstance.
4. If you're going to try writing your own object creation code, please use caching. None of the implementaions tested were about to compete with Activator.CreateInstance unless they used some form of method caching. Furthermore, without caching our fastest implementation ended up becoming our slowest method of dynamically generating an object.
5. If you plan on only creating on object once, you're better off using Activator.CreateInstance. The method that performed the best in this test did so because they were able to cache the process of building the object.
6. Formatter services should never be used for building objects at runtime.

## Thanks!

I really appreciate you taking the time to read to the end. This information took several days to code, tabulate, and review. I hope you found it helpful.