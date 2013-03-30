---
comments: true
date: 2012-05-01 17:12:00
layout: post
slug: how-to-implement-configurable-dynamic-data-filters-in-asp-net-4-5
title: How to Implement Configurable Dynamic Data Filters in ASP.NET 4.5
wordpress_id: 100
categories:
- Development
tags:
- .NET 4.5 Beta
- ASP.NET
- Dynamic Data
---

Every time, when we speaking about data driven web applications there is a task of providing data filtering feature or configurable filters with ability to save the search criteria individually for each user. The most convenient filtering experience I have ever encountered were the bug tracking systems. Fast and simple. To get the idea of what I'm talking about just look at [Redmine Issues page](http://www.redmine.org/projects/redmine/issues).
Can we implement something similar with pure ASP.NET, particularly with [_ASP.NET Dynamic Data_](http://msdn.microsoft.com/en-us/library/ee845452.aspx)? Why Dynamic Data? Because of its focus on metadata which is set by attributes from [DataAnnotations namespace](http://msdn.microsoft.com/en-us/library/system.componentmodel.dataannotations.aspx) and convention over configuration approach for building data driven applications. Its simple and convenient, and does not take much efforts to extend it.

For filtering Dynamic Data offers us [_Filter Templates_](http://msdn.microsoft.com/en-us/library/ee377606.aspx#filter_templates) with [_FilterRepeater_](http://msdn.microsoft.com/en-us/library/system.web.dynamicdata.filterrepeater.aspx) control. To get the idea of how Dynamic Data Filter Templates are working I highly recommend reading a great post of [Oleg Sych](http://www.olegsych.com/) ["Understanding ASP.NET Dynamic Data: Filter Templates"](http://www.olegsych.com/2010/07/understanding-aspnet-dynamic-data-filter-templates/).

Until .NET 4.5 there were no extension points where we could retake control over filter templates creation. And surprisingly, I found that interface [_IFilterExpressionProvider_](http://msdn.microsoft.com/en-us/library/system.web.dynamicdata.ifilterexpressionprovider(v=vs.110).aspx) became public in .NET 4.5. So now we can extend Dynamic Data filtering mechanism.


# ASP.NET Dynamic Data QueryableFilterRepeater


For the jump start lets remind how List PageTemplate in Dynamic Data looks like:

{% codeblock lang:aspx-cs %}
<asp:QueryableFilterRepeater runat="server" ID="FilterRepeater">
	<ItemTemplate>
		<asp:Label runat="server" Text='<%# Eval("DisplayName") %>' OnPreRender="Label_PreRender" />
		<asp:DynamicFilter runat="server" ID="DynamicFilter" OnFilterChanged="DynamicFilter_FilterChanged" /><br />
	</ItemTemplate>
</asp:QueryableFilterRepeater>

<asp:GridView ID="GridView1" runat="server" DataSourceID="GridDataSource" >
<%-- Contents and styling omited for brevity --%>
</asp:GridView>

<asp:EntityDataSource ID="GridDataSource" runat="server" EnableDelete="true" />

<asp:QueryExtender TargetControlID="GridDataSource" ID="GridQueryExtender" runat="server">
	<asp:DynamicFilterExpression ControlID="FilterRepeater" />
</asp:QueryExtender>
{% endcodeblock %}

The purpose of [_QueryableFilterRepeater_](http://msdn.microsoft.com/en-us/library/system.web.dynamicdata.queryablefilterrepeater.aspx) is to generate set of filters for a set of columns. It should contain [_DynamicFilter_](http://msdn.microsoft.com/en-us/library/system.web.dynamicdata.dynamicfilter.aspx) control which is the actual placeholder for a _FilterTemplate_ control. _QueryableFilterRepeater_ implements _IFilterExpressionProvider_ interface that is supported by [_QueryExtender_](http://msdn.microsoft.com/en-us/library/system.web.ui.webcontrols.queryextender.aspx) via [_DynamicFilterExpression_](http://msdn.microsoft.com/en-us/library/system.web.dynamicdata.dynamicfilterexpression.aspx) control.

{% codeblock lang:csharp %}
public interface IFilterExpressionProvider
{
  IQueryable GetQueryable(IQueryable source);
  void Initialize(IQueryableDataSource dataSource);
}
{% endcodeblock %}

The complete call sequence is represented on diagram below.

{% img center {% root_url %}/images/posts/dynamic-data-queryextender-interactions.png 'Sequence diagram showing QueryExtender interaction with Dynamic Data controls' 'Sequence diagram showing QueryExtender interaction with Dynamic Data controls' %}


# Building Configurable Alternative to QueryableFilterRepeater


As _QueryableFilterRepeater_ is creating filters automatically, the only thing we can do is to hide _DynamicFilter_ on client- or on server-side. To my mind it is not good idea, so a custom implementation of _IFilterExpressionProvider_ is needed. It should support the same item template model as in _QueryableFilterRepeater_ but with ability to add/remove filter controls between postbacks.

{% codeblock lang:csharp %}
[ParseChildren(true)]
[PersistChildren(false)]
public class DynamicFilterRepeater : Control, IFilterExpressionProvider
{
	private readonly List<IFilterExpressionProvider> filters = new List<IFilterExpressionProvider>();
	private IQueryableDataSource dataSource;

	IQueryable IFilterExpressionProvider.GetQueryable(IQueryable source)
	{
		return filters.Aggregate(source, (current, filter) => filter.GetQueryable(current));
	}

	void IFilterExpressionProvider.Initialize(IQueryableDataSource queryableDataSource)
	{
		Contract.Assert(queryableDataSource is IDynamicDataSource);
		Contract.Assert(queryableDataSource != null);

		if (ItemTemplate == null)
			return;
		dataSource = queryableDataSource;

		Page.InitComplete += InitComplete;
		Page.LoadComplete += LoadCompeted;
	}
}
{% endcodeblock %}

The only disappointing thing is the content generation of _DynamicFilter_ which is done on _Page.InitComplete_ event.


> Oleg Sych tried to change the situation, but [his suggestion](https://connect.microsoft.com/VisualStudio/feedback/details/588383/dont-use-page-initcomplete-in-queryablefilterrepeater-and-dynamicfilter-controls) is closed now and seems nothing will be changed. I just reposted his suggestion on [visualstudio.uservoice.com](http://visualstudio.uservoice.com/forums/121579-visual-studio/suggestions/2763785-don-t-use-page-initcomplete-in-queryablefilterrepe) in hope that this time, we will succeed.


To make things working, _DynamicFilter_ control should initialize itself via _EnsureInit_ method which is generally speaking responsible for _FitlerTempate_ lookup and loading. In other words to force the _DynamicFilter_ to generate its content this method should be called. The only way to do it is to use reflection, since _EnsureInit_ is private.

{% codeblock lang:csharp %}
private static readonly MethodInfo DynamicFilterEnsureInit;

static DynamicFilterRepeater()
{
	DynamicFilterEnsureInit = typeof (DynamicFilter).GetMethod("EnsureInit", BindingFlags.NonPublic | BindingFlags.Instance);
}

private void AddFilterControls(IEnumerable<string> columnNames)
{
	foreach (MetaColumn column in GetFilteredMetaColumns(columnNames))
	{
		DynamicFilterRepeaterItem item = new DynamicFilterRepeaterItem { DataItemIndex = itemIndex, DisplayIndex = itemIndex };
		itemIndex++;
		ItemTemplate.InstantiateIn(item);
		Controls.Add(item);

		DynamicFilter filter = item.FindControl(DynamicFilterContainerId) as DynamicFilter;
		if (filter == null)
		{
			throw new InvalidOperationException(String.Format(CultureInfo.CurrentCulture,
				"FilterRepeater '{0}' does not contain a control of type '{1}' with ID '{2}' in its item templates",
				ID,
				typeof(QueryableFilterUserControl).FullName,
				DynamicFilterContainerId));
		}
		filter.DataField = column.Name;

		item.DataItem = column;
		item.DataBind();
		item.DataItem = null;

		filters.Add(filter);
	}

	filters.ForEach(f => DynamicFilterEnsureInit.Invoke(f, new object[] { dataSource }));
}

private IEnumerable GetFilteredMetaColumns(IEnumerable filterColumns)
{
	return MetaTable.GetFilteredColumns()
		.Where(column => filterColumns.Contains(column.Name))
		.OrderBy(column => column.Name);
}

private class DynamicFilterRepeaterItem : Control, IDataItemContainer
{
	public object DataItem { get; internal set; }
	public int DataItemIndex { get; internal set; }
	public int DisplayIndex { get; internal set; }
}
{% endcodeblock %}

Another problem that should be solved - filter controls instantiation. As it was pointed before, all things in Dynamic Data that are connected with filtering are initialized at _Page.InitCompleted_ event. And if you want your dynamic filters to work, they should be instantiated before or at _InitComplete_ event. So far I see only one way to solve this - method _AddFilterControls_ should be called twice, first time to instantiate filter controls that were present on the page (_InitComplete_ event) and second time for newly added columns that are to be filtered (_LoadComplete_ event).

{% codeblock lang:csharp %}
private void InitComplete(object sender, EventArgs e)
{
	if (initComleted)
		return;

	addedOnInitCompleteFilters.AddRange(FilterColumns);
	AddFilterControls(addedOnInitCompleteFilters);

	initComleted = true;
}

private void LoadCompeted(object sender, EventArgs eventArgs)
{
	if (loadCompleted)
		return;

	AddFilterControls(FilterColumns.Except(addedOnInitCompleteFilters));

	loadCompleted = true;
}
{% endcodeblock %}


# Encapsulating DynamicFilterRepeater


_DynamicFilterRepeater_ is only a part of more general component though. Everything it does is rendering of filter controls and providing of filter expression. But to start working, _DynamicFilterRepeater_ needs two things - _IQueryableDataSource _and list of columns to be filtered. Since filtering across the website should be consistent and unified it would be good to encapsulate _DynamicFilterRepeater_ in _UserControl_ which will serve as HTML layout and a glue between page (with _IQueryableDataSource_, _QueryExtender_ and data source bound control) and _DynamicFilterRepeater_. In my example I chose _GridView_.

{% codeblock lang:aspx-cs%}
<asp:Label runat="server" Text="Add fitler" AssociatedControlID="ddlFilterableColumns" />
<asp:DropDownList runat="server" ID="ddlFilterableColumns" CssClass="ui-widget"
	AutoPostBack="True"
	ItemType="<%$ Code: typeof(KeyValuePair<string, string>) %>"
	DataValueField="Key"
	DataTextField="Value"
	SelectMethod="GetFilterableColumns"
	OnSelectedIndexChanged="ddlFilterableColumns_SelectedIndexChanged">
</asp:DropDownList>

<input type="hidden" runat="server" ID="FilterColumns" />
<dd:DynamicFilterRepeater runat="server" ID="FilterRepeater">
	<ItemTemplate>
		<div>
			<asp:Label ID="lblDisplayName" runat="server"
				Text='<%# Eval("DisplayName") %>'
				OnPreRender="lblDisplayName_PreRender" />
			<asp:DynamicFilter runat="server" ID="DynamicFilter" />
		</div>
	</ItemTemplate>
</dd:DynamicFilterRepeater>
{% endcodeblock %}

Remember I have mentioned about two-stage filter controls instantiation and a storage for list of filtered columns? Yes, this user control is a place where list of filtered columns could be stored. To get list of filtered columns before _Page.InitComplete_ event I'm using a little trick - the hidden input field serves as a storage for filtered columns list. Enforcing hidden input to have its ID generated on server makes it possible to retrieve value directly from _Page.Form_ collection at any stage of page lifecycle.

{% codeblock lang:csharp %}
public partial class DynamicFilterForm : UserControl
{
	public DynamicFilterRepeater FilterRepeater;
	public Type FitlerType { get; set; }

	[IDReferenceProperty(typeof(GridView))]
	public string GridViewID { get; set; }

	[IDReferenceProperty(typeof(QueryExtender))]
	public string QueryExtenderID { get; set; }

	private MetaTable MetaTable { get; set; }
	private GridView GridView { get; set; }
	protected QueryExtender GridQueryExtender { get; set; }

	protected override void OnInit(EventArgs e)
	{
		base.OnInit(e);
		MetaTable = MetaTable.CreateTable(FitlerType);

		GridQueryExtender = this.FindChildControl<QueryExtender>(QueryExtenderID);
		GridView = this.FindChildControl<GridView>(GridViewID);
		GridView.SetMetaTable(MetaTable);

		// Tricky thing to retrieve list of filter columns directly from hidden field
		if (!string.IsNullOrEmpty(Request.Form[FilterColumns.UniqueID]))
			FilterRepeater.FilterColumns.AddRange(Request.Form[FilterColumns.UniqueID].Split(','));

		((IFilterExpressionProvider)FilterRepeater).Initialize(GridQueryExtender.DataSource);
	}

	protected override void OnPreRender(EventArgs e)
	{
		FilterColumns.Value = string.Join(",", FilterRepeater.FilterColumns);
		base.OnPreRender(e);
	}
	// event handlers ommited
}
{% endcodeblock %}


# Conclusions


While this solution works, I'm a bit concerned about it. Existent infrastructure was in my way all the time I experimented with _IFilterExpressionProvider_, and I had to look deep inside the mechanisms of Dynamic Data to understand and find ways to come round its restrictions. And this leads me to only one conclusion - Dynamic Data was not designed to provide configurable filtering. So my answer on question about possibility of configurable filtering experience implementation with Dynamic Data is _yes_, but be careful what you wish for, since it was not designed for such kind of scenarios.

Here I did not mentioned how to save filters, but it is pretty simple, and all we need is to save somewhere associative array of "column-value" for a specific page. Complete source code is available on [GitHub](https://github.com/manekovskiy/Configurable-Dynamic-Data-Filters) and you will need Visual Studio 11 Beta with localdb setup to run sample project.

I would gladly accept criticism, ideas or just thoughts on this particular scenario. Share, do coding and have fun!