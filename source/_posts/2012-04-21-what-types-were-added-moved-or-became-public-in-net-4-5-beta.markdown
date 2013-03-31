---
comments: true
date: 2012-04-21 00:00:06
layout: post
slug: what-types-were-added-moved-or-became-public-in-net-4-5-beta
title: What Types Were Added, Moved or Became Public in .NET 4.5 Beta
wordpress_id: 105
categories:
- Development
tags:
- .NET
- .NET 4.5 Beta
---

There is a [whitepaper](http://www.asp.net/vnext/overview/whitepapers/whats-new) about new features of ASP.NET 4.5, dozens of blog posts, videos from conferences, some tutorials and [MSDN](http://msdn.microsoft.com/en-us/library/ms171868(v=vs.110).aspx) topic describing overall changes. But why there are no reports about what types were actually added in .NET 4.5 Beta? Where are lists of "new types added to assembly N" or "types that became public in assembly N"? I understand that these lists are not very interesting and that it is more convenient to read descriptions of the new features. Nobody cares about details until they start working against you. And this is normal, right and ok, but now I have some free time and I want to share info about new types that were added or became public in .NET 4.5 comparing to .NET 4.0.

Why I became so interested in this? Well, I accidentally found that [_System.Web.DynamicData.IFilterExpressionProvider_](http://msdn.microsoft.com/en-us/library/system.web.dynamicdata.ifilterexpressionprovider(v=vs.110).aspx) interface became public. Briefly, this allows to work with [_QueryExtender_](http://msdn.microsoft.com/en-us/library/system.web.ui.webcontrols.queryextender.aspx) and finally (I really wanted this since .NET 3.5) support its [_DynamicFilterExpression_](http://msdn.microsoft.com/en-us/library/system.web.dynamicdata.dynamicfilterexpression.aspx). <del>Maybe on next weekend I'll post</del> I've already posted about [my experiments with IFilterExpressionProvider](http://amanek.com/?p=100), take a look if you are interested, and now lets return to the main topic.

It is pretty simple to do compiled assemblies diff if you have [NDepend](http://www.ndepend.com/) or similar tool, but what if you (like me) have no license? I started thinking to enumerate public types via reflection, but soon recalled that [.NET 4.5 beta replaces assemblies of .NET 4.0](http://www.west-wind.com/weblog/posts/2012/Mar/13/NET-45-is-an-inplace-replacement-for-NET-40) during installation and Assembly.LoadFrom will not work. To overcome this I decided to parse and compare XML documentation that comes along with all .NET assemblies. Simple as that, every public type is documented and difference between old and new version of documentation will give me at least names of types.

Ok, where to get xml documentation files for .NET 4.0? Binaries with xml docs of .NET 4.0 and 4.5 are located in _<SysDrive>:\Program Files[(x86)]\Reference Assemblies\Microsoft\Framework\.NETFramework_.


{% img center /images/posts/reference-assemblies-folder.png 'Reference assemblies folder location' 'Reference assemblies folder location' %}

What I wanted is to get some statistics. There are 969 new public types in .NET 4.5. But it does not mean that those are completely new things, because it is not, it means that out of the box .NET 4.5 Beta has +969 new types comparing to .NET 4.0 and now there are totally 14971 public and documented types in .NET 4.5. Almost 15K only public types - that's incredibly huge number.

{% img center /images/posts/comparison-of-types-count-between-net40-and-net45.png 'Round Diagram showing types count of .NET 4.0 and .NET 4.5 Beta' 'Round Diagram showing types count of .NET 4.0 and .NET 4.5 Beta' %}

Most of new types are located in _System.IdentityModel_, _System.Web_ and _System.Windows.Controls.Ribbon_ assemblies. Taking into account that _System.IdentityModel_ is providing authentication and authorization features and _System.Windows.Controls.Ribbon_ is UI library allowing use of Microsoft Ribbon for WPF, we can make a conclusion that vast amount of new changes is connected with web.

{% img center /images/posts/new-types-count-by-assembly.png 'Histogram of new types count by asembly' 'Histogram of new types count by asembly' %}

But the most interesting thing was to examine minor changes and see that something new and really useful has been added. And I encourage you to look over list of new classes and I bet you will find something interesting.

LINQPad script with which I did the documentation comparison is listed below. Excel report with some diagrams is also available [online](https://docs.google.com/open?id=0B4z0as-FFbTdaUVlOXZBblctRDQ).

{% codeblock lang:csharp %}
void Main()
{
	// in case of non 64 bit system change "Program Files (x86)" to "Program Files"
	string net40Dir = @"C:\Program Files (x86)\Reference Assemblies\Microsoft\Framework\.NETFramework\v4.0\";
	string net45Dir = @"C:\Program Files (x86)\Reference Assemblies\Microsoft\Framework\.NETFramework\v4.5\";

	// 1. Get all types from all xml doc files in both directories that are containing .NET assemblies and group them by assemblies
	var net40Grouped = GetPublicTypesByAssembly(net40Dir);
	var net45Grouped = GetPublicTypesByAssembly(net45Dir);

	// 2. Get list of newly added assemblies
	var newAssemblies = net45Grouped.Where(kvp => !net40Grouped.ContainsKey(kvp.Key)).ToList();
	Console.WriteLine("New assemblies in .NET 4.5 Beta: (total count - {0})", newAssemblies.Count);
	newAssemblies.ForEach(kvp => Console.WriteLine(kvp.Key));

	Console.WriteLine();

	// 3. Get all assemblies that are not present in .NET 4.5 beta
	var nonExistentAssemblies = net40Grouped.Where(kvp => !net45Grouped.ContainsKey(kvp.Key)).ToList();

	Console.WriteLine("Assemblies that are not present in .NET 4.5 Beta folder comparing to .NET 4.0: (total count - {0})", nonExistentAssemblies.Count);
	nonExistentAssemblies.ForEach(kvp => Console.WriteLine(kvp.Key));

	Console.WriteLine();

	// 4. Get all new types in .NET 4.0 and .NET 4.5 Beta assemblies
	var net40 = net40Grouped.SelectMany(kvp => kvp.Value).ToList();
	var net45 = net45Grouped.SelectMany(kvp => kvp.Value).ToList();

	var newTypes = net45.Except(net40).ToList();
	Console.WriteLine("Types count in .NET 4.0:|{0}", net40.Count);
	Console.WriteLine("Types count in .NET 4.5 Beta:|{0}", net45.Count);
	Console.WriteLine("New types count in .NET 4.5 Beta comparing to .NET 4.0:|{0}", newTypes.Count);

	// 5. Get assemblies that are containing new types
	var assembliesWithChanges = net45Grouped.Where(kvp => newTypes.Any(type => kvp.Value.ContainsValue(type.Value)));

	// 6. Remove existent in .NET 4.0 types from assembliesWithChanges to get clear lists of new types grouped by assemblies
	var newTypesGrouped = assembliesWithChanges
		.ToDictionary(typesGroup => typesGroup.Key, typesGroup => typesGroup.Value.Except(net40)
		.Select(kvp => kvp.Value).ToList());

	Console.WriteLine("New types by assembly:");
	foreach (var assemblyWithNewTypes in newTypesGrouped)
	{
		Console.WriteLine("{0}|{1}", assemblyWithNewTypes.Key, assemblyWithNewTypes.Value.Count);
		foreach (var typeName in assemblyWithNewTypes.Value)
		{
			Console.WriteLine(typeName);
		}
		Console.WriteLine();
	}
}

Dictionary<string, Dictionary<int, string>> GetPublicTypesByAssembly(string xmlDocsDirectory)
{
	string[] xmlDocFiles = Directory.GetFiles(xmlDocsDirectory, "*.xml");
	var result = new Dictionary<string, Dictionary<int, string>>();

	foreach (var xmlDoc in xmlDocFiles)
	{
		var root = XDocument.Load(xmlDoc).Root;
		if (root == null) continue;

		var members = root.Element("members");
		if (members == null) continue;

		var typesByAssembly = members.Elements("member")
			.Where(e => e.Attribute("name").Value.StartsWith("T:"))
			.ToDictionary(e => e.Attribute("name").Value.GetHashCode(), e => e.Attribute("name").Value.Substring(2) /* T: */);

		result.Add(Path.GetFileNameWithoutExtension(xmlDoc) + ".dll", typesByAssembly);
	}

	return result;
}
{% endcodeblock %}
	
And at the end here are links that will help a bit to embrace the changes of .NET 4.5 Beta:
	
  * [What's New in the .NET Framework 4.5 Beta](http://msdn.microsoft.com/en-us/library/ms171868\(v=vs.110\).aspx)
  * [Deep dive into the kernel of the .NET Framework](http://channel9.msdn.com/Events/BUILD/BUILD2011/TOOL-813T)
  * [.NET Framework Versions and Dependencies](http://msdn.microsoft.com/en-us/library/bb822049\(v=vs.110\).aspx)
  * [What's Obsolete in the .NET Framework](http://msdn.microsoft.com/en-us/library/ee461502\(v=vs.110\).aspx)
  * [List of known issues in .NET 4.5 Beta](http://go.microsoft.com/fwlink/?LinkID=237569)

Happy digging in new .NET and don't hesitate sharing!
