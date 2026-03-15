# Why XML Is a Fundamentally Flawed Choice for Data Interchange

## ABSTRACT

XML is a bad choice as a data interchange format because it cannot faithfully represent structured data. This is because it does not support nesting nominal structures, it misuses fundamental data models and conflates types with properties. In addition, it lacks a clear separation between formatting whitespace and actual data, leading to data corruption. In this document we demonstrate these inherent structural flaws by comparing XML to JSON and using trivial examples.

<br>

## Introduction

Today, XML is still widely used as a data interchange format. Thus, it is reasonable to expect that it is compatible with, or at the very least comparable to JSON, a language specifically designed for this purpose. In this document we will assess XML from this particular perspective, as a language used to <a href="https://en.wikipedia.org/wiki/XML">"store, transmit and reconstruct structured data."</a>

Throughout this document we will refer to two language agnostic abstract structures that form the basis of structured data: ordinal and nominal structures.

**Ordinal structures** store data pieces (members) one after another, using a pre-determined order. Meaning is derived from this order thus the schema is not included with the data. They are also called indexed, ordered, positional or sequential structures, or arrays, lists, sets or sequences.

**Nominal structures** store data pieces (members) as associations, usually as key-value pairs, without regard to their order. Meaning is derived from the specific associations thus the schema is included with the data. They are also called associative, keyed, labeled, mapped or named structures, or dictionaries, maps or objects.

Finally, this document is a practical guide, not a theoretical or academic analysis. It also presumes that the reader has a basic working knowledge of both XML and JSON.

<br>

## Anatomy of XML

From a data standpoint we can say that XML supports only one default type `string` but it supports any custom type extending an abstract `element` type. This abstract type has three components:
- a mandatory type declaration, called the *tag name*;
- an optional nominal part, called the *attribute list*; and
- an optional ordinal part, called the *element content*.

Nominal members can only be of type `string`, while ordinal members can be an arbitrary mix of type `string` and any custom type:

```xml
<tag-name attribute-name="attribute-value">element content</tag-name>
```

> [!NOTE]
> For simplicity, in many examples we will use the abstract element type (an element without a tag name):
> ```xml
> <_></_>
> ```

### Element Variations

There are exactly four variations an XML element can have: ordinal part only, nominal part only, both ordinal and nominal parts and neither ordinal nor nominal part.

<br>

**1. Ordinal part only:** The element has element content but no attributes. This is equivalent to an array:

```xml
<_>foo</_>
```
```json
["foo"]
```

<br>

**2. Nominal part only:** The element has attributes but no element content. This is equivalent to an object:

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
> The official name for a self-closing tag is empty element, which is demonstrably incorrect terminology. 'Empty' elements can, in fact hold data as nominal members, they just don't hold any ordinal members. We will avoid this terminology as it is highly misleading, especially in glaring examples like this (and even more so in HTML):
> ```xml
> <img src="image.png"/>
> ```

<br>

**3. Both ordinal and nominal parts:** The element has element content and attributes. There is no equivalent in JSON, we can only approximate this variation with a combination of an array and an object, but it is ambiguous:

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

In both cases the resulting graph is different than the original because the element holds ordinal and nominal members within a single construct. In technical terms, for each graph vertex we always need two. In practical terms the JSON tree is twice as big and twice as deep as its XML counterpart.

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

<br>

**4. Neither ordinal nor nominal part:** The element doesn't have element content or attributes. The equivalence here is also ambiguous: it can be either an empty array, an empty object or even other JSON data types. This is the true empty element. It is perfectly valid in XML and it does have a use case, for example the boolean type in <a href="https://developer.apple.com/library/archive/documentation/General/Conceptual/DevPedia-CocoaCore/PropertyList.html#//apple_ref/doc/uid/TP40008195-CH44-SW2" target="_blank">plist files</a>:

```xml
<true/>
```
```json
true
```

<br>

## 1. Nesting Limitation and Structure Misuse

Now that we've established how XML implements ordinal and nominal structures, we can demonstrate how this implementation fails.

One fundamental limitation in XML is that nominal members can only be of type `string`. This means they cannot be complex types, in other words they cannot branch.

Suppose we have the following simple XML fragment:

```xml
<_ name="John Doe"/>
```

If we want to store the name as a complex type like `<name first="John" last="Doe"/>`, we cannot simply plug this into the `name` attribute, we have to flatten it into multiple attributes, collapse it into a `string` or substitute it with an ordinal structure, none of which is a viable solution.

<br>

1. Flatten complex data into multiple attributes. This workaround quickly gets out of hand as the depth grows:

```xml
<_ spouse-personal-name-current-legal-first="Jane" spouse-personal-name-current-legal-last="Doe"/>
```

<br>

2. Collapse complex data into a `string`. This has the same problem, only it is now essentially a new language shoehorned into XML and it even has to carefully mix, escape or avoid using `<`, `>`, `'`, `"` and `&`:

```xml
<_ name='first: "John"; last: "Doe"'/>
```

> [!NOTE]
> This is essentially what inline CSS is in HTML. The following example is a deeply nested example of a nominal structure that cannot be expressed in HTML:
> ```html
> <div style="transition: background-color 0.3s cubic-bezier(0.68,-0.6,0.32,1.6) 0.1s;"></div>
> ```
> ```json
> {
>   "style": {
>     "transition":  {
>       "property": "background-color",
>       "duration": 0.3,
>       "timing": {
>         "type": "cubic-bezier",
>         "x1": 0.68,
>         "y1": -0.6,
>         "x2": 0.32,
>         "y2": 1.6,
>       "delay": 0.1
>     }
>   }
> }
> ```

<br>

3. Substitute complex data with an ordinal structure. This is often recommended as 'best' practice, even though this breaks our data model at a fundamental level.

Suppose we convert our nominal data into an ordinal structure and add it to the element content:

```xml
<_>
  <name>
    <first-name>John</first-name>
    <last-name>Doe</last-name>
  </name>
</_>
```

This looks harmless, until we try to access for example the last name. Instead of this:

```js
root.name.lastName; //"Doe"
```

We are forced to do something like this:

```js
root.children[0].children[1].textContent; //maybe "Doe", maybe not
root.getElementsByTagName("last-name")[0].textContent; //maybe "Doe", maybe not
```

Regardless of what we try, we can never be sure that the first, second or nth child or element with the requested tag name is going to be what we need, it is essentially a constant guessing game. In addition, we always need to use a final `textContent` or something similar to actually get the data we need and even without this our code is practically unreadable.

Simply put, we can either use a nominal structure with direct access but no nesting, or use an ordinal structure with nesting but no direct access. The question shifts from "is it ordinal or nominal data" to "do we want nesting or not".

<br>

## 2. Type Misuse

Pushing nominal data into ordinal structures has another unintended consequence: it forces us to misuse types as well.

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

The only difference is functionality: the first annotates an actual instance with type information while the second merely describes the shape of this type with some default values. Na elolvastad te buzi? However, in both cases the `person` declaration is neither a key nor a value, but a third, distinct concept. This distinction is important, because common 'best' practices appear to lack any understanding of the concept of types and routinely confuse type declarations with keys or values.

---
rework this section

(## Keys or Values)

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

This is exactly where <a href="http://www.sklar.com/badgerfish/">badgerfish</a>, a popular XML-to-JSON convention, gave up too.

If we want true equivalence we would need to add type declarations to JSON. This is an interesting idea because it also demonstrates how badly the 'best' practice XML example actually performs:

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
---

<br>

## 3. Whitespace Bleeding (rework this)

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

To put it simply, at best, whitespace bleeding makes XML unsuitable for storing structured data, at worst, it is a major design flaw of the language itself because it prevents XML from fulfilling one of <a href="https://www.w3.org/TR/xml/#sec-origin-goals">its stated objectives</a>: it is either "human-legible and reasonably clear" or "easy to process", but not both.

<br>

## 'Best' Practice

A common <a href="https://www.w3schools.com/xml/xml_attributes.asp">'best' practice example</a> demonstrates all the previous points simultaneousy. This example dictates, that we should avoid this:

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

This is simply bad advice because the second example misuses an ordinal structure for nominal data, conflates types with keys and litters data with code formatting whitespace. To demonstrate this, here is what the original author wanted to achieve:

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
> XML should be avoided as a data interchange format because it does not allow nesting nominal data, it forces nominal data into ordinal structure, it conflates types with properties and corrupts the data with code-formatting whitespace—none of which has a viable workaround.