---
comments: true
date: 2012-09-20 19:22:55
layout: post
slug: adding-client-side-validation-support-for-phoneattribute-or-fighting-the-look-behind-in-javascript
title: Adding Client-Side Validation Support for PhoneAttribute or Fighting the Lookbehind in JavaScript
wordpress_id: 174
categories:
- Development
tags:
- .NET 4.5
- Data Annotations
- JavaScript
- Validation
---

Today, I was working on JavaScript implementation of validation routine for [PhoneAttribute](http://msdn.microsoft.com/en-us/library/system.componentmodel.dataannotations.phoneattribute.aspx) in context of my hobby project [DAValidation](http://amanek.com/building-data-annotations-validator-control-with-client-side-validation/). Examining the sources of .NET 4.5 showed that the validation is done via regular expression:

{% img center /images/posts/phone-validation-regexp-with-non-supported-lookbehind-highlighted.png 'Unsupported lookbehind part of phone validation regexp pattern' 'Unsupported lookbehind part of phone validation regexp pattern' %}

And here is the problem - the pattern uses lookbehind feature [that is not supported in JavaScript](http://www.regular-expressions.info/lookaround.html).
Quote from [regular-expressions.info:](http://www.regular-expressions.info)


> Finally, flavors like [JavaScript](http://www.regular-expressions.info/javascript.html), [Ruby](http://www.regular-expressions.info/ruby.html) and [Tcl](http://www.regular-expressions.info/tcl.html) do not support lookbehind at all, even though they do support lookahead.


This lookbehind is used to match the "+" sign at the beginning of string, i. e. check the existence of the prefix. To make this work in JavaScript pattern should be reversed and lookbehind assertion should be replaced with lookahead (replace prefix check to suffix). And that's it! The resulting pattern is:

{% codeblock lang:js %}
^(\d+\s?(x|\.txe?)\s?)?((\)(\d+[\s\-\.]?)?\d+\(|\d+)[\s\-\.]?)*(\)([\s\-\.]?\d+)?\d+\+?\((?!\+.*)|\d+)(\s?\+)?$
{% endcodeblock %}

As a proof here is test html page:

{% codeblock lang:html %}
<html> 
    <head> 
        <title>Phone Number RegExp Test Page</title> 
    </head> 
    <body> 
        <script> 
            function validateInput() {                 
                var phoneRegex = new RegExp("^(\\d+\\s?(x|\\.txe?)\\s?)?((\\)(\\d+[\\s\\-\\.]?)?\\d+\\(|\\d+)[\\s\\-\\.]?)*(\\)([\\s\\-\\.]?\\d+)?\\d+\\+?\\((?!\\+.*)|\\d+)(\\s?\\+)?$", "i"); 
                 
                var input = document.getElementById("tbPhone"); 
                var value = input.value.split("").reverse().join(""); 
                alert(phoneRegex.test(value)); 
            }             
        </script> 
         
        <input type="text" id="tbPhone" /> 
        <button onclick="javascript:testPhone()">Validate</button> 
    </body> 
</html>
{% endcodeblock %}

While working on pattern reversing I was using my favorite regular expressions building and testing tool [Expresso](http://www.ultrapico.com/Expresso.htm). Also, a great article of Steven Levithan [Mimicking Lookbehind in JavaScript](http://blog.stevenlevithan.com/archives/mimic-lookbehind-javascript) helped to look deeper and actually find the right solution of the problem.



PS. Now, as I finally finished adding [support for new .NET 4.5 validation attributes](http://davalidation.codeplex.com/workitem/701) the new version of [DAValidation](http://davalidation.codeplex.com/) will be published soon. Stay tuned ;)
