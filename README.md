QuickInject
===========

QuickInject is a Unity 3.5 based IoC container that aims to give the Unity container a performance advantage in basic scenarios.

The goal is to improve performance for basic DI scenarios and provide a simple mechanism to improve optimizations. QuickInject is different from other dependency injection containers in that it has some hook points that allow you (the consumer) to inspect and modify the final generated code for a given resolution -- this also allows unique optimizing abilities not possible with other containers.

##### Goals:

 * Compatible with Unity-style registrations
 * Re-use LifetimeManagers provided by Unity
 * Significantly improve resolution time

##### Supported Unity Features:

 * Child Containers
 * Injection Factory
 * Func<> and Lazy<> of unregistered types
 * Unity Lifetimes
 
##### Unity Features Not Supported:

 * Resolution Overrides * (see ParameterizedInjectionFactory)
 * Interception
 * Open Generic Types (SomeType<>)
 * Named registrations
 * Multiple Injection Members
 
Unique QuickInject Features
---------------------------

 * Generated Code Inspection Engine (**extremely powerful**)
 * ParameterizedLambdaExpressionInjectionFactory -- Expression Trees for resolutions, which then get inlined at resolution
 * ParameterizedInjectionFactory -- Like Unity's Resolution Overrides, but much more performant
 * Dependency Graph Output
 
 
Inspection Engine a.k.a IBuildPlanVisitor
-----------------------------------------

One of the most unique features of QuickInject is its **IBuildPlanVisitor**. Before every new resolution that QuickInject has not generated a build plan for, it calls a user-provided method that can visit the entire Expression Tree about to be compiled.

```cs
public class MyVisitor : IBuildPlanVisitor
{
    public Expression Visitor(Expression expression, Type type, bool slowPath)
    {
        // inspect and/or modify expression
        return expression; // always return expression
    }
}
```

This allows late-bound tasks like Dependency Inspection, Instrumentation ... or even modification of the Expression Tree!

You can register multiple visitors, each tasked to do their own part, and they each chain expressions (i.e. they see expressions returned by the previous visitor)

```cs
var container = new QuickInjectContainer();
container.AddBuildPlanVisitor(new MyLifetimeManagerOptimizerVisitor()); // expression was modified
container.AddBuildPlanVisitor(new LogExpressionsToDiskVisitor()); // reads the modified expression
...
```

This powerful feature allows the developer complete control over what the generated code will be for a given Type.

**Ideas for using this feature**:

* Adding instrumentation for your needs
* Removing LifetimeManager checks for TransientLifetimeManager
* Removing LifetimeManager checks for Http Request lifetimes when you are certain that this is the first resolution for that Http Request (think Controller resolution)
* De-duplicating dependencies (e.g. IFoo depends on IBar and IBaz, and IBar depends on IBaz, you can potentially remove code associated with IBaz the second time)
* Making certain lifetime managers more efficient: for example if the ThreadSpecificLifetimeManager calls into TLS for every lifetime check, you could hoist that outside so that all your thread lookups after the first are free.

ParameterizedLambdaExpressionInjectionFactory&lt;T&gt;
------------------------------------------------------

Imagine scenarios where you do multiple re-resolutions of an InjectionFactory that is some sort of Dictionary in of itself:


```cs
static Foo MyFactory(IUnityContainer container)
{
     var dictionary = container.Resolve<SomeSpecialDictionary>();
     return dictionary["someString"] as Foo;
}
```

What if you could write the above as a Lambda Expression Tree that QuickInject can understand?

```cs
static ParameterizedLambdaExpressionInjectionFactory<Foo, T> MyFactory(string someString)
{
     Type fooType = typeof(Foo);
     ParameterExpression fooParam = Expression.Parameter(fooType);
     
     Expression<Func<Foo, T>> expressionCanBeInlined = Expression.Lambda<Func<Foo, T>>( ... someString ...);
     return new ParameterizedLambdaExpressionInjectionFactory<Foo, T>(expressionCanBeInlined);
}

container.RegisterType<Foo>(MyFactory());
```

This will place the Expression Tree at the resolution site, i.e. it will not generate a function call and because of de-duplication, it will only result in one single resolution of Foo, even though previously multiple function calls to the opaque MyFactory would have to be called.


ParameterizedInjectionFactory&lt;T&gt;
--------------------------------------

A less powerful but more convenient cousin of the ParameterizedLambdaExpressionInjectionFactory&lt;T&gt;, ParameterizedInjectionFactory&lt;T&gt; helps in cases where you have methods like this:


```cs
static Foo MyFactory(IUnityContainer container)
{
     var ia = container.Resolve<IA>();
     var ib = container.Resolve<IB>();
     var ic = container.Resolve<IC>();
     
     return new Foo("someParamIWantHere", ia, ib, ic);
}
```

This can be re-written as:

```cs
static Foo MyFactory(IA ia, IB ib, IC ic)
{
     return new Foo("someParamIWantHere", ia, ib, ic);
}
```

And it would be registered like this:

```cs
container.RegisterType<Foo>(new ParameterizedInjectionFactory<IA, IB, IC, Foo>(MyFactory));
```


This helps if you have a lot of the This helps if you have a lot of the same *IA*, *IB*, *IC* repeated in different factories, and without this it would lead to new resolutions entering the lookup code path, with this it can be minimized.
same *IA*, *IB*, *IC* repeated in different factories, and without this it would lead to new resolutions entering the lookup code path, with this it can be minimized.

RegisterDependencyTreeListener
------------------------------

Another unique feature of QuickInject is the ability for it to give you the full object graph that was needed to compute a particular type.

```cs
void RegisterDependencyTreeListener(Action<ITreeNode<Type>> root);
```

You can register your listener method and generate visual representation of your object dependency graph. This is useful because when using Child Containers you could be pulling dependencies only known easily at runtime. Hence this can be a great tool to understand relationships and guide refactoring efforts.

```cs
var container = new QuickInjectContainer();
container.RegisterDependencyTreeListener( ... your method ... );
```