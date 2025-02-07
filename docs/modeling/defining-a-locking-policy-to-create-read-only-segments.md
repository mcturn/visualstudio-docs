---
title: Defining a Locking Policy to Create Read-Only Segments
description: Learn how you can define a policy for a program to lock part or all of a domain-specific language (DSL) model so that it can be read but not changed.
ms.custom: SEO-VS-2020
ms.date: 11/04/2016
ms.topic: conceptual
author: mgoertz-msft
ms.author: mgoertz
manager: jmartens
ms.workload:
  - "multiple"
---
# Defining a Locking Policy to Create Read-Only Segments
The Immutability API of the Visual Studio Visualization and Modeling SDK allows a program to lock part or all of a domain-specific language (DSL) model so that it can be read but not changed. This read-only option could be used, for example, so that a user can ask colleagues to annotate and review a DSL model but can disallow them from changing the original.

 In addition, as author of a DSL, you can define a *locking policy.* A locking policy defines which locks are permitted, not permitted, or mandatory. For example, when you publish a DSL, you can encourage third-party developers to extend it with new commands. But you could also use a locking policy to prevent them from altering the read-only status of specified parts of the model.

> [!NOTE]
> A locking policy can be circumvented by using reflection. It provides a clear boundary for third-party developers, but does not provide strong security.

 More information and samples are available at the Visual Studio [Visualization and Modeling SDK](https://code.msdn.microsoft.com/Visualization-and-Modeling-313535db) website.

[!INCLUDE[modeling_sdk_info](includes/modeling_sdk_info.md)]

## Setting and Getting Locks
 You can set locks on the store, on a partition, or on an individual element. For example, this statement will prevent a model element from being deleted, and will also prevent its properties from being changed:

```csharp
using Microsoft.VisualStudio.Modeling.Immutability; ...
element.SetLocks(Locks.Delete | Locks.Property);
```

 Other lock values can be used to prevent changes in relationships, element creation, movement between partitions, and re-ordering links in a role.

 The locks apply both to user actions and to program code. If program code attempts to make a change, an `InvalidOperationException` will be thrown. Locks are ignored in an Undo or Redo operation.

 You can discover whether an element has a any lock in a given set by using `IsLocked(Locks)` and you can obtain the current set of locks on an element by using `GetLocks()`.

 You can set a lock without using a transaction. The lock database is not part of the store. If you set a lock in response to a change of a value in the store, for example in OnValueChanged, you should allow changes that are part of an Undo operation.

 These methods are extension methods that are defined in the <xref:Microsoft.VisualStudio.Modeling.Immutability> namespace.

### Locks on partitions and stores
 Locks can also be applied to partitions and the store. A lock that is set on a partition applies to all the elements in the partition. Therefore, for example, the following statement will prevent all the elements in a partition from being deleted, irrespective of the states of their own locks. Nevertheless, other locks such as `Locks.Property` could still be set on individual elements:

```csharp
partition.SetLocks(Locks.Delete);
```

 A lock that is set on the Store applies to all its elements, irrespective of the settings of that lock on the partitions and the elements.

### Using Locks
 You could use locks to implement schemes such as the following examples:

- Disallow changes to all elements and relationships except those that represent comments. This allows users to annotate a model without changing it.

- Disallow changes in the default partition, but allow changes in the diagram partition. The user can rearrange the diagram, but cannot alter the underlying model.

- Disallow changes to the Store except for a group of users who are registered in a separate database. For other users, the diagram and model are read-only.

- Disallow changes to the model if a Boolean property of the diagram is set to true. Provide a menu command to change that property. This helps ensure users that they do not make changes accidentally.

- Disallow addition and deletion of elements and relationships of particular classes, but allow property changes. This provides users with a fixed form in which they can fill the properties.

## Lock values
 Locks can be set on a Store, Partition, or individual ModelElement. Locks is a `Flags` enumeration: you can combine its values using '&#124;'.

- Locks of a ModelElement always include the Locks of its Partition.

- Locks of a Partition always include the Locks of the Store.

  You cannot set a lock on a partition or store and at the same time disable the lock on an individual element.

|Value|Meaning if `IsLocked(Value)` is true|
|-|-|
|None|No restriction.|
|Property|Domain properties of elements cannot be changed. This does not apply to properties that are generated by the role of a domain class in a relationship.|
|Add|New elements and links cannot be created in a partition or store.<br /><br /> Not applicable to `ModelElement`.|
|Move|Element cannot be moved between partitions if `element.IsLocked(Move)` is true, or if `targetPartition.IsLocked(Move)` is true.|
|Delete|An element cannot be deleted if this lock is set on the element itself, or on any of the elements to which deletion would propagate, such as embedded elements and shapes.<br /><br /> You can use `element.CanDelete()` to discover whether an element can be deleted.|
|Reorder|The ordering of links at a roleplayer cannot be changed.|
|RolePlayer|The set of links that are sourced at this element cannot be changed. For example, new elements cannot be embedded under this element. This does not affect links for which this element is the target.<br /><br /> If this element is a link, its source and target are not affected.|
|All|Bitwise OR of the other values.|

## Locking Policies
 As the author of a DSL, you can define a *locking policy*. A locking policy moderates the operation of SetLocks(), so that you can prevent specific locks from being set or mandate that specific locks must be set. Typically, you would use a locking policy to discourage users or developers from accidentally contravening the intended use of a DSL, in the same manner that you can declare a variable `private`.

 You can also use a locking policy to set locks on all elements dependent on the element's type. This is because `SetLocks(Locks.None)` is always called when an element is first created or deserialized from file.

 However, you cannot use a policy to vary the locks on an element during its life. To achieve that effect, you should use calls to `SetLocks()`.

 To define a locking policy, you have to:

- Create a class that implements <xref:Microsoft.VisualStudio.Modeling.Immutability.ILockingPolicy>.

- Add this class to the services that are available through the DocData of your DSL.

### To define a locking policy
 <xref:Microsoft.VisualStudio.Modeling.Immutability.ILockingPolicy> has the following definition:

```csharp
public interface ILockingPolicy
{
  Locks RefineLocks(ModelElement element, Locks proposedLocks);
  Locks RefineLocks(Partition partition, Locks proposedLocks);
  Locks RefineLocks(Store store, Locks proposedLocks);
}
```

 These methods are called when a call is made to `SetLocks()` on a Store, Partition, or ModelElement. In each method, you are provided with a proposed set of locks. You can return the proposed set, or you can add and subtract locks.

 For example:

```csharp
using Microsoft.VisualStudio.Modeling;
using Microsoft.VisualStudio.Modeling.Immutability;
namespace Company.YourDsl.DslPackage // Change
{
  public class MyLockingPolicy : ILockingPolicy
  {
    /// <summary>
    /// Moderate SetLocks(this ModelElement target, Locks locks)
    /// </summary>
    /// <param name="element">target</param>
    /// <param name="proposedLocks">locks</param>
    /// <returns></returns>
    public Locks RefineLocks(ModelElement element, Locks proposedLocks)
    {
      // In my policy, users can never delete an element,
      // and other developers cannot easily change that:
      return proposedLocks | Locks.Delete);
    }
    public Locks RefineLocks(Store store, Locks proposedLocks)
    {
      // Only one user can change this model:
      return Environment.UserName == "aUser"
           ? proposedLocks : Locks.All;
    }
```

 To make sure that users can always delete elements, even if other code calls `SetLocks(Lock.Delete):`

 `return proposedLocks & (Locks.All ^ Locks.Delete);`

 To disallow change in all the properties of every element of MyClass:

 `return element is MyClass ? (proposedLocks | Locks.Property) : proposedLocks;`

### To make your policy available as a service
 In your `DslPackage` project, add a new file that contains code that resembles the following example:

```csharp
using Microsoft.VisualStudio.Modeling;
using Microsoft.VisualStudio.Modeling.Immutability;
namespace Company.YourDsl.DslPackage // Change
{
  // Override the DocData GetService() for this DSL.
  internal partial class YourDslDocData // Change
  {
    /// <summary>
    /// Custom locking policy cache.
    /// </summary>
    private ILockingPolicy myLockingPolicy = null;

    /// <summary>
    /// Called when a service is requested.
    /// </summary>
    /// <param name="serviceType">Service requested</param>
    /// <returns>Service implementation</returns>
    public override object GetService(System.Type serviceType)
    {
      if (serviceType == typeof(SLockingPolicy)
       || serviceType == typeof(ILockingPolicy))
      {
        if (myLockingPolicy == null)
        {
          myLockingPolicy = new MyLockingPolicy();
        }
        return myLockingPolicy;
      }
      // Request is for some other service.
      return base.GetService(serviceType);
    }
}
```
