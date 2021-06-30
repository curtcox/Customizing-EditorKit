# EditorKit - Document - Views relations tutorial (Overview).

By [Stanislav Lapitsky]()

I am often asked to explain some details or overall concept of EditorKits using.
People ask how Views work or how Document interacts with Views or how different actions changes model and views reflect the changes and so on.
Very often they don't have some base knowledge and I have to explain common things and concepts. The next articles were written to simplify the explanations.
Hope the articles are useful. If not just skip them and try to have better understanding from the source code - the best approach I can suggest.

Table of content of the tutorial

* Overview
* [Document](/docs/Document.md)
* [ViewFactory and Views](/docs/Views.md)
* [Reader and Writer]()
* [Actions]()
* [Example]()

EditorKit encapsulates Document (Model) with tools to write and read specified type of content, Views, and some actions to work with content.
So it provides Document via createDefaultDocument() method, allows to write and read the Document data, in other words save and restore the Document,
provides ViewFactory to create representations of the Document via getViewFactory() method, and list of Actions used to work with this type of content via
getActions(). To illustrate this let's consider HTMLViewFactory. It provides HTMLDocument with all necessary settings as model and HTMLFactory 
for the Document structure showing. Calling of the write() method will produce HTML representation of desired fragment of the HTMLDocument
(according to specified offset and length). Calling of read method will insert HTML in the Document.
HTMLEditorKit also add some specific action like table/row/column insertion, ordered and unordered lists inserting etc.

When JEditorPane.setEditorKit() is called it means the JEditorPane will work with kit specific content, read/write/edit/show all the content defined by the kit.

The next article explains data model of EditorKit - [Document]()

[Back to Table of Contents](/README.md)
