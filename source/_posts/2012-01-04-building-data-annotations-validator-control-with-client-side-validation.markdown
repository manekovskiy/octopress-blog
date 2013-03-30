---
comments: true
date: 2012-01-04 21:16:45
layout: post
slug: building-data-annotations-validator-control-with-client-side-validation
title: Building Data Annotations Validator Control With Client-Side Validation
wordpress_id: 47
categories:
- Development
tags:
- ASP.NET
- Data Annotations
- Validation
---

When I had worked on ASP.NET MVC project I really liked how input is validated with Data Annotations attributes. And when I had to return to the Web Forms, and write a simple form with some validation, I was wondering how I lived before with standard validator controls. For me, it was never convenient, when I had to write an enormous amount of server tags just to state that "this is required field which accepts only numbers in specified range...". Yes, there is nothing terrible in declaration of two or three validation controls instead of one. But, if I had a choice, I would like to write only one validator per field and keep all input validation logic as far as I can from the UI markup. And being a developer the code-only approach is most natural for me.

_System.ComponentModel.DataAnnotations_ namespace was introduced in .NET 3.5, and now its classes are supported by wide range of technologies like WPF, Silverlight, RIA Services, ASP.NET MVC, ASP.NET Dynamic Data but not in Web Forms. I thought that someone had already implemented ready-to-use Validator Control with client-side validation, but after searching the Web and most popular open source hosting services I found nothing. Ok, not nothing, but implementations what I have found lacked client-side validation and had some other issues. So I decided to write my own Data Annotations Validator that will also support client-side validation.

## Creating Data Annotations Validator Control

### Server-Side

As I wanted to achieve compatibility with existing validation controls (new validator is not a replacement for an old ones, it is just an addition to them), it was decided to inherit from _BaseValidator_. This class does all necessary initialization on both client- and server-sides and exposes all necessary methods for overriding.

{% codeblock lang:csharp %}
public class DataAnnotationsValidator : BaseValidator
{ }
{% endcodeblock %}
  
First of all _EvaluateIsValid_ method of _BaseValidator_ should be overridden.  

{% codeblock lang:csharp %}
protected override bool EvaluateIsValid()
{
	object value = GetControlValidationValue(ControlToValidate);
	foreach (ValidationAttribute validationAttribute in ValidationAttributes)
	{
		// Here, we will try to convert value to type specified on RangeAttibute.
		// RangeAttribute.OperandType should be either IConvertible or of built in primitive types
		var rangeAttibute = validationAttribute as RangeAttribute;
		if (rangeAttibute != null)
		{
			value = Convert.ChangeType(value, rangeAttibute.OperandType);
		}

		if (validationAttribute.IsValid(value)) continue;

		ErrorMessage = validationAttribute.FormatErrorMessage(DisplayName);
		return false;
	}

	return true;
}
{% endcodeblock %}

The only interesting aspect of this method is line 16. I'm using _FormatErrorMessage_ method of _ValidationAttribute_ to use all the goodness like support of resources and proper default error message formatting. So, now there is no need to invent something with error messages.

Next thing to deal with is where to get _ValidationAttributes_ collection. There is _System.Web.DynamicData.MetaTable_ class that could be used to retrieve attributes. It was introduced in the first versions of ASP.NET Dynamic Data and now in 4.0 version of Dynamic Data, _MetaTable_ has a static method _[CreateTable](http://msdn.microsoft.com/en-us/library/dd989458.aspx)_ which accepts _Type_ as input parameter. Why using _MetaTable_, why not retrieve attributes of _Type_ directly from _PropertyInfo_ for specified property name? Because _MetaTable_ also supports retrieving of custom attributes that are applied to property through [_MetadataTypeAttribute_](http://msdn.microsoft.com/en-us/library/system.componentmodel.dataannotations.metadatatypeattribute.aspx) and merges attributes applied to property both in metadata class and entity class. And again, why inventing something new when everything that is needed is right here?

{% codeblock lang:csharp %}
MetaTable.CreateTable(ObjectType))
	.GetColumn(PropertyName).Attributes
	.OfType<ValidationAttribute>()
{% endcodeblock %}

Now let's look a bit into the future - if we place ObjectType property into _DataAnnotationsValidator_, it means that we should specify this property for every validator control on page. This is redundancy and leads to copy-pasting which is not acceptable. Lets step aside and create _MetadataSource_ control that will act like metadata provider for validators on page.  

{% codeblock lang:csharp %}
public class MetadataSource : Control
{
	public Type ObjectType { get; set; }

	private MetaTable metaTable;
	private MetaTable MetaTable
	{
		get { return metaTable ?? (metaTable = MetaTable.CreateTable(ObjectType)); }
	}

	public IEnumerable GetValidationAttributes(string property)
	{
		return MetaTable.GetColumn(property).Attributes.OfType();
	}

	public string GetDisplayName(string objectProperty)
	{
		var displayAttribute = MetaTable.GetColumn(objectProperty).Attributes
			.OfType()
			.FirstOrDefault();

		return displayAttribute == null ? objectProperty : displayAttribute.GetName();
	}
}
{% endcodeblock %}
  
Here I also thought about [DisplayAttribute](http://msdn.microsoft.com/en-us/library/system.componentmodel.dataannotations.displayattribute.aspx) which is used to format default error message. Now how _ObjectType_ of _MetadataSouce_ should be specified? Well, we can do it programmatically on Page_Load or do it.. programmatically with [CodeExpressionBuilder](http://weblogs.asp.net/infinitiesloop/archive/2006/08/09/The-CodeExpressionBuilder.aspx) to keep all control setup in one place.  

{% codeblock lang:html %}
<dav:MetadataSource runat="server"
	ID="msFoo"
	ObjectType="<%$ Code: typeof(Foo) %>" />
{% endcodeblock %}
  
Now, with existence of _MetadataSource_ all fields of _DataAnnotationsValidator_ are initialized in the OnInit method  
    
{% codeblock lang:csharp %}
protected override void OnLoad(EventArgs e)
{
	base.OnLoad(e);

	if (!ControlPropertiesValid())
		return;

	MetadataSource = this.FindChildControl(MetadataSourceID);

	ValidationAttributes = MetadataSource.GetValidationAttributes(ObjectProperty);
	DisplayName = MetadataSource.GetDisplayName(ObjectProperty);
}
{% endcodeblock %}

This is all what was needed to provide server-side validation.

### Client-Side

First of all, lets check what standard validator controls could be replaced with _Data Annotations_ attributes.

Data Annotations Attribute | Standard Validator Control 
-------------------------- | -------------------------- 
RequiredAttribute          | RequiredFieldValidator     
StringLengthAttribute      | -                          
RegularExpressionAttribute | RegularExpressionValidator 
-                          | CompareValidator           
RangeAttribute             | RangeValidator             

I have no ideas how to replace _CompareValidator_ and I don't think it is so critical and necessary to think on it. Time to look how standard validator controls are working on the client side.

Every validator that works on the client side should override _AddAttributesToRender_ method of _BaseValidator_ class. This method adds some fields to resulting javascript object. For example, _RequiredFieldValidator_ adds _evaluationfunction_ and _initialvalue_ fields.

{% codeblock lang:csharp %}
protected override void AddAttributesToRender(HtmlTextWriter writer) {
    base.AddAttributesToRender(writer);
    if (RenderUplevel) {
        string id = ClientID;
        HtmlTextWriter expandoAttributeWriter = (EnableLegacyRendering) ? writer : null;
        AddExpandoAttribute(expandoAttributeWriter, id, "evaluationfunction", "RequiredFieldValidatorEvaluateIsValid", false);
        AddExpandoAttribute(expandoAttributeWriter, id, "initialvalue", InitialValue);
    }
}
{% endcodeblock %}
 
And resulting javascript block for _RequiredFieldValidator_ will look next:  
  
{% codeblock lang:js %}
<script type="text/javascript">
//<![CDATA[
var rfvSampleTextBox = document.all ? document.all["rfvSampleTextBox"] : document.getElementById("rfvSampleTextBox");
rfvSampleTextBox.controltovalidate = "tbSampleTextBox";
rfvSampleTextBox.evaluationfunction = "RequiredFieldValidatorEvaluateIsValid";
rfvSampleTextBox.initialvalue = "";
//]]>
</script>
{% endcodeblock %}
  
After examining source code of standard validator controls I found that every control sets _evaluationfunction_ field that states a name for a javascript function that actually performs validation on the client-side. RequiredFieldValidator evaluation function is represented below.  

{% codeblock lang:js %}
function RequiredFieldValidatorEvaluateIsValid(val) {
    return (ValidatorTrim(ValidatorGetValue(val.controltovalidate)) != ValidatorTrim(val.initialvalue))
}
{% endcodeblock %}
  
The _val_ parameter is a validator object that was initialized with all the fields that were set in the _AddAttributesToRender_ method. Plain and simple, if you need to supply your validator on client-side with some information override _AddAttributesToRender_ and add what you want. To replace standard validators _DataAnnotationsValidator_ is doing a little trick - it adds all standard _evaluationfunction_ names, error messages and all necessary fields that are used by standard validation functions. Evaluation function of _DataAnnotationsValidator:_  

{% codeblock lang:js %}
<script type="text/javascript">
//<![CDATA[
function DataAnnotationsValidatorIsValid(val) {
    var functionsToEvaluate = val.validatorFunctions.split(';;');
	var errorMessages = val.errorMessages.split(';;');

    for (var funcIndex in functionsToEvaluate) {
        var result = eval(functionsToEvaluate[funcIndex] + "(val)");
        if(result === false) {
			val.errormessage = errorMessages[funcIndex];
			val.innerText = errorMessages[funcIndex];
            return false;
        }
    }
    return true;
}
//]]>
</script>
{% endcodeblock %}
  
This function is registered in the _OnPreRender_ stage of control lifecycle.  

{% codeblock lang:csharp %}
protected override void OnPreRender(EventArgs e)
{
	base.OnPreRender(e);

	if (RenderUplevel)
	{
        var scriptManager = ScriptManager.GetCurrent(Page);
        if (scriptManager != null && scriptManager.IsInAsyncPostBack)
            ScriptManager.RegisterClientScriptResource(this, GetType(), DAValidationScriptFileName);
        else
    		Page.ClientScript.RegisterClientScriptResource(GetType(), DAValidationScriptFileName);
	}
}
{% endcodeblock %}
  
The only thing that remains is to get list of fields and values that are needed for validation functions. Every _Data Annotation_ validation attribute will have an _Adapter_ class that stores an array of _ClientValidationRule_ classes. _ClientValidationRule_ is just a container for storing javascript object _field names_ and _evaluationfunction_.  

{% codeblock lang:csharp %}
public class ClientValidationRule
{
    public string EvaluationFunction { get; set; }
    public Dictionary<string, object> Parameters { get; private set; }

    public string ErrorMessage { get; set; }

    public ClientValidationRule()
    {
        Parameters = new Dictionary<string, object>();
        EvaluationFunction = string.Empty;
    }
}
{% endcodeblock %}
 
And _ValidationAttributeAdapter_ acts like a bridge between existing _ValidationAttibute_ and its _ClientValidationRules._  

{% codeblock lang:csharp %}
internal class ValidationAttributeAdapter
{
    protected ValidationAttribute Attribute { get; set; }
    protected string DisplayName { get; set; }
    protected string ErrorMessage { get; set; }

    public ValidationAttributeAdapter(ValidationAttribute attribute, string displayName)
    {
        Attribute = attribute;
        DisplayName = displayName;
        ErrorMessage = Attribute.FormatErrorMessage(DisplayName);
    }

    public virtual IEnumerable<ClientValidationRule> GetClientValidationRules()
    {
        return Enumerable.Empty<ClientValidationRule>();
    }
}
{% endcodeblock %}
  
All _ValidationAttributeAdapter_ classes are registered within _ValidationAttributeAdapterFactory_ in a _Dictionary_.  
   
{% codeblock lang:csharp %}
private static readonly Dictionary<Type, Func<ValidationAttribute, string, ValidationAttributeAdapter>> PredefinedCreators
    = new Dictionary<Type, Func<ValidationAttribute, string, ValidationAttributeAdapter>>
    {
        {
            typeof(RangeAttribute),
            (attribute, displayName) => new RangeAttributeAdapter(attribute as RangeAttribute, displayName)
        }, {
            typeof(RegularExpressionAttribute),
            (attribute, displayName) => new RegularExpressionAttributeAdapter(attribute as RegularExpressionAttribute, displayName)
        }, {
            typeof(RequiredAttribute),
            (attribute, displayName) => new RequiredAttributeAdapter(attribute as RequiredAttribute, displayName)
        }, {
            typeof (StringLengthAttribute),
            (attribute, displayName) => new StringLengthAttributeAdapter(attribute as StringLengthAttribute, displayName)
        }
    };

public static ValidationAttributeAdapter Create(ValidationAttribute attribute, string displayName)
{
    Debug.Assert(attribute != null, "attribute parameter must not be null");
    Debug.Assert(!string.IsNullOrWhiteSpace(displayName), "displayName parameter must not be null, empty or whitespace string");

    // Added suport for ValidationAttribute subclassing. See http://davalidation.codeplex.com/workitem/695
    var baseType = attribute.GetType();
    Func<ValidationAttribute, string, ValidationAttributeAdapter> predefinedCreator;
    do
    {
        if (!PredefinedCreators.TryGetValue(baseType, out predefinedCreator))
            baseType = baseType.BaseType;
    }
    while (predefinedCreator == null && baseType != null && baseType != typeof(Attribute));

    return predefinedCreator != null 
        ? predefinedCreator(attribute, displayName) 
        : new ValidationAttributeAdapter(attribute, displayName);
}
{% endcodeblock %}
    
As I said previously, idea was borrowed directly from ASP.NET MVC, so if you are familiar with its validation mechanisms you don't need to learn how it works here. Approach here is almost identical to ASP.NET MVC which was [described](http://bradwilson.typepad.com/blog/2010/10/mvc3-unobtrusive-validation.html) well by Brad Wilson. As in ASP.NET MVC _DAValidation.IClientValidatable_ interface exposes an extension point and we can now create a custom validation attribute, implement _IClientValidatable_ interface, write validation function or mix existing ones and get both server- and client-side validation. There is a great set of validation attributes - [Data Annotations Extensions](http://dataannotationsextensions.org/) created by [Scott Kirkland](http://weblogs.asp.net/srkirkland/) so it is only client function must be changed in order to use them with _DataAnnotationsValidator_.  

{% codeblock lang:csharp %}
public interface IClientValidatable
{
    IEnumerable<ClientValidationRule> GetClientValidationRules();
}
{% endcodeblock %}

And that's all, now we have fully functional control that makes validation with Data Annotations possible in the ASP.NET Web Forms universe.

Complete source code could be found on [Codeplex](http://davalidation.codeplex.com/). There you can download latest version of control and example project.
