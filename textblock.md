# Textblock

- [Textblock](#textblock)
	- [Introduction](#introduction)
		- [Textblock Logic](#textblock_logic)
		- [Unicode](#unicode)
		- [Bidi Properties](#bidi_properties)
		- [Usages](#usages)
- [Edje Entry](#edje_entry)
- [Elementary Entry (Widget)](#elementary_entry)

## <a name=introduction></a>Introduction
This document serves as a complement to the existing [Textblock Documentation and Tutorial](http://docs.enlightenment.org/auto/efl/group__Evas__Object__Textblock.html#Evas_Object_Textblock_Tutorial "") in the official EFL documents.

The Textblock object is an extensive tool for handling complex texts, such as ones that have multiple styles and multilined. More complex features offer text alignement, wrapping and item embedding e.g. icons.
Currently, this is the favorable way of handling text in EFL, and demand for more a comprehensive explanation had led to the forming of this document.

### The Textblock Object
EFL's Evas Textblock object offers many features, which are highly popular in most text applications nowadays. Currently, it supports the following:
- Multilines
- Wrapping (word/char/mixed) and ellipsis
- Alignment (auto/manual)
- Styles
- Formatting
- Item embedding (e.g. icons)
- Bidirectional text (BiDi)

Moreover, the Evas Textblock object has been extended to [Edje Entry](http://docs.enlightenment.org/auto/edje/group__Edje__Text__Entry.html) and [Elementary Entry](http://docs.enlightenment.org/auto/elementary/group__Entry.html) to offer even more features.
Bidirectional text is under development and is aimed to comply with current Unicode standards.
More on those in the following sections.

### Markup
Setting a text in Textblock is performed by using the evas_object_textblock_text_markup_set/prepend methods.
The textblock object introduces a light [markup language](http://en.wikipedia.org/wiki/Markup_language) for using the special text formatting it offers. For example, the following markup:
```c
<font_weight=bold>Hello</font_weight> World
```
will produce a [<b>Hello</b> World] text:<br>
![example-tb-markup](https://eflisrael.github.io/efldocs/textblock/data/images/examples/example-markup-font-bold.png)<br>
There are many more formatting options that Textblock offers.

### Formatting
Formatting can be achieved by either using the formatting functions, or by adding formatting tags when setting the markup text. Tags add convenience and are converted to the proper formatting property. Thus, instead of ```<font_weight=bold>Hello</font_weight> World``` we can just use ```<b>Hello</b> World```. The ```<b>``` and the ```<i>``` are default tags that are embedded in Textblock. Different tags can be supported by setting a style to the Textblock object using the ```evas_object_textblock_style_set``` method.

For all formattings that textblock offers, please consult the [Textblock Style Page](http://docs.enlightenment.org/auto/efl/evas_textblock_style_page.html).

## <a name=textblock_logic></a>Textblock Logic
When a text is set to Textblock, Textblock will save the text data. But, Textblock can't show directly the raw text. For handling complex texts, Textblock will makes logical and visual datas with the text and Textblock style.

Firstly, when the text is set to Textblock, the text will be processed to nodes. Textblock will makes text nodes for normal text, makes format nodes for markup tags. Normally, each paragraph has one text node. In other words, only the markup tags that make a new paragraph can split the text nodes. Format nodes are made for each markup tags, which are wrapped with "<", ">".

After setting a text to the Textblock, Textblock will relayout according to new format or new text. In this time, Textblock makes Items that would be used for layouting using Nodes.
Text Item will be added when some text has different visual properties. In other words, Text Items are visually seperated. Format Item will be added when there is visible format. Visible format means markup tags that has their visual position. For example, "\<br/\>".

These Items will be appended to a Line. And Lines makes a Paragraph.

### Nodes
When a user sets the textblock's markup using `evas_object_textblock_text_markup_set` (or `evas_object_textblock_text_markup_prepend`), the text is being parsed to create [Text Nodes](#nodes_text) and [Format Nodes](#nodes_format) from occurrences of plain text and tags, respectively. Nodes contain the minimal amount of information required, so that the heavy work of layouting can be performed on the rendering stages.

Format nodes are assigned to text nodes, and each text node can have many format nodes assigned to it.
The relation (pointers) between the two types is depicted in the following diagram:
![nodes](https://eflisrael.github.io/efldocs/textblock/data/diagrams/svg/tb-nodes-text-format.svg)

Solid lines represent required pointers to make it work, dotted lines represent list pointers (for iteration) and dashed lines represent pointers for optimization.

#### <a name="nodes_text"></a>Text Nodes [`Evas_Object_Textblock_Node_Text`]
Text nodes hold information of the actual text. The text is being stripped-down of format tags, and stored as unicode data.
The stripped format tags create format nodes with relevant information.

#### <a name="nodes_format"></a>Format Nodes [`Evas_Object_Textblock_Node_Format`]
These represent the format instances determined by the style and the format tags in a text.

### Paragraph
Paragraphs are delimited by a Paragraph Separator (PS) format tag. Each Paragraph
has lines, and each line describes the visual ordering of items.
The paragraph objects holds a logical list of all of its items.

![example1](https://eflisrael.github.io/efldocs/textblock/data/diagrams/svg/tb-items-paragraphs-example.svg)
(1) This is true if legacy newline support is turned off. Otherwise, a newline item creates a new paragraph.

### Lines
Lines are created by either having a Line Break format tag, or by a property of the
paragraph, such as line-wrapping.
Essentially, each line is a list of Textblock items.

### Items
Items are elements that describe how a line is laid out. Their order in each line of a paragraph determins the visual placement of the text in the rendering stage.
Each item is associated with a text node, as well as its offset in that node, so that multiple items may represent different sections of the same text node. Each item is associated with a format node.
There are two types of items: Text and Format. Each extend the Item object.

#### Text Items
Text items are essentially an extension to Evas Text Props.

###### Text Props
The text props (=properties?) is the core text structure of Evas. It is used throughout all text-related Evas objects.
Mainly, the text props consists of:
    - An array of glyphs
    - An array of glyph information.
    
Glyph information is being populated during the pre-layout stage, and the glyphs themselves at the rendering stage.
The information of each glyph, which is required to handle text properly, is as follows:
    - Pen position after glyph
    - x/y bearing
    - Width
    - Opentype Information:
        - Cluster index
        - x/y offset
        
![text_props_dia](https://eflisrael.github.io/efldocs/textblock/data/diagrams/svg/tb-text-item-struct.svg)

The Text_Props structure contains information for a whole paragraph. Also, it can be shared among multiple text items, which had been created as a result of one text item being split, for various resons. One example for such a case is text wrapping, and is demonstrated in the following diagram:
![text_props_wrapping_dia1](https://eflisrael.github.io/efldocs/textblock/data/diagrams/svg/tb-textprops-wrapping01.svg)
This is a rough example of how a single line of the "Hello World" text is linked to its text props. Note that the glyph information does not hold a character or a glyph, but only the information as described in the list above. The letters of each corresponding glyph informaion are placed in the diagram for the convenience of the reader.

If the text happends to get wrapped, due to width restrictions of the textblock's window, then the line will be split. In this case, the wrapping is set to be a "word-wrap".
![text_props_wrapping_dia2](https://eflisrael.github.io/efldocs/textblock/data/diagrams/svg/tb-textprops-wrapping02.svg)
As we can see, the single item has been split to three items. Each Text Item has its own Text Props structure, which corresponds to different ranges in the glyph information array. For example, the item of the word "World" is placed in the second line, but uses the same allocated space of the glyph information.

#### Format Items
Format items are special formats that exist in the text itself. Text elememnts such as the paragraph separator, line break, tabs and `<item>` tags create format items. Tabs and `<item>` format items can actually take up space in the text.

## <a name=layout_process></a>Layout Process
Textblock's layouting is divided to two main stages:
  - Creation of logical items as well as their paragraphs and lines (pre-layout)
  - Individual visual handling of each paragraph for cases such as lines geometry, wrapping etc.

### Pre-Layout
At this stage all of the first-level information (that has been set by user functions), such as text and format nodes, is processed. Each text node produces a single paragraph. Logical textblock items are created and assigned to their respectful paragraphs. This after being set with actual formats, that have been produced from processing the format nodes.

### Visual Layout
This is where all the logical items are processed, to make the lines in each paragraph. 
Geometries of all items, lines and paragraphs are calculated here. All line wrapping and ellipsis handling is done here as well. This stage results in what is called "The Formatted Layout".

## <a name=layout_process></a>Formatted vs. Native Size
The width and the height values of the resulant layout are known as the "formatted size". It has the width that the user has set for the textblock object (using the `evas_object_size_set` function), and the the required height so that the whole text fits in it.
On the other hand, we have the "native size" values. This can be thought of as the formatted size, but with its width already set to infinity.
The difference between the two types of sizing is that text wrapping can *not* occur with the latter, as the width is big enough to fit the whole line of the original text. So, the formatted height >= native height.
In actuality, the native size is calculated a bit differently: instead of running the layout process with an infinite width value, a different layouting mechanism is used. In this kind of layouting we skip wrapping handling. Lines are only created when meeting line or paragraph breaks. This makes the native size calculation a lot faster.


## <a name=unicode></a>Unicode
### References
- UAX #29 - Text Segmentation http://www.unicode.org/reports/tr29/
- UAX #9 - Bidirectional Algorithm http://www.unicode.org/reports/tr9/

## <a name=bidi_properties></a>Bidi Properties

## <a name=usages></a>Usages

# <a name=edje_entry></a>Edje Entry
Basically, [Edje Entry](http://docs.enlightenment.org/auto/edje/group__Edje__Text__Entry.html) is based in evas textblcok. It is supports many visual features for text input entry, which are not supported by evas textblock.
- Text edit cursor
- Cursor Handler
- Selection
- Selection Handler
- Key event processing
- Simple usage of item format, anchor format
- Ecore IMF(Input Method Framework) supporting - Virtual Keyboard

<br>
EFL developers can makes their own styles for entry using EDC, including Evas Textblock style, visual style, cursor, handler and so on.<br>
But, usually, Edje Entry is not used independently. It works as a part of Elementary Entry.

# <a name=elementary_entry></a>Elementary Entry (Widget)
