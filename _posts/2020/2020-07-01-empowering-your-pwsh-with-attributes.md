---
title: Empowering your code with Attributes
subtitle: Use Cases for PowerShell Classes. Validators, argument completers, and more.
category: powershell
tags: [powershell, classes, validator, casting]
date: 2020-03-03
---
## Empowering Your Code with Attributes

PowerShell has a lot of functionality built-in that is based on relatively sophisticated programming concepts. One example of this is PowerShell's attributes.

The first time you write a PowerShell script, you may not use any attributes. But the first time you ask a veteran to help you turn one into a function, you probably had to use a bunch of them without even understanding what they do!

Almost all writers of PowerShell scripts and functions (and compiled cmdlets) should know about the `[CmdletBinding()]` attribute which is applied to the `param()` block to implicitly add the "common" parameters like `-Verbose` and `-Debug` and triggers support for the special `-?` automatic help parameter, because built-in help is one of the core PowerShell concepts.

Most people who've written more than one or two scripts have probably also seen the `[Parameter()]` attribute which can be used to make things `Mandatory` or to split them up by `ParameterSetName`.

And of course, if you've written functions or scripts meant for other people to use, you've [probably](https://redmondmag.com/articles/2018/09/25/validate-input-in-powershell-functions-1.aspx) [become](https://riptutorial.com/powershell/example/29958/parameter-validation) [accustomed](https://jdhitsolutions.com/blog/powershell/2206/powershell-scripting-with-validateset/) [to](https://jdhitsolutions.com/blog/powershell/2188/powershell-scripting-with-validaterange/) [using](https://devblogs.microsoft.com/scripting/simplify-your-powershell-script-with-parameter-validation/) [the](https://learn-powershell.net/2014/02/04/using-powershell-parameter-validation-to-make-your-day-easier/) [built-in](https://mikefrobbins.com/2015/03/31/powershell-advanced-functions-can-we-build-them-better-with-parameter-validation-yes-we-can/) [validation attributes](https://docs.microsoft.com/en-us/dotnet/api/system.management.automation.validateargumentsattribute?view=powershellsdk-7.0.0) like `ValidateSet` and `ValidateRange` -- which keep getting better in each version of PowerShell.

You may have done all of this and more, without ever knowing more about attributes than the fact that there's a bit of square-bracket syntax which just magically ... works! After all, PowerShell uses _custom attributes_ to enhance parameter validation, type casting, intellisense, command discovery, and more.

### What is an Attribute?

Attributes are _metadata_. That is, they are part of the data (within .NET compiled programs) that _describes_ the code. In PowerShell (or rather, in .NET) you can examine and _modifify_ the metadata on classes, methods, and assemblies even while the code is running.

> Disclaimer: Normally, Attributes are an advanced topic in programming. When you're learning PowerShell, you may learn to use them inadvertenlty, without understanding what they are, but in most .NET languages beginners aren't taught about them until they are already familiar with .NET's syntax, common types, and even with object oriented concepts including writing classes and methods, and using inheritance -- because that makes them easier to understand.
>
> If you pick up a book on C#, for instance, you might find "Attributes" way back in chapter 17 or 18, after chapters on Delegates, Exception handling, Threads, and even debugging. It will be with other metadata concepts like assemblies, reflection and security, _probably_ right before the book stops talking about _concepts_ entirely and starts talking about _specific applications_ of C# like working with databases, or writing web pages.
>
> Attributes are critical in real world programming simply because the frameworks we use for working with databases, writing web sites and services, or even writing PowerShell commands use metadata extensively, but you don't need much more understanding of them than "they are metadata about code" to be able to use them,

### Using Attributes in Code

In .NET, an `Attribute` is a class that inherits from `System.Attribute` directly or indirectly.

Attributes are applied to classes, methods, fields and variables by specifying the attribute name in square brackets right before the thing you want the attribute to apply to. Note that although the name of custom attribute classes should always end with "Attribute", you can optionally leave that off when _using_ the attribute. However, in PowerShell, the parentheses that go after the attribute name are never _optional_ (although they are in C#, if the attribute doesn't require parameters to its constructor). Thus, in this PowerShell example, we see `ValidateNotNullAttribute` applied to a `string` variable:

```PowerShell
[ValidateNotNull()]
[string]$Name = ""
```

In PowerShell, even more things than usual support attributes: not only classes and functions but also scripts, including anonymous scriptblocks, as well as parameters, fields, and even variables. Two special cases are worth calling out:

1. Scriptblocks, functions and scripts are annotated by putting the attribute notation _just before_ the `param` statement -- making the param statement necessary.
2. Class Properties. PowerShell class properties do not (yet) support _getters_ and _setters_ on fields, but do support validation and transform attributes. As such, writing _custom_ attributes becomes a possible work-around for some scenarios.

#### Setting Attribute Properties

Some attributes have constructor arguments, and some have additional properties you can set. Usually you can find this information in the documentation, but since this is PowerShell, you can also discover them! Try writing the attribute out using the `::new` type construction syntax in the console to get the OverLoadDefinitions, and construct one and run it through `Get-Member` to see what properties it has. For instance, one of the old PowerShell validation attributes has some new optional functionality in PowerShell 6 and 7:

```
> [ValidateScript]::new

OverloadDefinitions
-------------------
ValidateScript new(scriptblock scriptBlock)


> [ValidateScript]::new({}) | Get-Member -Type Property

TypeName: System.Management.Automation.ValidateScriptAttribute

Name         MemberType Definition
----         ---------- ----------
ErrorMessage Property   string ErrorMessage {get;set;}
ScriptBlock  Property   scriptblock ScriptBlock {get;}
TypeId       Property   System.Object TypeId {get;}
```

As you can see, the `ScriptBlock` property is required in the constructor, but there's a new `set`able property: the `ErrorMessage`.

In PowerShell, any settable property of an attribute can be set within the annotation parentheses as `Name = $Value`, and if it's a boolean property, you can just write the name and leave off the assignment to set it true. Lets take a look at an example:

```PowerShell
filter Get-Base64Content {
    [CmdletBinding()]
    param(
        [ValidateScript({ Test-Path $_ -Type Leaf }, ErrorMessage = "The Path must point to an existing file")]
        [Parameter(Mandatory, ValueFromPipeline, ValueFromPipelineByPropertyName)]
        [Alias("PSPath")]
        [string]$Path
    )
    [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String((Get-Content -Raw $Path)))
}
```

Since that was a pretty complete example, there are a few other things in this code sample you _may_ not be familiar with:

1. **The `filter` keyword** is just like the `function` keyword, except that the _default_ code block is `process` instead of `end`. Since pipeline input is only available in the `process` block, the `filter` keyword makes it easier to write functions that work with pipeline input.
2. You should almost _always_ specify **the `[CmdletBinding()]` attribute** on scripts and functions. It forces strict parameter binding, so you have to specify all your parameter names in the `param()`, and it also adds the default parameters for `-Verbose`, `-ErrorAction`, etc., including the `-?` parameter for getting help -- this is important, because people will expect `-?` to work and if you don't specify `[CmdletBinding()]`, you need to implement that yourself.
3. **The `ValidateScript` attribute**. In PowerShell 5.1 you didn't have the `ErrorMessage` parameter, but as of PowerShell 6, we have this new (optional) property so we can easily customize the message.
4. **The `Parameter` attribute** supports making the `Path` parameter mandatory, and also declaring that it will accept values from the pipeline. Because we've also specified `ValueFromPipelineByPropertyName`, PowerShell will first try to match properties of pipeline objects, and only if that fails, cast the actual pipeline object.
5. **The `Alias` attribute** is generally used for three reasons in PowerShell. The last holds true here:
    1. You can provide aliases for a function to create short names like `gci` for `Get-ChildItem`
    2. You can provide aliases for a parameter to create shorter, simpler, or easier to type names
    3. You can provide aliases for parameters which accept `ValueFromPipelineByPropertyName` to facilitate matching the properties of objects which are likely to be piped to your function. In this case, objects output by Get-ChildItem, like files, have a `PSPath` parameter which is the fully qualified path to the item.


## Writing Your Own Attributes

Since attributes are just classes that derive from `System.Attribute`, you can writing your own in PowerShell very simply. Just inherit like this:

```PowerShell
class HumbleAttribute : Attribute {

}
```

Of course, this `HumbleAttribute` doesn't _do_ anything... except that it's metadata. That means we can find _things_ which have that attribute on them, so if we have a command with the `[Humble()]` attribute on it:

```PowerShell
function Test-Humility {
    [Humble()][CmdletBinding()]
    param()
    Write-Host "I'm humble"
}
```

We could find all commands with that attribute like this:

```PowerShell
Get-Command | Where { $_.ScriptBlock.Attributes.TypeId.Name -eq "HumbleAttribute" }
```

Of course, any properties or methods you add to the attribute can be used at will, and you can inherit from other attributes, so we could, for instance, extend the `CmdletBinding` attribute with a list of dependencies like this:


```PowerShell
class BuildCmdletAttribute : System.Management.Automation.CmdletBindingAttribute {
    [string[]]$Requires

    BuildCmdletAttribute() {}

    BuildCmdletAttribute([string[]]$Requires) {
        $this.Requires = $Requires
    }
}
```

Now we could add that attribute to some build steps, like this:

```PowerShell

function publish {
    [BuildCmdlet(("init", "build"))]param()
    Write-Information "BUILDING: $ModuleName from $Path"
}

function update {
    [BuildCmdlet("init")]param()
    Write-Information "UPDATING dependencies"
    Install-RequiredModules
}

function build {
    [BuildCmdlet(("update","init"))]param()
    Write-Information "BUILDING: YourModuleName from $PScriptRoot"
    Build-Module
}

function init {
    [BuildCmdlet()]param()
    Write-Information "INITIALIZING build variables"
    $Version = gitversion -showvariable SemVer
}
```

And you could write commands that take advantage of that metadata, by pulling it out from the `Attributes`, so that we could, for example, sort a list of these `BuildCmdlet` objects into dependency order, and then ... run them:

```PowerShell
Get-Command |
    Where-Object { $_.ScriptBlock.Attributes.TypeId.Name -eq "BuildCmdletAttribute" } |
    Add-Member ScriptProperty Requires {
        $this.ScriptBlock.Attributes.Where{ $_.TypeId.Name -eq "BuildCmdletAttribute"}.Requires
    } -Passthru -Force |
    Add-Member ScriptProperty Weight {
        $Weight = 1
        foreach ($command in $this.Requires) {
            $Weight += [Math]::Max((Get-Command $command).Weight, 1)
        }
        $Weight
    } -Passthru  -Force |
    Sort-Object Weight -Ov Commands |
    ForEach { & $_ -InformationAction Continue }
```

There's so much more that you can do with Attributes in PowerShell due to the fact that PowerShell has inherent functionality built around them. You can write attributes that produce intellisense for [tab-completion](https://powershell.one/powershell-internals/attributes/auto-completion), and attributes that [coerce values to a certain type](https://powershell.one/powershell-internals/attributes/transformation) (even interacting with the user), and more. Check out Dr. Tobias Weltner's series of articles, starting with his [primer](https://powershell.one/powershell-internals/attributes/primer) where you're sure to learn about at least one built-in attribute you never knew existed...