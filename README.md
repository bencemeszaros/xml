# Why XML Is a Fundamentally Flawed Choice for Data Interchange

## ABSTRACT

XML is a bad choice as a data interchange format because it cannot faithfully represent structured data. This is because it does not support nesting inside nominal structures, it misuses fundamental data models and conflates types with properties. In addition, it lacks a clear separation between formatting whitespace and actual data, leading to data corruption. In this document we demonstrate these inherent structural flaws by comparing XML to JSON, using trivial examples.

## Introduction

Today, XML is still widely used as a data interchange format. Thus, it is reasonable to expect that it is compatible with, or at the very least comparable to JSON, a language specifically designed for this purpose. In this document we will assess XML from this particular perspective, as a language used to <a href="https://en.wikipedia.org/wiki/XML">"store, transmit and reconstruct structured data."</a>

Throughout this document we will refer to two language agnostic abstract structures that form the basis of structured data: ordinal and nominal structures.

**Ordinal structures** store data pieces (members) one after another, using a pre-determined order. Meaning is derived from this order, thus the schema is not included with the data. They are also called indexed, ordered, positional or sequential structures, or arrays, lists, sets or sequences.

**Nominal structures** store data pieces (members) as associations, usually as key-value pairs, without regard to their order. Meaning is derived from the specific associations, thus the schema is included with the data. They are also called associative, keyed, labeled, mapped or named structures, or dictionaries, maps or objects.

Finally, we would like to emphasize that this document is a practical guide, not a theoretical or academic analysis. It also presumes that the reader has a basic working knowledge of both XML and JSON.

## Anatomy of XML

From a data standpoint we can say that XML supports only one default type `string` but it supports any custom type extending an abstract `element` type. This abstract type has three components:
- a mandatory type declaration, called the *tag name*;
- an optional nominal part, called the *attribute list*; and
- an optional ordinal part, called the *element content*.

```xml
<tag-name attribute-name="attribute-value">element content</tag-name>
```

> [!NOTE]
> For simplicity, in many examples we will use the abstract element type (an element without a tag name):
> ```xml
> <_></_>
> ```

## Element Variations

Based on the model of ordinal and nominal structures, there are exactly four possible variations of an XML element:
- ordinal part only;
- nominal part only;
- both ordinal and nominal parts; and
- neither ordinal nor nominal part.

### Ordinal part only

The first option is when the element has element content but no attributes. This is equivalent to an array:

```xml
<_>foo</_>
```
```json
["foo"]
```

### Nominal part only

The second option is when the element has attributes but no element content. This is equivalent to an object:

```xml
<_ foo="bar"></_>
```
```json
{"foo": "bar"}
```

> [!NOTE]
> If there is no ordinal part we can use a self-closing tag. We will use this syntax to simplify our examples:
> ```xml
> <_ foo="bar"/>
> ```

> [!CAUTION]
> The official name for a self-closing tag is empty element, which is demonstrably incorrect terminology. "Empty" elements can, in fact hold data as nominal members, they just don't hold any ordinal members. We will avoid this terminology as it is highly misleading, especially in glaring examples like this (and even more so in HTML):
> ```xml
> <img src="image.png"/>
> ```

### Both ordinal and nominal parts

The third option is when the element has both element content and attributes. There is no equivalent in JSON, we can only approximate this variation with a combination of an array and an object, but it is ambiguous:

```xml
<_ foo="bar">baz</_>
```

- One option is to nest an array into an object. In this case we need a surrogate property and a naming convention to avoid collision with an existing attribute. A common approach is to use a name that would be invalid as an attribute:

```json
{
  "foo": "bar",
  "@children": ["baz"]
}
```

- The other option is to nest an object into an array. In this case we need a convention to avoid ambiguity and define how we manipulate the indices within the element content:

```json
[
  {"foo": "bar"},
  "baz"
]
```

However, in both cases the resulting graph is different than the original because the XML element holds ordinal and nominal members within a single construct. In technical terms, for each graph vertex we always need two. In practical terms the JSON tree is twice as big and twice as deep as its XML counterpart.

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
> This is exactly how arrays work in JS, we just can't define them via a single literal. In JS, arrays are regular objects, therefore we can set arbitrary properties on them. In fact, array indices can work the same as any other object property:
> ```js
> const arr = ["foo"]; //define ordinal members
> arr.bar = "baz"; //define nominal members
>
> arr[0]; //"foo"
> arr["0"]; //"foo" (works as any other property)
> arr["bar"]; //"baz" (stored directly on the array)
> ```

### Neither ordinal nor nominal part

The fourth and last option is when the element doesn't have element content or attributes. The equivalence here is also ambiguous: it can be either an empty array, an empty object or even other JSON data types. This is the true empty element. It is perfectly valid in XML and it does have a use case, for example the boolean type in <a href="https://developer.apple.com/library/archive/documentation/General/Conceptual/DevPedia-CocoaCore/PropertyList.html#//apple_ref/doc/uid/TP40008195-CH44-SW2" target="_blank">plist files</a>:

```xml
<true/>
```
```json
true
```

## Nesting Limitation and Workarounds

Now that we've established how XML implements ordinal and nominal structures, we can demonstrate how this implementation fails. Generally speaking the problem is, XML unnecessarily treats ordinal and nominal structures differently.

One such example is that ordinal members can be of any type, while nominal members can only be of type `string`. This means that nominal members cannot be complex types, therefore cannot branch. This limitation has far reaching consequences.

Suppose we have the following simple XML fragment:

```xml
<_ name="John Doe"/>
```

If we want to store the name as a complex type like `<name first="John" last="Doe"/>`, we cannot simply plug this into the `name` attribute, we have to use a hack: we have to flatten it into a single `string`, explode it into multiple attributes, or misuse an ordinal structure for this data, none of which is a viable solution.

### Flatten complex data into a single `string`
This is essentially a new language with custom encoders/decoders shoehorned into XML and it even has to carefully mix, escape or avoid using `<`, `>`, `'`, `"` and `&`:

```xml
<_ name='first: "John"; last: "Doe"'/>
```

> [!NOTE]
> This is actually what inline CSS is in HTML. The following example is a deeply nested example of a nominal structure that cannot be expressed in HTML:
> ```html
> <div style="transition: background-color 0.3s ease, transform 0.3s ease, opacity 0.5s linear;"></div>
> ```
> ```json
> {
>   "style": {
>     "transition":  [
>       [
>         "background-color",
>         "0.3s",
>         "ease"
>       ],
>       [
>         "transform",
>         "0.3s",
>         "ease"
>       ],
>       [
>         "opacity",
>         "0.5s",
>         "linear"
>       ]
>     ]
>   }
> }
> ```

### Explode complex data into multiple attributes
The second option is to abuse attribute names instead. This workaround isn't much better either, especially if the depth of the original structure grows. But at least it is within XML:

```xml
<_ spouse-personal-name-current-legal-first="Jane" spouse-personal-name-current-legal-last="Doe"/>
```

### Misuse an ordinal structure for nominal data
The third option is to simply forget about attributes altogether and misuse the element content for this type of data. This is often the recommended "solution" for this limitation, even though it forces nominal data into an ordinal model that breaks our data model at a fundamental level.

Suppose we convert the previous nominal data into an ordinal structure and add it to the element content:

```xml
<_>
  <name>
    <first-name>John</first-name>
    <last-name>Doe</last-name>
  </name>
</_>
```

This looks harmless, until we try to access for example the last name. Instead of simply this:

```js
root.name.lastName; //"Doe"
```

We are forced to do something like this:

```js
root.children[0].children[1].textContent; //maybe "Doe", maybe not
root.getElementsByTagName("last-name")[0].textContent; //maybe "Doe", maybe not
```

Regardless of what we try, we can never be sure that the first, second or nth child or element with the requested tag name is going to be what we need, it is essentially a constant guessing game. In addition, we always need to use a final `textContent` or something similar to actually get the data and even without this step our code is already practically unreadable.

This problem is very subtle and highly deceptive since it is easy to encode a nominal structure into an ordinal structure, but it is extremely hard to decode it back into the original nominal structure. If you only care about encoding, you'll probably never realize this fundamental problem.

To put it simply, we can either use a nominal structure with direct access but no nesting, or use an ordinal structure with nesting but no direct access. XML forces us to shift our question from "do we want ordinal or nominal structure" to "do we want nesting or not".

## Type Misuse

Pushing nominal data into an ordinal structure has another unintended consequence: it forces us to misuse types as well.

In XML, tag names are essentially type declarations. We can demonstrate this by comparing an XML element to a JS class. A simple XML element like this:

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

The only difference is functionality: the first annotates an actual instance with type information while the second merely describes the shape of this type with some default values. However, in both cases the `person` declaration is neither a key nor a value, but a third, distinct concept. This distinction is important, because the common recommendation to prefer element content over attributes appears to lack any understanding of the concept of types and routinely confuse type declarations with keys or values.

To demonstrate this, let's convert XML to JSON but this time preserve tag names as well. Since JSON doesn't support any form of explicit type declarations, we do have to map tag names either to a key or to a value. If XML tag names could be considered keys or values, this conversion would yield the original data structure back, but this is obviously not the case.

Suppose we promote tag names to keys on the parent object. This is possible with extremely simple elements:

```xml
<_><a/></_>
```
```json
{
  "a": {}
}
```

However, even here, let alone in ay other cases, we face a litany of issues: the surrogate keys can clash with existing attributes, the root element has no parent, it is entirely possible that an element has multiple children with the same tag name and text nodes can freely mix with element children.

- To avoid surrogate tag properties to clash with attribute properties we have to alter either the original attributes or the new tag properties:

```xml
<_ a="foo"><a/></_>
```
```json
{
  "@a": "foo",
  "a": {}
}
```
```json
{
  "a": "foo",
  "@a": {}
}
```

- To handle the tag name of the root element we need a surrogate root element that alters our graph:

```xml
<a/>
```
```json
{
  "a": {}
}
```

- To handle multiple children with the same tag name we need a surrogate array too that alters our graph once again:

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

- And if text nodes mix with child elements we are stuck without any solution:

```xml
<_ text="foo"><a>bar</a>baz</_>
```

The best we can do here is to add a surrogate property for text nodes as well, but even if we avoid clashing with an existing attribute once again, once we separate child elements and text nodes we cannot reconstruct their original order (the conversion is lossy):

```json
{
  "text": "foo",
  "@text": "baz",
  "a": {
    "@text": "bar"
  }
}
```

And if we keep their order, we cannot promote tag names to properties (we are back to square one):

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

This is exactly where <a href="http://www.sklar.com/badgerfish/">badgerfish</a>, a popular XML-to-JSON convention, gave up too.

Another way to demonstrate how types differ from keys and values is by simply adding type declarations to JSON. This would bring JSON much closer to XML and clearly show that the XML model is not at all what its author wanted to achieve:

```xml
<xml>
  <name>
    <first-name>John</first-name>
    <last-name>Doe</last-name>
  </name>
</xml>
```
```pseudo-json
xml [
  name [
    first-name ["John"],
    last-name ["Doe"]
  ]
]
```

## Whitespace Bleeding

Whitespace handling is another area where XML treats ordinal and nominal structures differently. In the attribute list around names and values it is always treated as formatting and thus discarded, but in the element content it is always treated as actual content and fully preserved by default. This also has far reaching consequences.

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

To put it simply, at best, whitespace bleeding makes XML unsuitable for storing structured data, at worst, it is a major design flaw of the language itself because it prevents XML from fulfilling one of <a href="https://www.w3.org/TR/xml/#sec-origin-goals">its stated objectives</a>: it is either "human-legible and reasonably clear" or "easy to process", but not both.

## "Best" Practice

A common <a href="https://www.w3schools.com/xml/xml_attributes.asp">"best" practice example</a> demonstrates all previous points simultaneousy. This example dictates that we should avoid this:

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

This is simply bad advice because the second example misuses an ordinal structure for nominal data, conflates types with keys and litters data with code formatting whitespace. To really drive this point home, here is what the original author wanted to achieve:

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

We can give much better advice:

> [!IMPORTANT]
> XML should be avoided as a data interchange format because it does not allow nesting inside nominal structures, it forces nominal data into ordinal structures, it conflates types with properties and corrupts data with code-formatting whitespace—none of which has a viable workaround.