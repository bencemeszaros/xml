# On the Structural Non-Equivalence of XML and JSON

## ABSTRACT
JSON cannot faithfully represent XML data. This is because of the mismatch between the types and declarations they support, but ultimately because unlike JSON, XML and by extension SGML and all SGML-like languages are inherently incapable of faithfully representing structured data in the first place. In this document, we demonstrate the structural differences between XML and JSON and highlight the fundamental design flaws in XML that make it a poor choice for storing structured data, using trivial examples.

## Introduction
- artificial nodes
- artificial properties

## Anatomy of XML

XML supports one built-in type `string`, any number of custom types extending a base element type and comments. In contrast, JSON supports six built–in types: `null` `boolean` `number` `string` `array` and `object`, no custom types and no comments.

- tag name, attribute list, element content between the opening and closing tags, two data types (default and custom)
- anatomy of JSON: six data types (default only)

## Elements
The naive assumption is that an element is equivalent to an array or an object. This is only true in special edge cases, specifically if the XML element has either ordinal members (element content) or nominal members (attribute list) exclusively.

- An XML element with only an ordinal part (element content only) would equate to an array:
```xml
<__>foo</__>
```
```json
["foo"]
```

- And an XML element with only a nominal part (attribute list only) would equate to an object:
```xml
<__ foo="bar" />
```
```json
{"foo": "bar"}
```

Aside from the fact that processing a JSON with alternating arrays and objects would be highly confusing, if an element has both ordinal and nominal members this convention wouldn't even work. For generic XML elements neither a single array nor a single object would suffice. Here, the next naive assumption is that a combination of an array and an object would work, but the resulting combination would not be equivalent to the original element, since an element holds ordinal and nominal members within a single construct.

Suppose we have a complex XML element:
```xml
<__ foo="bar">baz</__>
```

If we try to approximate this element with a combination of an array and an object we only have two options: we can either nest an array into an object or an object into an array.

- If we nest an array into an object we also need a surrogate property and a naming convention to avoid clashing with an existing attribute. A common approach is to use a name that would be invalid as an attribute:
```json
{
    "foo": "bar",
    "@children": ["baz"]
}
```

- If we nest an object into an array the result is ambiguous and we also need a convention to define how we manipulate the indices within the element content:
```json
[
    {"foo": "bar"},
    "baz"
]
```

Regardless of our choice, both options produce a different graph: instead of a single graph vertex we always need two. (In practical terms, the JSON graph is twice as big as the XML graph and its depth keeps increasing (or even outright doubling)):

For example, an XML graph with 3 vertices and a depth of 3:
```xml
<__>
  <__>
    <__></__>
  </__>
</__>
```

- would equate either to a JSON graph with 6 vertices and a depth of 6:
```json
{
  "@children": [
    {
      "@children": [
        {
          "@children": []
        }
      ]
    }
  ]
}
```

- or at the minimum to a JSON graph with 6 vertices and a depth of 4:
```json
[
  {},
  [
    {},
    [
      {}
    ]
  ]
]
```

If we want true equivalence we would need to add a new hybrid structure to JSON that merges, not combines, an ordinal and a nominal structure. This would also work for all three types of XML elements:
```pseudo-json
(
    "foo": "bar",
    "baz"
)
```

> [!NOTE]
> This is exactly how arrays work in JS, we just can't define them via a single literal. In JS (almost) everything is an object, including arrays, so we can set arbitrary properties on them. In fact, array indices can work the same as any other property:
>
> ```js
> const arr = ["foo"]; //define ordinal members
> arr.bar = "baz"; //define nominal members
>
> arr[0]; //"foo"
> arr["0"]; //"foo" (works as any other property)
> arr["bar"]; //"baz" (stored directly on the array)
> ```

## Whitespace
The treatment of whitespace in XML is inconsistent across contexts. In the attribute list around names and values it is always treated as formatting and thus discarded, but in the element content it is always treated as actual content and fully preserved by default. This has far reaching consequences.

For example, the following two examples are equivalent:
```xml
<__ foo="bar"/>
```
```xml
<__
  foo
    =
      "bar"
        />
```

but the following two are not:
```xml
<__>foo</__>
```
```xml
<__>
  foo
</__>
```

This becomes apparent if we convert the last example to a JSON array:
```json
["\n  foo\n"]
```

But beyond inconsistency and non-equivalence the biggest issue is data corruption. Since XML provides no boundary between content and formatting, whitespace added simply to make the code readable for humans bleeds into and alters the actual data. Once formatting and data is mixed together, it is practically impossible to separate the two.

> [!NOTE]
> Whitespace bleeding is a significant issue in HTML as well. One common example is when inline-block elements are indented, for example when describing a horizontal menu:
> ```html
> <style>li {display: inline-block}</style>
> <ul>
>   <li>foo</li>
>   <li>bar</li>
>   <li>baz</li>
> </ul>
> ```
> This renders as `foo` `bar` `baz` instead of `foobarbaz` because formatting whitespace is preserved between elements. To keep the semblance of indentation but remove unwanted whitespace, one approach exploits the very inconsistency we just described:
> ```html
> <ul
>   ><li>foo</li
>   ><li>bar</li
>   ><li>baz</li
> ></ul>
> ```
> It is difficult to classify this, or any similar approach, as an actual solution to the problem.

In contrast, JSON clearly defines a boundary between content and formatting: whitespace added inside a string is fully preserved, whitespace added outside a string is fully discarded, there is no possibility of formatting whitespace corrupting the data:

```json
[
  "foo"    ,    "bar"
  ,    "baz"
    ]
```

At best, whitespace bleeding makes XML unsuitable for storing structured data, at worst, it is a major design flaw of the language because it prevents XML from fulfilling its stated objectives: it is either "human-legible and reasonably clear" or "easy to process", but not both.[^1]

## Text nodes
This leads to another unsolvable dilemma:
- If we treat pure text and child elements the same inside the element content, this example would equate to a JSON like this:

```json
[
  "\n  ",
  ["Dogs"],
  "\n  chase\n  ",
  ["cats"],
  "\n"
]
```

- And if we somehow separate them, we won’t be able to reconstruct the original order:

```json
{
  "@children": [
    {
```

Even in trivial cases it is very difficult, or even impossible, to separate whitespace we want to keep as data from whitespace we want to discard as formatting.

## Tag names

XML tag names are essentially type declarations. We can demonstrate this by comparing an XML element to a JS class. A simple XML element like this:

```xml
<person name="John Doe" age="30" />
```

is structurally equivalent to this:

```js
class person {
  name = "John Doe";
  age = 30;
}
```

They only differ in functionality: the first annotates an actual instance with type information while the second merely describes the shape of this type with some default values. However, in both cases the `person` declaration is neither an identifier nor a value, but a third, distinct concept. This is important, because common "best" practices appear to lack any understanding of the concept of types and routinely confuse type declarations with identifiers (properties).

For example, "best" practice dictates that we should avoid this:

```xml
<note
  day="10"
  month="01"
  year="2008"
  to="Tove"
  from="Jani"
  heading="Reminder"
  body="Don't forget me this weekend!"
></note>
```

in favor of this:

```xml
<note>
  <date>
    <day>10</day>
    <month>01</month>
    <year>2008</year>
  </date>
  <to>Tove</to>
  <from>Jani</from>
  <heading>Reminder</heading>
  <body>Don't forget me this weekend!</body>
</note>
```

This is good advice only in the sense that any other alternative would be much worse. In reality, the second example not only conflates type declarations with identifiers, it also forces nominal data into an ordinal model, breaking our data model at a fundamental level. This 'advice' stems from a completely unrelated limitation, namely that attribute values cannot branch, which is a fundamental flaw in the very design of XML.

> Simply put, XML by design forces us to deliberately misuse type declarations and model types.

Unfortunately, JSON does not support any form of explicit type declarations, so ironically we have to map tag names either to an identifier or to a value.




- mapping string, custom hybrids to null, boolean, number, string, array, object
- plist dict to object? Other types?

## Comments
## Prolog

<table>
  <thead>
    <tr>
      <th>Name</th>
      <th>Age</th>
      <th>Role</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Alice</td>
      <td>30</td>
      <td>Developer</td>
    </tr>
    <tr>
      <td>Bob</td>
      <td>25</td>
      <td>Designer</td>
    </tr>
    <tr>
      <td>Charlie</td>
      <td>28</td>
      <td>Manager</td>
    </tr>
  </tbody>
</table>

[^1]: https://www.w3.org/TR/xml/#sec-origin-goals
