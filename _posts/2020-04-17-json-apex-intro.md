---
layout: post
title: "Working with JSON in Apex: A Primer"
---

> This post is adapted from a [community wiki](https://salesforce.stackexchange.com/questions/302034/how-do-i-get-started-working-with-json-in-apex) I created for [Salesforce Stack Exchange](https://salesforce.stackexchange.com).

Apex provides multiple routes to achieving JSON serialization and deserialization of data structures. This post summarizes use cases and capabilities of *untyped* deserialization, *typed* (de)serialization, manual implementations using `JSONGenerator` and `JSONParser`, and tools available to help support these uses. The objective is to provide an introduction and overview of routes to working effectively with JSON in Apex, and links to other resources.

## Summary

Apex can serialize and deserialize JSON to strongly-typed Apex classes and also to generic collections like `Map<String, Object>` and `List<Object>`. In most cases, it's preferable to define Apex classes that represent data structures and utilize typed serialization and deserialization with `JSON.serialize()`/`JSON.deserialize()`. However, some use cases require applying untyped deserialization with `JSON.deserializeUntyped()`.

The `JSONGenerator` and `JSONParser` classes are available for manual implementations and should be used only where automatic (de)serialization is not practicable, such as when keys in JSON are reserved words in Apex, or when low-level access is required.

The key documentation references are the [`JSON`](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_class_System_Json.htm) class in the Apex Developer Guide and the section [JSON Support](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_methods_system_json_overview.htm). Other relevant documentation is linked from those pages.

## Complex Types in Apex and JSON

JSON offers maps (or objects) and lists as its complex types. JSON lists map to Apex `List` objects. JSON objects can map to *either* Apex classes, with keys mapping to instance variables, or Apex `Map` objects. Apex classes and collections can be intermixed freely to construct the right data structures for any particular JSON objective. 

Throughout this post, we'll use the following JSON as an example:

```json
{
	"errors": [ "Data failed validation rules" ],
	"message": "Please edit and retry",
	"details": {
		"record": "001000000000001",
		"record_type": "Account"
	}
}
```

This JSON includes two levels of nested objects, as well as a list of primitive values. 

## Typed Serialization with `JSON.serialize()` and `JSON.deserialize()`

The methods `JSON.serialize()` and `JSON.deserialize()` convert between JSON and typed Apex values. When using `JSON.deserialize()`, you must specify the type of value you expect the JSON to yield, and Apex will attempt to deserialize to that type. `JSON.serialize()` accepts both Apex collections and objects, in any combination that's convertible to legal JSON.

These methods are particularly useful when converting JSON to and from  Apex classes, which is in most circumstances the preferred implementation pattern. The JSON example above can be represented with the following Apex class:

```apex

public class Example {
	public List<String> errors;
	public String message;
	
	public class ExampleDetail {
		Id record;
		String record_type;
	}
	
	public ExampleDetail details;
}
```

To parse JSON into an `Example` instance, execute

```apex
Example ex = (Example)JSON.deserialize(jsonString, Example.class);
```

Alternately, to convert an `Example` instance into JSON, execute

```apex
String jsonString = JSON.serialize(ex);
```

Note that nested JSON objects are modeled with one Apex class per level of structure. It's not required for these classes to be inner classes, but it is a common implementation pattern. Apex only allows one level of nesting for inner classes, so deeply-nested JSON structures often convert to Apex classes with all levels of structure defined in inner classes at the top level.

`JSON.serialize()` and `JSON.deserialize()` can be used with Apex collections and classes in combination to represent complex JSON data structures. For example, JSON that stored `Example` instances as the values for higher-level keys:

```json
{
	"first": { /* Example instance */ },
	"second": { /* Example instance */},
	/* ... and so on... */
}
```

can be serialized from, and deserialized to, a `Map<String, Example>` value in Apex.

For more depth on typed serialization and deserialization, review the [`JSON`](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_class_System_Json.htm) class documentation. Options are available for:

- Suppression of `null` values
- Pretty-printing generated JSON
- Strict deserialization, which fails on unexpected attributes

## Untyped Deserialization with `JSON.deserializeUntyped()`

In some situations, it's most beneficial to deserialize JSON into Apex collections of primitive values, rather than into strongly-typed Apex classes. For example, this can be a valuable approach when the structure of the JSON may change in ways that aren't compatible with typed deserialization, or which would require features that Apex does not offer like algebraic or union types.

Using the `JSON.deserializeUntyped()` method yields an `Object` value, because Apex doesn't know at compile time what type of value the JSON will produce. It's necessary when using this method to *typecast* values pervasively.

Take, for example, this JSON, which comes in multiple variants tagged by a `"scope"` value:

```json
{
	"scope": "Accounts",
	"data": {
		"payable": 100000,
		"receivable": 40000
	}
}
```
or

```
{
	"scope": {
		"division": "Sales",
		"organization": "International"
	},
	"data": {
		"closed": 400000
	}
}

```

JSON input that varies in this way cannot be handled with strongly-typed Apex classes because its structure is not uniform. The values for the keys `scope` and `data` have different types. 

This kind of JSON structure can be deserialized using `JSON.deserializeUntyped()`. That method returns an `Object`, an untyped value whose actual type at runtime will reflect the structure of the JSON. In this case, that type would be `Map<String, Object>`, because the top level of our JSON is an object. We could deserialize this JSON via

```apex
Map<String, Object> result = (Map<String, Object>)JSON.deserializeUntyped(jsonString);
```

The untyped nature of the value we get in return cascades throughout the structure, because Apex doesn't know the type at compile time of *any* of the values (which may, as seen above, be heterogenous) in this JSON object.

As a result, to access nested values, we must write defensive code that inspects values and typecasts at each level. The example above will throw a `TypeException` if the resulting type is not what is expected.

To access the data for the first element in the above JSON, we might do something like this:

```apex
Object result = JSON.deserializeUntyped(jsonString);

if (result instanceof Map<String, Object>) {
    Map<String, Object> resultMap = (Map<String, Object>)result;
	if (resultMap.get('scope') == 'Accounts' &&
	    resultMap.get('data') instanceof Map<String, Object>) {
		Map<String, Object> data = (Map<String, Object>)resultMap.get('data');
	
		if (data.get('payable') instanceof Integer) {
			Integer payable = (Integer)data.get('payable');
			
			AccountsService.handlePayables(payable);
		} else {
			// handle error
		}
	} else {
		// handle error
	}
} else {
	// handle error
}
```

While there are other ways of structuring such code, including catching `JSONException` and `TypeException`, the need to be defensive is a constant. Code that fails to be defensive while working with untyped values is vulnerable to JSON changes that produce exceptions and failure modes that won't manifest in many testing practices. Common exceptions include `NullPointerException`, when carelessly accessing nested values, and `TypeException`, when casting a value to the wrong type.

## Manual Implementation with `JSONGenerator` and `JSONParser`

The [`JSONGenerator`](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_class_System_JsonGenerator.htm#!) and [`JSONParser`](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_class_System_JsonParser.htm#apex_class_System_JsonParser) classes allow your application to manually construct and parse JSON. 

Using these classes entails writing explicit code to handle each element of the JSON. Using `JSONGenerator` and `JSONParser` typically yields much more complex (and much longer) code than using the built-in serialization and deserialization tools. However, it may be required in some specific applications. For example, JSON that includes Apex reserved words as keys may be handled using these classes, but cannot be deserialized to native classes because reserved words (like `type` and `class`) cannot be used as identifiers.

As a general guide, use `JSONGenerator` and `JSONParser` only when you have a specific reason for doing so. Otherwise, strive to use native serialization and deserialization, or use external tooling to generate parsing code for you (see below).
 
## Generating Code with `JSON2Apex`

[JSON2Apex](https://json2apex.herokuapp.com/) is an [open source](https://github.com/superfell/json2apex) Heroku application. JSON2Apex allows you to paste in JSON and generates corresponding Apex code to parse that JSON. The tool defaults to creating native classes for serialization and deserialization. It automatically detects many situations where explicit parsing is required and generates `JSONParser` code to deserialize JSON to native Apex objects.

JSON2Apex does not solve every problem related to using JSON, and generated code may require revision and tuning. However, it's a good place to start an implementation, particularly for users who are just getting started with JSON in Apex.