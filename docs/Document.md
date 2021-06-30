# JEditorPane's model - Document
By [Stanislav Lapitsky]()

Document interface implementation represents model of data to be shown in JTextComponent.
In fact Document is a chars sequence with tree like structure of Elements based on the chars.
Each Element has start and end offset – references to positions in the char sequence and a place
in hierarchy – reference to parent Element and list of children Elements.
(For more information about Document and Element read Javadocs Document and Element).
The start and end offsets mentioned above are provided by implementation of Position interface
and the char sequence basement is represented in most cases by GapContent class used in swings' 
Document implementations like AbstractDocument, DefaultStyledDocument, and PlainDocument.

The simplest implementation of Document interface is PlainDocument which is used in JTextArea to
represent plain text with one font used for all Elements. A more powerful interface StyledDocument
is used to represent text where fragments may have different font’s attributes like family, size,
color, and other effects. DefaultStyledDocument implements the StyledDocument interface and used
in most EditorKits directly or as super class for the kit’s Document.

Syled content in JEditorPane looks like this.
[image1]

DefaultStyledDocument structure is something like this.
[image2]

And the next image how it looks like in tree structure tool.
[image3]

## Document editing.

Document has 2 main methods which allow changing structure by changing the main char seqence.
```
    public void insertString(int offset, String str, AttributeSet a) throws BadLocationException;
    public void remove(int offs, int len) throws BadLocationException;
```
During user typing text Document’s structure is updated automatically. As it’s said above each element has start and end offsets.
When something is inserted before an offset the offset is automatically increased by the inserted content length.
Accordingly if content before offset is removed offset is decreased by the removed content length.

In DefaultStyledDocument a new paragraph is created for each “\n” char in the past str argument of insertString().
Also if AttributeSet has different set of attributes than text in the position a new leaf element is created.
For example if you have plain text “abc” and call insertString() in 1 position passing AttributeSet with bold
attribute inside one text Element will be converted in 3 plain/bold/plain accordingly.

DefaultStyledDocument has more flexible methods which allow changing structure.
```
    public void setCharacterAttributes(int offset, int length, AttributeSet s, boolean replace)
    public void setParagraphAttributes(int offset, int length, AttributeSet s, boolean replace)
```    
The two methods apply specified attributes to desired fragment. The setCharacterAttributes sets attributes on leaf elements’ 
level (TextElement s) and the setParagraphAttributes sets attributes one level on paragraph level.
There are two default attributes set to be applied to elements. To be applied to character Element.

* FontFamily
* FontSize
* Bold
* Italic
* Underline
* StrikeThrough
* Superscript
* Subscript
* Foreground
* Background
* ComponentAttribute
* IconAttribute

And for paragraph Elements.

* FirstLineIndent
* LeftIndent
* RightIndent
* LineSpacing
* SpaceAbove
* SpaceBelow
* Alignment
* TabSet

Also there is universal method which allows not only insert content with some attributes but change Elements’ structure creating multiple levels of hierarchy.
To support e.g. tables or another complicated structures.
```
	public void insert(int offset, int length, ElementSpec[] data, DefaultDocumentEvent de)
```  
And bunch of protected methods which could be useful if custom class extending DefaultStyledDocument is necessary.
```
    protected void create(ElementSpec[] data) 
    protected void insert(int offset, ElementSpec[] data) throws BadLocationException
```    
You can find an example how to implement custom structure in the article about [tables in JEditorPane](/docs/JEditorPane_tables.md).

In fact the ElementSpec[] array is sequence of elements’ tags. 
If the ElementSpec has type=ElementSpec.StartTagType it means all the next elements will be children of this one e.g. open tag in HTML.
If type= ElementSpec.EndTagType means current tag must be closed. The leaf ElementSpec.EndTagType must have type= ElementSpec.ContentType
and content of the leaf must be provided. E.g. new ElementSpec(new SimpleAttributeSet(), ElementSpec.ContentType, "\n".toCharArray(), 0, 1); 
will create a text Element representing end of paragraph.

The article gives information about data model and the next important step is the model representations by Views described in the next article.
