---
layout: post
title: How to use named values with JSON content in Azure API Management Policies
comments: true
---

When you want to use a `Named Value` in Azure API Management that contains a JSON string inside an APIM policy expression, you might run into some problems.
In this blogpost, I'm going to cover how I worked around them and finally fixed them.

# What do I want to achieve ?

I have an API defined in API Management for which multiple products are defined.  Depending on the product that is used when calling the API (based on the subscription key), I want to pass in a specific value to the backend.

My first approach was to define a `named value` in APIM for every product that is defined.  The name of each `named value` followed a specific convention: `some-value-<productname>`.  For instance, `some-value-product-a`, `some-value-product-b`, etc...

I then created an APIM policy that would use this named value, like this:

```json
<set-header name="myheader" exist-action="override">
    <value>{{some-value-@(context.Product.Id)}}</value>    
    </value>
</set-header>
```

and I thought I was done.  Unfortunately it is not possible to dynamically construct and use named values.  I learned that named values are not evaluated at run-time when executing a policy.  Instead, the `named value` reference is replaced by it's value when the APIM policy is saved, and therefore it is obvious that you cannot dynamically reference `named value` instances.

# JSON to the rescue ?

To work around this, I thought of creating one single `named value` that contains a JSON object which contains a property for every defined APIM product, like this:

```json
{
    "product-a": "somevalue",
    "product-b": "anothervalue",
    "product-c": "foobar"
}
```

The idea was to retrieve and parse this named value in the `set-header` policy:

```json
<set-header name="myheader" exist-action="override">
    <value>
    @{
        var contents = "{{my-named-value}}";

        var parsedJson = (Newtonsoft.Json.Linq.JObject)JsonConvert.DeserializeObject(contents);

        return (string)parsedJson[context.Product.Id];
    }
    </value>
</set-header>
```

But, when saving this policy the following error was given:

> Error in element 'set-header' on line xx, column yy: Unterminated string literal. Strings that start with a quotation mark (") must be terminated before the end of the line. However, strings that start with @ and a quotation mark (@") can span multiple lines.

It looks like APIM gets confused because there are quotes inside the contents of the `named value`.

# Work around using a context variable

The final solution is to make use of the `set-variable` policy.  For some reason, it is possible to assign the `named value` that contains a JSON object to a context variable, and use that context variable inside a `set-header` policy.

The final solution looks like this:

```json
<set-variable name="myvariable" value="{{my-named-value}}" />

<set-header name="myheader" exists-action="override">
    <value>
    @{
        var contents = (string)context.Variables["myvariable"];
        var minifiedJson = contents.Replace("\n", "").Replace("\r", "").Replace("\\", "");

        var parsedJson = (Newtonsoft.Json.Linq.JObject)JsonConvert.DeserializeObject(minifiedJson);

        if( parsedJson.ContainsKey(context.Product.Id) == false )
        {
            return "some-fallback-value";
        }
        return (string)parsedJson[context.Product.Id];
    }
    </value>
</set-header>
```

Pay attention to the fact that you need to do some string replacements before you're able to parse the JSON contents of the `named value`.

Hope this helps!
Frederik