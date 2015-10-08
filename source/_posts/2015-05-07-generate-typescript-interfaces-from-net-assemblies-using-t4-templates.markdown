---
layout: post
title: "Generate TypeScript Interfaces From .NET Assemblies Using T4 Templates"
date: 2015-05-07 11:05
comments: true
categories:
- Development
tags:
- .NET
- TypeScript
- Codegeneration
---

## Introduction

When it comes to writing the HTML/JavaScript client for *your* ("your" here means you own the code or have direct access to the assemblies) web service there is one thing that bothers everyone -  translating classes from .NET to JavaScript. The problem is that whenever your service contract changes you need to reflect this change in your client application. Yes, most of the time this is not the case when the service is already in production but when the client and the service are both being written at the same time I think you would agree that continuous changes in service contract are a common thing. 

Another big issue - even if current service contract (read API version) is "frozen" and is not going to change in future you still have to manually translate all your .NET classes to the JavaScript. It is OK if you have a handful of classes, but can you imagine (or even recall) the pain of translating couple of dozens of C# classes to the JavaScript?

That's it, that is why I've decided to share my approach to this issue of translating the .NET classes to JavaScript.

## The Problem

Lets imagine the situation where we have two teams that are working on two projects - the server side and the client side. The server side is represented by ASP.NET WebAPI service and the client side is an HTML/JavaScript application. As the server project progresses client team notices that it continuously have to make little adjustments "here and there" to keep up to date with the WebAPI changes in its DTO classes. So the problem is to automate this tedious for both teams process.

> As of writing this post I've found that there is a question on StackOverflow showing an interest to this topic - [How to reuse existing C# class definitions in TypeScript projects](http://stackoverflow.com/questions/12957820/how-to-reuse-existing-c-sharp-class-definitions-in-typescript-projects). 

## The Solution

As always there are two ways of solving the problem - use existing solution or write a new one.

There are at least two tools are available - [TypeLite](http://type.litesolutions.net/) and [T4TS](https://github.com/cskeppstedt/t4ts). Everything is good with these tools but when it came to customization it turned out that you need to decorate the classes with some fancy attributes or code transformation functions. This means that you should mix in the requirements like module/property naming convention to the classes that are not even aware of existence of some client project that indirectly depends on them. 

You can call me a purist but hey, why would I need to keep the metadata required for one project in another? And why should I complicate things and instruct the team working with a server side of how to decorate the classes with attributes that are needed by other team? Simple things should be simple. I just want my C# classes/structs/enums to be transformed to the TypeScript interfaces/classes/enums.

<p><span class="pullquote-right" data-pullquote="Most of the time you will not find the &quot;ready for use&quot; solution that will 100% satisfy you. Best case is that you'll find something that is simple and easy to change."></span>
From my experience when it comes to codegeneration most of the time you will not find the "ready for use" solution that will 100% satisfy you. Best case is that you'll find something that is simple and easy to change.
</p>
 
So I've chosen the second path - hack my own solution. For DTOs I've decided to write a code generator based on [T4 Text Templates](https://msdn.microsoft.com/en-us/library/ee844259%28v=vs.120%29.aspx) and reflection. And since I have a TypeScript based project my code generation templates are producing [TypeScript](http://www.typescriptlang.org/) code. Why TypeScript? For me, the main reason is compile time errors. I like that I can have classes and interfaces which usage will be checked by compiler at development time and I will see the mistakes before I run the app. Also it is worth to mention that TypeScript supports almost all features of the the [ECMAScript 6](https://github.com/lukehoban/es6features#readme) which is also good because investing time in TypeScript now I will be up to date with the latest standard available.

I also strongly believe that it is critically important to run code generation on every build and have no auto-generated things committed in a source control. This approach minimizes the probability of mistakes made by engineers (yes, I've had experience when warnings like `// This code was auto-generated` were ignored).

### Code

Since I'm going to generate classes I have to describe the metadata I need. This would be the name of the interface/enum and a list of its members:

{% codeblock lang:csharp Metadata Classes - MetadataModels.cs %}

internal enum DtoTypeKind
{
	Interface,
	Enum,
	Class
}

internal class DtoType
{
	public string Name { get; set; }
	public DtoTypeKind Kind { get; set; }
	public IEnumerable<DtoMember> Members { get; set; }
}

internal class DtoMember
{
	public string Name { get; set; }
	public Type Type { get; set; }
}

{% endcodeblock %}

`MetadataHelper` class is the heart of solution - it will extract the data needed for codegeneration using reflection:

{% codeblock lang:csharp MetadataHelper.cs %}

internal static class MetadataHelper
{
	public static DtoType[] GetDtoTypesMetadata(IEnumerable<Type> types)
	{
		return types
			.Where(t => !t.IsAbstract) // We are not interested in abstract classes
			.Where(t => t.GetCustomAttribute<DataContractAttribute>() != null)
			.Select(t => new DtoType
			{
				Name = t.Name,
				// struct => interface
				// class => class
				// enum => enum. Must check for enum first because it is a ValueType and we want to avoid enums to be generaed as interfaces
				Kind = t.IsEnum
						? DtoTypeKind.Enum
						: t.IsValueType
							? DtoTypeKind.Interface
							: DtoTypeKind.Class,
				Members = t.IsEnum // For enum types we should get its values except the "value__" field
					? t.GetFields()
						.Where(f => f.GetCustomAttribute<DataMemberAttribute>() != null && f.Name != "value__")
						.Select(f => new DtoMember
						{
							Name = f.Name,
							Type = f.FieldType
						})
					: t.GetProperties(BindingFlags.Public | BindingFlags.Instance)
						.Where(p => p.GetCustomAttribute<DataMemberAttribute>() != null)
						.Select(p => new DtoMember
						{
							Name = p.Name,
							Type = p.PropertyType
						})
			})
			.ToArray();
	}
}

{% endcodeblock %}

To be able to parameterize and run codegeneration on every build I'm using preprocessed T4 templates (for more information on topic please refer to the Oleg Sych's [Understanding T4: Preprocessed Text Templates](http://www.olegsych.com/2009/09/t4-preprocessed-text-templates/) blog post). Preprocessed template generates a partial class that I'll be able to extend with metadata I need:

{% codeblock lang:csharp %}

internal partial class TypesGenerator
{
	public DtoType[] DtoTypes { get; set; }

	private IEnumerable<DtoType> Interfaces { get { return DtoTypes.Where(t => t.Kind == DtoTypeKind.Class || t.Kind == DtoTypeKind.Interface); } }
	private IEnumerable<DtoType> Enums { get { return DtoTypes.Where(t => t.Kind == DtoTypeKind.Enum); } }
}

{% endcodeblock %}

And the actual template. It also contains a helper method that translates .NET types to the corresponding TypeScript type names. 

{% codeblock lang:csharp T4 Template - TypesGenerator.tt %}
<#@ template language="C#" visibility="internal" #>
<#@ assembly name="System.Core" #>
<#@ import namespace="System.Collections.Generic" #>
//------------------------------------------------------------------------------
// <auto-generated>
//     This code was generated by a tool.
//     Runtime Version: <#= Environment.Version #>
//
//     Changes to this file may cause incorrect behavior and will be lost if
//     the code is regenerated.
// </auto-generated>
//------------------------------------------------------------------------------

"use strict";

// Interfaces
<# foreach(var @interface in this.Interfaces) { #>

export interface <#= @interface.Name #> {

<#    foreach(var member in @interface.Members) { #>
	<#= member.Name#>?: <#= GetTypeScriptFieldTypeName(member.Type) #>;
<#    } #>
}
<# } #>


// Enums
<# foreach(var @enum in this.Enums) { #>

export enum <#= @enum.Name #> {
<#    foreach(var member in @enum.Members) { #>
	<#= member.Name #> = <#= (int)Enum.Parse(member.Type, member.Name) #>,
<#    } #>
}
<# } #>
<#+
	/// <summary>
	/// Returns a corresponding TypeScript type for a given .NET type
	/// </summary>
	public static string GetTypeScriptFieldTypeName(Type type)
	{
		var numberTypes = new HashSet<Type>
		{
			typeof(sbyte), typeof(byte), typeof(short),
			typeof(ushort), typeof(int), typeof(uint),
			typeof(long), typeof(ulong), typeof(float),
			typeof(double), typeof(decimal)
		};
		var stringTypes = new HashSet<Type>
		{
			typeof(char), typeof(string), typeof(Guid)
		};

		var result = "";
		var isCollectionType = false;
		// Check if it is a generic. We support only generics which are compatible with IEnumerable<T> and have only one generic argument
		if (type.IsGenericType) {
			if (!typeof(IEnumerable<object>).IsAssignableFrom(type) && type.GetGenericArguments().Length > 1) {
				throw new Exception(string.Format("The generic type {0} must implement IEnumerable<T> and must have no more than 1 generic argument.", type.FullName));
			}
			// strip the generic type leaving the first generic argument
			type = type.GetGenericArguments()[0];
			isCollectionType = true;
		}

		// Check if it is a primitive type
		if (numberTypes.Contains(type)) result = "number";
		else if (stringTypes.Contains(type)) result = "string";
		else if (type == typeof(bool)) result = "boolean";
		// It is enum/class/struct -> return its name as-is
		else result = type.Name;

		if(isCollectionType) result += "[]";

		return result;
	}
#>

{% endcodeblock %}

And the usage is very simple. I've created a console application which could be launched for example on CI server during the build.  

{% codeblock lang:csharp %}

static void Main(string[] args)
{
	if (args.Length == 0 || args[0].IndexOfAny(Path.GetInvalidPathChars()) >= 0)
	{
		throw new ArgumentException("Invalid argument. First argument should be a valid file path.");
	}

	var fileName = args[0];
	var typesMetadata = MetadataHelper.GetDtoTypesMetadata(typeof(Todo).Assembly.ExportedTypes);
	var typesGenerator = new TypesGenerator { DtoTypes = typesMetadata };
	File.WriteAllText(fileName, typesGenerator.TransformText().Trim());
}

{% endcodeblock %}

## Conclusion

As you can see with a very little effort I've got a working and open to any customizations codegenerator. As always the code from this post is available on [Github](https://github.com/manekovskiy/webapi-typescript-proxy-generator). Feel free to clone and adjust to your needs.

Keep the code simple!