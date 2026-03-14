# On the Structural Non-Equivalence of XML and JSON

## ABSTRACT

JSON cannot faithfully represent XML data. This is because XML and JSON support vastly different types and declarations, but ultimately because unlike JSON, SGML and all its derivative languages including XML, cannot faithfully represent structured data themselves. In this document, we demonstrate the structural differences between XML and JSON and highlight the fundamental limitations in XML that make it a poor choice for representing structured data, using trivial examples.

## Introduction

Today, XML is still widely used as a data interchange format. Thus, it is reasonable to expect that it is compatible with, or at the very least comparable to JSON, a language specifically designed for this purpose. In this document we will assess XML from this particular perspective, as a language used to "store, transmit and reconstruct structured data".[^1]

Also, throughout this document we will refer to two language agnostic abstract structures that form the basis of structured data: ordinal and nominal structures.

**Ordinal structures** store data pieces (members) one after another, using a pre-determined order. Meaning is derived from this order thus the schema is not included with the data. They are also called indexed, ordered, positional or sequential structures, or arrays, lists, sets or sequences.

**Nominal structures** store data pieces (members) as associations, usually as key-value pairs, without regard to their order. Meaning is derived from the specific associations thus the schema is included with the data. They are also called associative, keyed, labeled, mapped or named structures, or dictionaries, maps or objects.

Finally, this document is a practical guide, not a theoretical or academic analysis. It also presumes that the reader has a basic working knowledge of both XML and JSON.

## Anatomy of XML

From a data standpoint we can say that XML supports only one default type `string` but it supports any custom type extending an abstract `element` type. This abstract type has three components:
- a mandatory type declaration, called the *tag name*;
- an optional nominal part, called the *attribute list*; and
- an optional ordinal part, called the *element content*.

The opening tag contains the tag name and the attribute list while the element content is between the opening and closing tags. Nominal members can only be of type `string`, while ordinal members can be an arbitrary mix of type `string` and any custom type. If there are no ordinal members, the opening tag can be merged with the closing tag, forming a so-called self-closing tag.

In addition, XML supports comments that conceptually act as self-closing tags with a single ordinal `string` member. Here is a brief overview of the main parts of XML:

```xml
<tag-name attribute-name="attribute-value">element content</tag-name>
<self-closing-tag/>
<!-- comment -->
```

> [!NOTE]
> For simplicity, in many examples we will use the abstract element type (an element without a tag name):
> ```xml
> <_>
> ```

## Element Variations

There are exactly four variations that an XML element can have: nominal part only, ordinal part only, both nominal and ordinal parts and neither nominal nor ordinal part.

**1. Nominal part only:** The element has attributes but no element content. This is equivalent to an object:

```xml
<_ foo="bar"></_>
```
```json
{"foo": "bar"}
```

> [!NOTE]
> If there is no ordinal part the opening and closing tags can be merged. This is called a self-closing tag. We will use this syntax to simplify our examples:
> ```xml
> <_ foo="bar"/>
> ```

> [!CAUTION]
> The official name for a self-closing tag is empty element, which is demonstrably incorrect terminology. 'Empty' elements can, in fact hold data as nominal members, they just don't hold any ordinal members. We will avoid this terminology as it is highly misleading, especially in glaring examples like this (and even more so in HTML):
> ```xml
> <img src="image.png"/>
> ```

**2. Ordinal part only:** The element has element content but no attributes. This is equivalent to an array:

```xml
<_>foo</_>
```
```json
["foo"]
```

**3. Both nominal and ordinal parts:** The element has attributes and also element content. There is no equivalent in JSON, we can only approximate this variation with a combination of an array and an object, but it is ambiguous:

```xml
<_ foo="bar">baz</_>
```

1. An array nested into an object. In this case we need a surrogate property and a naming convention to avoid collision with an existing attribute. A common approach is to use a name that would be invalid as an attribute:

    ```json
    {
      "foo": "bar",
      "@children": ["baz"]
    }
    ```

2. An object nested into an array. In this case we need a convention to avoid ambiguity and define how we manipulate the indices within the element content:

    ```json
    [
      {"foo": "bar"},
      "baz"
    ]
    ```

In both cases the resulting graph is different than the original because the element holds ordinal and nominal members within a single construct.

In technical terms, for each graph vertex we always need two. In practical terms the JSON tree is twice as big and twice as deep as its XML counterpart:

For example, an XML graph with 3 vertices and a depth of 3:

```xml
<_>
  <_>
    <_></_>
  </_>
</_>
```

would equate to a JSON graph with 6 vertices and a depth of 6:

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

or at the minimum to a JSON graph with 6 vertices and a depth of 4:

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

True equivalence would be a new hybrid structure in JSON that merges, not combines, an ordinal and a nominal structure:

```pseudo-json
(
    "foo": "bar",
    "baz"
)
```

> [!NOTE]
> This is exactly how arrays work in JS, we just can't define them via a single literal. In JS arrays are regular objects, therefore we can set arbitrary properties on them. In fact, array indices can work the same as any other object property:
> ```js
> const arr = ["foo"]; //define ordinal members
> arr.bar = "baz"; //define nominal members
>
> arr[0]; //"foo"
> arr["0"]; //"foo" (works as any other property)
> arr["bar"]; //"baz" (stored directly on the array)
> ```

**4. Neither nominal nor ordinal part:** The element has no attributes or any element content. The equivalence here is also ambiguous: this can be either an empty array, an empty object or even other JSON data types. This is the true empty element. It is perfectly valid in XML and it does have a use case, for example the boolean type in plist files:

```xml
<true/>
```
```json
true
```







## Elements

The naive assumption is that an element is equivalent to an object. This is only true for a specific subset of elements, namely if they contain only nominal members:

```xml
<_ foo="bar"/>
```
```json
{"foo": "bar"}
```

But it is entirely possible that an element contains only ordinal members, in which case it is equivalent to an array instead:

```xml
<_>foo</_>
```
```json
["foo"]
```

Then, of course there is the third subset of elements that have both ordinal and nominal members. Here, the next naive assumption is that a combination of an array and an object would suffice, but this would produce a different graph since an element holds ordinal and nominal members within a single construct.

Suppose we have a complex XML element:

```xml
<_ foo="bar">baz</_>
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

Regardless of our choice, both options produce a different graph: instead of a single graph vertex we always need two. In practical terms, the JSON graph will be twice as big and twice as deep as the original XML graph.

For example, an XML graph with 3 vertices and a depth of 3:

```xml
<_>
  <_>
    <_></_>
  </_>
</_>
```

would equate to a JSON graph with 6 vertices and a depth of 6:

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

or at the minimum to a JSON graph with 6 vertices and a depth of 4:

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

If we want true equivalence we would need to add a new hybrid structure to JSON that merges, not combines, an ordinal and a nominal structure. This would also work for all three subsets of XML elements:

```pseudo-json
(
    "foo": "bar",
    "baz"
)
```

> [!NOTE]
> This is exactly how arrays work in JS, we just can't define them via a single literal. In JS arrays are regular objects, therefore we can set arbitrary properties on them. In fact, array indices can work the same as any other object property:
> ```js
> const arr = ["foo"]; //define ordinal members
> arr.bar = "baz"; //define nominal members
>
> arr[0]; //"foo"
> arr["0"]; //"foo" (works as any other property)
> arr["bar"]; //"baz" (stored directly on the array)
> ```

## Tag Names

XML tag names are essentially type declarations. We can demonstrate this by comparing an XML element to a JS class. A simple XML element like this:

```xml
<person name="John Doe" age="30" />
```

Is structurally equivalent to this:

```js
class person {
  name = "John Doe";
  age = 30;
}
```

The only difference is functionality: the first annotates an actual instance with type information while the second merely describes the shape of this type with some default values. However, in both cases the `person` declaration is neither a key nor a value, but a third, distinct concept. This distinction is important, because common 'best' practices appear to lack any understanding of the concept of types and routinely confuse type declarations with keys or values.

For example, 'best' practice dictates[^2] that we should avoid this:

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

This is bad advice because the second example not only conflates types with keys, it also pushes nominal data into an ordinal model, breaking our data model at a fundamental level.

To demonstrate this, here is what the original author wanted to achieve:

```json
{
  "date": {
    "day": 10,
    "month": 1,
    "year": 2008
  },
  "to": "Tove",
  "from": "Jani",
  "heading": "Reminder",
  "body": "Don't forget me this weekend!"
}
```

And here is what they actually ended up with instead:

```json
{
  "@type": "note",
  "@children": [
    "\n  ",
    {
      "@type": "date",
      "@children": [
        "\n    ",
        {
          "@type": "day",
          "@children": ["10"]
        },
        "\n    ",
        {
          "@type": "month",
          "@children": ["01"]
        },
        "\n    ",
        {
          "@type": "year",
          "@children": ["2008"]
        },
        "\n  "
      ]
    },
    "\n  ",
    {
      "@type": "to",
      "@children": ["Tove"]
    },
    "\n  ",
    {
      "@type": "from",
      "@children": ["Jani"]
    },
    "\n  ",
    {
      "@type": "heading",
      "@children": ["Reminder"]
    },
    "\n  ",
    {
      "@type": "body",
      "@children": ["Don't forget me this weekend!"]
    },
    "\n"
  ]
}
```

To really drive this point home, let's plug the 'best' practice XML into JS (or any other programming language) and try to access the year for example. How should we do it?

```js
note.getElementsByTagName("year")[0].textContent; //like this?
note.getElementsByTagName("date")[0].getElementsByTagName("year").textContent; //or this?
note.children[0].children[2].textContent; //or maybe this?
```
No matter what we try, it is always a guessing game:
- Is it the first `<year>` element in the entire document we need?
- Is it the first `<date>` element, then its first `<year>` element we need?
- Is the `<date>` element the first child of the root element, then its third child the `<year>` element we need?

In contrast, this is what the author wanted to achieve:

```js
note.date.year;
```

This 'advice' stems from a completely unrelated limitation, namely that attribute values cannot branch. It would be much better advice not to use XML as a data interchange format because it cannot nest nominal data and all workarounds are nothing short of disastrous.

## Keys or Values
Unfortunately, JSON does not support any form of explicit type declarations, so ironically we have to map tag names either to a key or to a value. Again, the naive assumption is that now we can revert the ordinal structure back to a nominal structure as it was originally intended, but it is only possible in a rare and exceptional situation and even that has major downsides.

Suppose we promote tag names to keys on the parent object. This is possible with extremely simple elements:

```xml
<_><a/></_>
```
```json
{
  "a": {}
}
```

However, the root element has no parent, the surrogate keys can clash with existing attributes, it is entirely possible that an element has multiple children with the same tag name and that it has text nodes along with its element children.

1. Surrogate root element alters our graph:

```xml
<a/>
```
```json
{
  "a": {}
}
```

2. Surrogate tag properties clash with attribute properties and the workaround is ambiguous:

```xml
<_ a="foo"><a/></_>
```
```json
{
  "a": "foo",
  "@a": {}
}
```

Or:

```json
{
  "@a": "foo",
  "a": {}
}
```

3. Multiple children with the same tag name needs a surrogate array that alters our graph once again:

```xml
<_><a/><a/></_>
```
```json
{
  "a": [
    {},
    {}
  ]
}
```

4. Text nodes mixed with child elements don't even have a possible workaround:

```xml
<_ text="foo"><a>bar</a>baz</_>
```

- The best option is to add a surrogate property for text nodes as well, but even if we avoid clashing with an existing attribute, once we separate child elements and text nodes we cannot reconstruct their original order (the conversion is lossy):

```json
{
  "text": "foo",
  "@text": "baz",
  "a": {
    "@text": "bar"
  }
}
```

- And if we keep their order, we cannot promote tag names to properties (we are back to square one):

```json
{
  "@children": [
    {
      "@type": "a",
      "@children": ["foo"]
    },
    "bar"
  ]
}
```

This is exactly where badgerfish, a popular XML-to-JSON convention, gave up too.[^3]

If we want true equivalence we would need to add type declarations to JSON. This is an interesting idea because it also demonstrates how badly the 'best' practice XML actually performs:

```pseudo-json
note [
  date [
    day ["10"],
    month ["01"],
    year ["2008"]
  ],
  to ["Tove"],
  from ["Jani"],
  heading ["Reminder"],
  body ["Don't forget me this weekend!"]
]
```

## Whitespace

But an even bigger issue is the treatment of whitespace in XML, which is inconsistent across contexts. In the attribute list around names and values it is always treated as formatting and thus discarded, but in the element content it is always treated as actual content and fully preserved by default. This has far reaching consequences.

For example, the following two examples are equivalent:

```xml
<_ foo="bar"/>
```
```xml
<_
  foo
    =
      "bar"
        />
```

But the following two are not:

```xml
<_>foo</_>
```
```xml
<_>
  foo
</_>
```

This becomes apparent if we faithfully convert the last example to an array:

```json
["\n  foo\n"]
```

But beyond inconsistency and non-equivalence the biggest issue is data corruption. Since XML provides no boundary between content and formatting, whitespace added simply to make the code readable for humans bleeds into and alters the actual data. Once content and formatting are mixed together it is practically impossible to separate the two.

> [!CAUTION]
> Whitespace bleeding is a significant issue in HTML as well. One common example is when inline-block elements are indented, for example when describing a horizontal menu:
> ```html
> <style>li {display: inline-block}</style>
> <ul>
>   <li>foo</li>
>   <li>bar</li>
>   <li>baz</li>
> </ul>
> ```
> This renders as `foo` `bar` `baz` instead of `foobarbaz` because formatting whitespace between elements is preserved as content. To keep the semblance of indentation but remove unwanted whitespace, one approach exploits the very inconsistency we just described:
> ```html
> <ul
>   ><li>foo</li
>   ><li>bar</li
>   ><li>baz</li
> ></ul>
> ```
> We leave it to the reader to decide whether this or any similar trick can be classified as actual solution to this problem.

In contrast, JSON clearly defines a boundary between content and formatting: whitespace added inside a string is fully preserved and whitespace added outside a string is fully discarded. There is no possibility of formatting whitespace corrupting the data:

```json
[
  "foo"    ,
                "bar"
  ,    "baz"
    ]
```

At best, whitespace bleeding makes XML unsuitable for storing structured data, at worst, it is a major design flaw of the language because it prevents XML from fulfilling one of its stated objectives: it is either "human-legible and reasonably clear" or "easy to process", but not both.[^4]

## CONCLUSION

> [!IMPORTANT]
> XML by design not only forces us to deliberately misuse type declarations and model types, it also corrupts our data.

[^1]: https://en.wikipedia.org/wiki/XML
[^2]: https://www.w3schools.com/xml/xml_attributes.asp
[^3]: http://www.sklar.com/badgerfish/
[^4]: https://www.w3.org/TR/xml/#sec-origin-goals



## Converting Text Nodes (drop this?)

This leads to another unsolvable dilemma:
- If we treat pure text and child elements the same inside the element content, this example would equate to a JSON like this:
- 
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
- 
```json
{
  "@children": [
    {
```

Even in trivial cases it is very difficult, or even impossible, to separate whitespace we want to keep as data from whitespace we want to discard as formatting.



- mapping string, custom hybrids to null, boolean, number, string, array, object
- plist dict to object? Other types?

## Converting Comments (merge with text nodes/whitespace?)
## Converting the Prolog (maybe not important)