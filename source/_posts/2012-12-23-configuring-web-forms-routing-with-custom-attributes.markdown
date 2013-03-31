---
comments: true
date: 2012-12-23 18:00:20
layout: post
slug: configuring-web-forms-routing-with-custom-attributes
title: Configuring Web Forms Routing With Custom Attributes
wordpress_id: 226
categories:
- Development
tags:
- ASP.NET
- Routing
- Web Forms
---


> **1/13/2013 Update:** Now _PhysicalFile_ property is filled and updated automatically, using [T4 template](https://github.com/manekovskiy/Web-Forms-Routing-Through-Attributes/blob/master/PhysicalFilePathUpdater.tt). Say good-bye to issues caused by typos and copy-pasting.


Recently I had to add routing to existent ASP.NET Web Forms application. I was (and I suppose I'm still) new to this thing so I started from [Walkthrough: Using ASP.NET Routing in a Web Forms Application](http://msdn.microsoft.com/en-us/library/dd329551\(v=vs.100\).aspx) and it seemed fine until I started coding.

The site was nothing special but approximately 50 pages. And when I started configuring all these pages it felt wrong - I was lost in all these route names, defaults and constraints. If it felt wrong, I thought, why not to try something else. I googled around and found a pretty good thing - [ASP.NET FriendlyUrls](http://nuget.org/packages/Microsoft.AspNet.FriendlyUrls). Scott Hanselman wrote about this in his [Introducing ASP.NET FriendlyUrls - cleaner URLs, easier Routing, and Mobile Views for ASP.NET Web Forms](http://www.hanselman.com/blog/IntroducingASPNETFriendlyUrlsCleanerURLsEasierRoutingAndMobileViewsForASPNETWebForms.aspx) post. At first glance it looked far easier and better, but I wanted to use [_RouteParameters_](http://msdn.microsoft.com/en-us/library/system.web.ui.webcontrols.routeparameter\(v=vs.100\).aspx) for my [_datasource_](http://msdn.microsoft.com/en-us/library/system.web.ui.datasourcecontrol.aspx) controls on pages. ASP.NET FriendlyUrls are providing only "URL segment" concept - string that could be extracted from URL (string between '/' characters in URL). URL Segments could not be constrained and thus automatically validated. Also, segments could not have names, so my idea to use _RouteParameter_ would be killed if I'd go with ASP.NET FriendlyUrls.

At the end of this little investigation I thought that it would be easier to tie together route configuration with page class via custom attribute and conventionally named properties for defaults and constraints. So every page class gets its routing configuration as follows:

{% codeblock lang:csharp %}
namespace RoutingWithAttributes.Foo
{
	[MapToRoute(RouteUrl = "Foo/Edit/{id}")]
	public partial class Edit : Page
	{
		public static RouteValueDictionary Defaults 
		{ 
			get
			{
				return new RouteValueDictionary { { "id", "" } };
			} 
		}

		public static RouteValueDictionary Constraints
		{
			get
			{
				return new RouteValueDictionary { { "id", "^[0-9]*$" } };
			}
		}
	}
}
{% endcodeblock %}

The code above states that _Edit _page in folder _Foo_ of my _RoutingWithAttributes_ web application will be accessible through _http://\<application-url\>/Foo/Edit_ hyperlink with optional _id_ parameter_._ Default value for _id _parameter is empty string but it should be integer number if provided.

For me this works better, it is self describing and I'm not forced to go to some _App_Start\RoutingConfig.cs_ file and search for it. Now how it is working under the hood? Nothing new or special - just a bit of reflection on _Application_Start_ event. And routes are still registered with _[RouteCollection.MapPageRoute](http://msdn.microsoft.com/en-us/library/system.web.routing.routecollection.mappageroute\(v=vs.100\).aspx) _method.

{% codeblock lang:csharp %}
protected void Application_Start(object sender, EventArgs e)
{
	RouteConfig.RegisterRoutes(RouteTable.Routes);
}

public class RouteConfig
{
	public static void RegisterRoutes(RouteCollection routes)
	{
		var mappedPages = Assembly.GetAssembly(typeof (RouteConfig))
				.GetTypes()
				.AsEnumerable()
				.Where(type => type.GetCustomAttributes(typeof (MapToRouteAttribute), false).Length == 1);

		foreach (var pageType in mappedPages)
		{
			var defaultsProperty = pageType.GetProperty("Defaults");
			var defaults = defaultsProperty != null ? (RouteValueDictionary)defaultsProperty.GetValue(null, null) : null;

			var constraintsProperty = pageType.GetProperty("Constraints");
			var constraints = constraintsProperty != null ? (RouteValueDictionary)constraintsProperty.GetValue(null, null) : null;

			var dataTokensProperty = pageType.GetProperty("DataTokens");
			var dataTokens = dataTokensProperty != null ? (RouteValueDictionary)dataTokensProperty.GetValue(null, null) : null;

			var routeAttribute = (MapToRouteAttribute)pageType.GetCustomAttributes(typeof(MapToRouteAttribute), false)[0];

			if(string.IsNullOrEmpty(routeAttribute.RouteUrl))
				throw new NullReferenceException("RouteUrl property cannot be null");

			if (string.IsNullOrEmpty(routeAttribute.PhysicalFile))
				throw new NullReferenceException("PhysicalFile property cannot be null");

			if(!VirtualPathUtility.IsAppRelative(routeAttribute.PhysicalFile))
				throw new ArgumentException("Property should be application relative URL", "PhysicalFile");

			routes.MapPageRoute(pageType.FullName, routeAttribute.RouteUrl, routeAttribute.PhysicalFile, true, defaults, constraints, dataTokens);
		}
	}
}
{% endcodeblock %}

Route name is equal to the _FullName_ property of page type. Since [_Type.FullName_](http://msdn.microsoft.com/en-us/library/system.type.fullname.aspx) includes both namespace and class name it guarantees route name uniqueness across the application.

To utilize route links generation I had to create two extension methods for _Page_ class. These methods are just wrappers for [_Page.GetRouteUrl_](http://msdn.microsoft.com/en-us/library/system.web.ui.page.getrouteurl\(v=vs.100\).aspx) method.

{% codeblock lang:csharp %}
public static class PageExtensions
{
	public static string GetMappedRouteUrl(this Page thisPage, Type targetPageType, object routeParameters)
	{
		return thisPage.GetRouteUrl(targetPageType.FullName, routeParameters);
	}

	public static string GetMappedRouteUrl(this Page thisPage, Type targetPageType, RouteValueDictionary routeParameters)
	{
		return thisPage.GetRouteUrl(targetPageType.FullName, routeParameters);
	}
}
{% endcodeblock %}

So now I can generate link to _Foo.Edit_ page as follows:

{% codeblock lang:aspx-cs%}
    <a href='<%= Page.GetMappedRouteUrl(typeof(RoutingWithAttributes.Foo.Edit), new { id = 1 }) %>'>Foo.Edit</a>
{% endcodeblock %}

And it will produce _http://\<application-url\>/Foo/Edit/1_ link.

Described approach helped me to accomplish task without frustration and I'm satisfied with the results.

Code for this article is [hosted on GitHub](https://github.com/manekovskiy/Web-Forms-Routing-Through-Attributes) feel free to use it if you liked the idea.