---
title: PowerShell Casting Oddities
subtitle: When PowerShell casts things differently than other .NET languages
category: powershell
tags: [powershell, classes, casting]
date: 2020-03-03
---
## PowerShell's casting can result in some weird results.

Before we begin, check out this C# class:

```csharp
using System;
public class User
{
    // The user's id (e.g.: StackOverflow id, employee number, or whatever)
    public UInt32 Id { get; set; }
    // The user's name (for display). First and last, nickname, whatever.
    public String Name { get; set; }
    // The user's email (for auth and communications)
    public String Email { get; set; }

    // Get the user from the datastore by Id
    public User(UInt32 id)
    {
        Id = id;
    }

    // Create a new user from their email
    public User(String email)
    {
        Email = email;
    }

    // Create empty users
    // Parameterless default constructors are common in WPF and PowerShell
    public User() { }

    // Prefer user Name but show Email otherwise
    public override string ToString()
    {
        return Name ?? Email;
    }
}
```

It's a pretty simple class with three constructors: the default (parameterless) one, and one each taking the email and id. PowerShell will cast using the constructors of a type, so in this case, you can cast either a string or a number --or a hashtable-- to `User`.

We can add this type to our PowerShell session using `Add-Type`:

```powershell
Add-Type -path User.cs
```

The most interesting of PowerShell's special casting powers is the hashtable cast. Any class that has a default (parameterless) constructor can be created by casting a hashtable of (some of) it's _settable_ properties:

```powershell
[User]@{
    Id = 1
    Name = "Jaykul"
}
```
```
Id Name   Email
-- ----   -----
 1 Jaykul
```

## One of the weirdest examples is this id constructor.

In the case of this particular `User` class, when we cast a number, like `[User]2` it's the same as calling `[User]::new(2)`, but the results of either are a little surprising:

```
Id Name Email
-- ---- -----
 0      2
```

The number was used for the `Email` value, instead of the `Id` that we expected...

The reason for this is actually straightforward, but it's due to a combination of reasons that aren't intuitive.

The bottom line is that the numeric constructor we might have _thought_ we were calling accepts a `uint` (an unsigned int), rather than a signed `int`, and literal numbers in PowerShell are always `[int]` (or `[double]` if they have a decimal place).

PowerShell (well, programming languages in genera) won't cast _implicitly_ if it looses information. Implicit casting is when you try to pass a value to a method (or function) that accepts a different type of value. Explicit casting is when you specify the type by writing it out.

So for instance, a `int` will cast _implicitly_ to a `double`, but not the other way around, because you would loose the fractional value. You can still cast it _explicitly_, by just specifying `[double]`.

PowerShell simply won't call the method that accepts an unsigned int unless you specifically cast the number to an unsigned int. As far as syntax, you can double-cast it:


```powershell
[User][uint]2
```

Or you can call the constructor more explicitly. An _explicit_ cast is one where you specify the type name that you want to cast to. When you specify the type like `[uint]` then PowerShell assumes you understand what you're doing.

```powershell
[User]::new([uint]2)
```

Either way, you'll get a user with an Id but no Name or Email.

```
Id Name Email
-- ---- --------
 2
```

Of course, the other part of the puzzle is why you get an unexpected result, instead of an error like:

> "InvalidArgument: Cannot convert the "2" value of type "System.Int32" to type "User".

The reason is _yet another_ PowerShell casting oddity: PowerShell can (and will) cast _anything_ to a string implicitly, by calling it's `ToString()` method. That's really convenient in a shell, where you regularly want to _display_ things as strings, but sometimes... well, something like this happens.

In other words, PowerShell _will_ implicitly cast an integer to `string`, but _will not_ cast implicitly it to `uint` -- so you get an unexpected constructor and result.