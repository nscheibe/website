---
title: GraphQL - The new Type System
author: Michael Staib
authorURL: https://github.com/michaelstaib
authorImageURL: https://avatars1.githubusercontent.com/u/9714350?s=100&v=4
---

With the start of the version 9 development we started rethinking the type system. 

As _Hot Chocolate_ is adopted by more and more people we received a lot of requests to open up the type system and allow people to extend the type system.

All of what I will discuss with you in this post is not yet released and we are in the midst of development on this one.

<!--truncate-->

Before I start let me outline what our main goals are with this update.

## Conventions

The _Hot Chocolate_ type system provides an easy api to create types, on top of that we have a lot of nice conventions that are applied so that as a developer I do not have to specify every litlle thing.

While this is nice for many people we received a lot of requests from people to let them define their own conventions.

Let me give you an example for this, let's say we have a simple .net type `Foo` like the following:

```csharp
public class Foo 
{
    public string Bar { get; set; }
}
```

When we register this class `Foo` with a schema then it would translate into the following `GraphQL` type:

```graphql
type Foo {
    bar: String
}
```

We have a view rules that we apply here so that the type looks nice in a GraphQL schema. In this little example we just changed the casing of the first letter of the field to match the GraphQL name-style.

If you are having a method like the following `GetBarAsync` this would also translate to a field name `bar`.

You could always opt to describe the type in detail with our schema type APIs but this is in some cases tedious since you would have to specify the name for each field explicitly.

## Attributes

The second request from the comunity to allow for custom attributes. _Hot Chocolate_ brings with it a set of attributes that can be used to describe a schema type actually drawing up a schema type.

```csharp
[GraphQLName("FooFoo")]
public class Foo 
{
    [GraphQLName("barBar")]
    [GraphQLNonNull]
    public string Bar { get; set; }
}
```

```graphql
type FooFoo {
    barBar: String!
}
```

## Custom Descriptors

The third request in this area was to open up the type system and let developer define their own descriptors. This actually is something we want to use ourselves.

So, let\`s say I want to introduce a new filter type. The filter type shall be an input type that describes filters for a certain entity. So, in case of our entity `Foo` I would like to be able to describe a new kind of input type.

```csharp
public class FooFilterType
    : FilterType<Foo>
{
    protected override void Configure(IFilterDescriptor<Foo> descriptor)
    {
        descriptor.Field(t => t.Bar).AllowEquals().AllowStartsWith();
    }

}
```

While this could be done with the current type system, it would be a huge effort since a lot of APIs are `internal` that you would need to make this simple.

## Descriptor Services

We reworked the type initialization flow so that you can now inject services into the initialization.

We provide two custom descriptor services:

- `INamingConventions`
- `ITypeInspector`

`INamingConventions` gives you the ability to define how names are are extracted from c# types.

```csharp
public interface INamingConventions
{
    NameString GetTypeName(Type type, TypeKind kind);

    string GetTypeDescription(Type type, TypeKind kind);

    NameString GetMemberName(MemberInfo member, MemberKind kind);

    string GetMemberDescription(MemberInfo member, MemberKind kind);

    NameString GetArgumentName(ParameterInfo parameter);

    string GetArgumentDescription(ParameterInfo parameter);

    NameString GetEnumValueName(object value);
}
```

Since, in many cases you just want to tune existing naming convetions you can inherit from our default implementation `DefaultNamingConventions` and overwrite what you want to change.

Also, if you want to fetch the name from a custom attribute, then this can also be done by overwriting one of the specified methods.

The, second service is dealing with the type structure and basically inspects a .net type to automatically infer a GraphQL type.

```csharp
public interface ITypeInspector
{
    IEnumerable<Type> GetResolverTypes(Type sourceType);

    IEnumerable<MemberInfo> GetMembers(Type type);

    ITypeReference GetReturnType(MemberInfo member, TypeContext context);

    IEnumerable<object> GetEnumValues(Type enumType);
}
```

Like with the naming conventions we provide a default implementation `DefaultTypeInspector` where you can extend rather then replace.

Both service can be provided via dependency injection into the schema creation.

## Descriptors and Type Initialization

The second part of the type system work focuses on opening up the type descriptors and allowing to fully extend and access the type initializations.

Each type (except for scalars) can be described by using descriptors. The descripto represents a fluent API that is used to describe a type.

The descriptor produces a type definition which is used by the initialization logic to set up the type.

The type is basically initialized in four steps:

1. Create the type instance.
   The first step like with any .net object is to create an instance of a schema type.

2. Initialize the type.
   The descriptor is created and executed. With the resulting type definition the type declares it`s dependencies on other types, resolvers and directives.

3. Complete the type name.
   The type receives it`s name. This is in most cases simple and we would not mention it. But in some cases you want to generate the type name depending on another type name. The type registrar will make sure that the names are completed in the correct order.

4. Complete the type.
   This is the final step that will complete all fields, resolvers, etc. After this step has completed a type will be immutable.

```csharp
public class FilterType<T>
    : InputObjectType<T>
{
    private readonly Action<IFilterDescriptor<T>> _configure;

    public FilterType()
    {
        _configure = Configure;
    }

    public FilterType(Action<IFilterDescriptor<T>> configure)
    {
        _configure = configure
            ?? throw new ArgumentNullException(nameof(configure));
    }

    protected override InputObjectTypeDefinition CreateDefinition(
        IInitializationContext context)
    {
        // create your new descriptor class or extend the original:
        FilterDescriptor<T> descriptor =
            FilterDescriptor.New<T>(
                DescriptorContext.Create(context.Services));
        _configure(descriptor);
        return descriptor.CreateDefinition();
    }

    // introduce you new configure merhod
    protected virtual void Configure(
        IFilterDescriptor<T> descriptor)
    {
    }

    // disable the original configure method by sealing it of
    protected sealed override void Configure(
        IInputObjectTypeDescriptor descriptor)
    {
        throw new NotSupportedException();
    }
}
```

With this we have our new new filter type ready to go.
This is just a simple case and we can hanlde now more complex cases. 

For instance we could tell the initialization context that certain dependant types have to be completed before our extended type is completed so that we can use the dependant type in order to specify our extended type.

```csharp
public class FilterType<T>
    : InputObjectType<T>
{
    private readonly Action<IFilterDescriptor<T>> _configure;

    public FilterType()
    {
        _configure = Configure;
    }

    public FilterType(Action<IFilterDescriptor<T>> configure)
    {
        _configure = configure
            ?? throw new ArgumentNullException(nameof(configure));
    }

    protected override InputObjectTypeDefinition CreateDefinition(
        IInitializationContext context)
    {
        // create your new descriptor class or extend the original:
        FilterDescriptor<T> descriptor =
            FilterDescriptor.New<T>(
                DescriptorContext.Create(context.Services));
        _configure(descriptor);
        return descriptor.CreateDefinition();
    }

    protected override void OnRegisterDependencies(
        IInitializationContext context,
        InputObjectTypeDefinition definition)
    {
        base.OnRegisterDependencies(context, definition);

        context.RegisterDependencyRange(
            definition.GetDependencies(),
            TypeDependencyKind.Named);
    }

    // introduce you new configure merhod
    protected virtual void Configure(
        IFilterDescriptor<T> descriptor)
    {
    }

    // disable the original configure method by sealing it of
    protected sealed override void Configure(
        IInputObjectTypeDescriptor descriptor)
    {
        throw new NotSupportedException();
    }
}
```

 






| [HotChocolate Slack Channel](https://join.slack.com/t/hotchocolategraphql/shared_invite/enQtNTA4NjA0ODYwOTQ0LTBkZjNjZWIzMmNlZjQ5MDQyNDNjMmY3NzYzZjgyYTVmZDU2YjVmNDlhNjNlNTk2ZWRiYzIxMTkwYzA4ODA5Yzg) | [Hot Chocolate Documentation](https://hotchocolate.io) | [Hot Chocolate on GitHub](https://github.com/ChilliCream/hotchocolate) |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------ | ---------------------------------------------------------------------- |

[hot chocolate]: https://hotchocolate.io
[hot chocolate source code]: https://github.com/ChilliCream/hotchocolate
