# Unity Registration
Unity does not require type to be registered to resolve it. 

Any concrete, constructable type could be resolved by Unity. It will even create and supply parameters, if required. In other words, if you can create type with new operator:
```C#
var value = new SomeClass(new SomeOtherClass());
```
you can resolve it from Unity:
```C#
var value = container.Resolve<SomeClass>();
```

You only need to register types if you want to specify:

* Object's lifetime policy
* Non default constructor 
* Instruct Unity to inject properties or fields
* Call a method on the created object
* To create mapping between service and implementation types
* All of the above

## What is a Registration and how it works
When you register a type, you are instructing Unity how to crate and initialize instance of that type during instantiation. You also instruct Unity how to deal with the crated instance during its lifetime.

Once registration is complete, Unity creates a blueprint of the type factory where it stores implementation details (name, to and from types, etc.), information about what members to inject and how, and lifetime manager responsible for managing the instance. <br>
At later time, when type is requested, Unity uses this blueprint to crate a pipeline (resolver delegate) to be used to create type. 

Each Unity container exposes a [collection](xref:Unity.IUnityContainer#Unity_IUnityContainer_Registrations) of available registrations presented as enumeration of [IContainerRegistration](xref:Unity.IContainerRegistration) objects. This collection could be used to filter and select certain registrations as well as to [check if type is registered](xref:#Unity.IUnityContainer#Unity_IUnityContainer_IsRegistered_System_Type_System_String_) and how.

## Registration Metadata
Unity registration may contains following information:

#### [Registered Type](xref:Unity.IContainerRegistration#Unity_IContainerRegistration_RegisteredType) [Required]
A type that will be requested during resolution is called **Registered Type**. In example below `SomeType` would be a registered type. 
```C#
container.RegisterType<SomeClass>();
...
var value = container.Resolve<SomeClass>();
```
Different container authors call this type by different names, [FromType](https://docs.microsoft.com/en-us/previous-versions/msp-n-p/ee650974(v%3dpandp.10)), [ServiceType](https://docs.microsoft.com/en-us/dotnet/api/microsoft.extensions.dependencyinjection.servicedescriptor.servicetype), etc. The key point to remember is that this is the Type container will be holding in internal registry and will be looking for when executing the resolve.

#### [Name](xref:Unity.IContainerRegistration#Unity_IContainerRegistration_Name) [Optional]
Each registration must be unique within the scope on a container it is registered with. A registration is identified by two pieces of information: `Registered Type` and `Name`. 
Adding name to registration allows multiple 'instances' of the same type to be registered with container. 

For example, if you register same service multiple times, each subsequent registration will override the last because in each case you are registering the same type `IService` with the same name `null`:
```C#
container.RegisterType<IService, Service1>();
container.RegisterType<IService, Service2>();
container.RegisterType<IService, Service3>();

var enumeration = container.Resolve<IEnumerable<IService>>();
var count = enumeration.Count();
```
The value of variable `count` will be `1`. 

Adding unique names to registrations makes each unique:
```C#
container.RegisterType<IService, Service1>("1");
container.RegisterType<IService, Service2>("2");
container.RegisterType<IService, Service3>("3");

var enumeration = container.Resolve<IEnumerable<IService>>();
var count = enumeration.Count();
```
In this example the value of variable `count` will be `3`. 

#### [Mapped To Type](xref:Unity.IContainerRegistration#Unity_IContainerRegistration_MappedToType) [Optional]
Sometimes it is referred as [ToType](https://docs.microsoft.com/en-us/previous-versions/msp-n-p/ee650974(v%3dpandp.10)), [ImplementationType](https://docs.microsoft.com/en-us/dotnet/api/microsoft.extensions.dependencyinjection.servicedescriptor.implementationtype), etc. and describes the type Unity should use to create the instance. 
`Mapped To Type` must be descendant of, or implement the `Registered Type`. In other words it must be assignable to a variable of `Registered Type`. 

This registration member crates a mapping between service and implementation types. In following example `IService` is mapped to `Service` and when Unity container is asked to resolve `IService` it will in turn create instance of type `Service` and return it as interface `IService`.
```C#
container.RegisterType<IService, Service>();

var result = container.Resolve<IService>();

Assert(typeof(Service) == result.GetType())
```
For more information see [Type Mapping](mapping.md). 

#### [Lifetime Manager](xref:Unity.IContainerRegistration#Unity_IContainerRegistration_LifetimeManager) [Optional]
This member holds reference to lifetime manager that Unity will be using to manage instance(s) of this type. For more information see [Lifetime Management](../lifetime/lifetime.md) 

## Registration Hierarchy
Unity container provide a way to create child containers (other systems refer to it as resolution scopes) and allows building sophisticated scope trees of registrations. There are few simple rules to follow when dealing with container hierarchies:

* **Types registered in predecessor containers always available in descendant containers**<br>
This is very simple concept, each registration is like a public virtual declaration in C# types. Every descendant inherits it and can use at will.

* **Types registered in descendant containers override the same registration of predecessors**<br>
Following the same analogy with public virtual declarations, each override registration installs its own declaration and hides the one in predecessor containers.