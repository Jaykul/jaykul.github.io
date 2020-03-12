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
    public uint Id { get; set; }
    // The user's name (for display). First and last, nickname, whatever.
    public string Name { get; set; }
    // The user's email (for auth and communications)
    public string Email { get; set; }

    // Get the user from the datastore by Id
    public User(uint id)
    {
        Id = id;
    }

    // Create a new user from their email
    public User(string email)
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

The bottom line is that the numeric constructor we might have _thought_ we were calling accepts a `uint` (an unsigned int), rather than a signed `int`, and literal numbers in PowerShell are always `[int]` (or `[double]` if they have a decimal place) unless you add a suffix.

### A little background.

> In programming, "casting" is when you convert an object from one type to another. We talk about "implicit casting" when you set a variable of one type with a value of another, or pass a value to a method (or function) that accepts a different type of value, and the language converts it from one to the other _for_ you without you asking for that. We talk about "explicit casting" when you specify the type you want to convert to:
>
> In general, programming languages (including PowerShell) prefer not to cast _implicitly_ if it's possible that you'll lose information. So you can implicitly cast from a smaller to a bigger type (e.g. 32bit integer to 64bit), or from 16bit unsigned to 32bit signed, but not to a smaller type (32bit to 16bit integer), or signed to unsigned.

In this case, our numbers are integers, so PowerShell won't _implicitly_ cast them to an unsigned integer, and won't call the method that accepts an unsigned int unless you explicitly make the number an unsigned int.

As usual with PowerShell, there are several ways to do that.

You can specify the type of numbers with a suffix (see [about numeric literals](https://docs.microsoft.com/en-us/powershell/module/Microsoft.PowerShell.Core/About/about_numeric_literals?view=powershell-7)) and then cast it or call the constructor:

```powershell
[User]2u
[User]::new(2u)
```

Or you can cast it to `uint`  and then either cast it again to `User` or call the constructor. Either way, if you explicitly specify the type, PowerShell assumes you understand what you're doing, and lets you.

```powershell
[User][uint]2
[User]::new([uint]2)
```

And in any of those cases, you'll get a user with just the Id set.

```
Id Name Email
-- ---- --------
 2
```

Of course, the other part of the puzzle is why you get an unexpected result, instead of an error like:

> "InvalidArgument: Cannot convert the "2" value of type "System.Int32" to type "User".

The reason is _yet another_ PowerShell casting oddity: PowerShell can (and will) cast _anything_ to a string implicitly, by calling it's `ToString()` method. That's really convenient in a shell, where you regularly want to _display_ things as strings, but sometimes... well, something like this happens.

In other words, PowerShell _will_ implicitly cast an integer to `string`, but _will not_ cast implicitly it to `uint` -- so you get an unexpected constructor and result.

## A few notes on designing types for PowerShell

### Use PowerShell's natural types

While it's fine to use whatever type you want for a property (remember that the hashtable constructor works fine), it's best to avoid unsigned integers and other types that won't cast automatically in constructors or methods. That's also true for types like generic lists. PowerShell has native syntax for arrays, but not for generic collections, and it will unroll generic collections and turn them into arrays if you output them anyway, so it's usually better in types we write for PowerShell to use arrays at the interface.

One option is to write the class using the "right" types as properties, including unsigned integers or generic collections, but to add a constructor that takes an integer or an array, etc.

For example, in this case, we could add this constructor:

```csharp
    public User(Int64 id)
    {
        if (id < 0) {
            throw new ArgumentOutOfRangeException("id","id must be a positive integer");
        }
        if (id > UInt32.MaxValue) {
            throw new ArgumentOutOfRangeException("id","id must be less than or equal to " + uint.MaxValue);
        }
        Id = (uint)id;
    }
```

While it may be a little complicated to write the converting constructor (and you have to think of things like needing to up-size to `int64` in order to accept the maximum value of an unsigned int), this will make it "just work" for users without needing to specify the type suffix.

### Write Type Converters

What if you're dealing with a class like that User one up above, that someone else wrote -- whether in a 3rd party API or in the .NET framework. The bottom line is that if you are getting integer input from users and you want it to just work, but you can't modify the class because it came from someone else, you can write a [PSTypeConverter](https://docs.microsoft.com/en-us/dotnet/api/system.management.automation.pstypeconverter).

You can even write it in pure PowerShell:

```powershell
class UserConverter : System.Management.Automation.PSTypeConverter {
    [bool] CanConvertFrom([PSObject]$psSourceValue, [Type]$destinationType) {
        # Is it an integer number we're trying to convert to a user?
        # NOTE: we're claiming we can convert any integer, but really we can't convert anything bigger than uint
        return $psSourceValue -eq ($psSourceValue -as [int64]) -and $destinationType -eq [User]
    }
    [object] ConvertFrom([PSObject]$psSourceValue, [Type]$destinationType, [IFormatProvider]$formatProvider, [bool]$ignoreCase) {
        return [User]::new([uint]$psSourceValue)
    }

    # The rest of the methods are implemented by calling one of those first two methods:
    [bool] CanConvertFrom([object]$sourceValue, [Type]$destinationType) {
        return $this.CanConvertTo($sourceValue, $destinationType)
    }
    [bool] CanConvertTo([object]$sourceValue, [Type]$destinationType) {
        return $this.CanConvertTo(([PSObject]$sourceValue), $destinationType)
    }
    [object] ConvertFrom([object]$sourceValue, [Type]$destinationType, [IFormatProvider]$formatProvider, [bool]$ignoreCase) {
        return $this.ConvertFrom(([PSObject]$sourceValue), $destinationType, $formatProvider, $ignoreCase)
    }
    [object] ConvertTo([object]$sourceValue, [Type]$destinationType, [IFormatProvider]$formatProvider, [bool]$ignoreCase) {
        return $this.ConvertFrom(([PSObject]$sourceValue), $destinationType, $formatProvider, $ignoreCase)
    }
}

Update-TypeData -TypeName 'User' -TypeConverter 'UserConverter' -Force
```

Ironically, if you use the type `[uint]` instead of `[int64]` in the `CanConvertFrom` method, the test will be more truthful (it won't claim it can convert numbers that are actually too big or too small to be converted), but the result will be unexpected: if you try to convert, say, the number `4294967296` the TypeConverter will admit it can't do the job, and PowerShell will convert it using string constructor, just as it would have _without_ the TypeConverter.

In other words, to get the behavior we want, we write the TypeConverter to lie, and claim it can convert any integer value, which will result in exceptions being thrown when those numbers _can't_ actually be converted to unsigned integers. Which, of course, is the behavior that we want (and matches the behavior of the integer constructor).
