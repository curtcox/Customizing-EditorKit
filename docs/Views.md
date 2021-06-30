# JEditorPane's views structure.
By [Stanislav Lapitsky]()

There are 3 main views used to represent Elements’ structure in JEditorPane – LabelView, BoxView, ParagraphView.

LabelView is a view to represent simple chunk of text with the same attributes.
It’s used on the leaf level of Document. Sometimes LabelView represents part of text Element when e.g.
the original view is too big and view is wrapped to be placed on multiple rows.
LabelView uses GlyphPainter to measure and render the model text using font will all parameters obtained from Element attributes.
Thus for the parent Element (or fragment of Element) the view provides width, height and paint the text with an appropriate font.
One or more label are joined into a row which extends the next mentioned class.

BoxView allows multiple child views to be laid out either vertically or horizontally.
The views will not wrap so, for example, a vertical arrangement of views will stay vertically arranged when container view is resized.
The BoxView is a super class for most views with children in JEditorPane e.g.
TableView in HTMLEditorKit or the BoxView is used directly e.g. to represent root Element of DefaultStyledDocument in StyledEditorKit.
The view has major and minor axis. If layout is horizontal the View.X_AXIS is major and View.Y_AXIS is minor. For vertical layout it’s opposite.

When size of the view is set it calls layout along minor and major axis methods.
```java
    protected void layoutMajorAxis(int targetSpan, int axis, int[] offsets, int[] spans)
    protected void layoutMinorAxis(int targetSpan, int axis, int[] offsets, int[] spans)
```      
The result of two layouts is four arrays – horizontal and vertical offsets and spans. To illustrate this let’s consider following example.

There is a BoxView with 3 children. Major axis is View.Y_AXIS.

Children have sizes:

w=100, height=20
w=80, height=20
w=100, height=40
Margins are top=10; left=15; bottom=30; right=20
After layout major offsets of the children
along Y axis are: {10, 30, 50} (started from 10 because of the top offset=10)
along X axis are: {15, 15, 15} (started from 15 because of the left offset=15)
And spans of the children
along Y axis are: {20, 20, 40}
along X axis are: {100, 80, 100}

And they’ll look like this:
[image]

ParagraphView also extends the BoxView class (rows are lay outed vertically) but direct super class is FlowView.
The latest one tries to flow its children into some partially constrained space.
Paragraph has restricted width (by container view) so it wraps content to create multiple lines each of them must fit available width.
At first paragraph has one big row consisting of all the LabelViews under paragraph.
If width of the row is bigger than available width of paragraph the row is broken in two rows.
The first row width is less or equals the available width of paragraph. For the second row paragraph again applies the same algorithm.
A few words about detecting proper break place. Normally paragraph detects the nearest place where the row can be broken
(nearest to the last suitable offset with position less than available width).
The place is detected by Break Weight algorithm. You can find more info the in the article about wrapping (see Line breaking algorithm section).
The paragraph’s layout is done by FlowStrategy class be replaced with custom one if alternative layout is necessary.

See views structure of a simple example based on StyledEditorKit:
[image]

Now let’s look inside the work of the views. When model is changed and a new Element is added parent View (listens Document changes)
gets a notification event and recreates array of children. For the newly created elements ViewFactory.create() method is called to 
create proper view for the Element. ViewFactory knows which view must be created for the specific Element depending on its type, attributes etc.

The source code of StyledViewFactory of StyledEditorKit is following.
```java
    static class StyledViewFactory implements ViewFactory {
 
        public View create(Element elem) {
            String kind = elem.getName();
            if (kind != null) {
                if (kind.equals(AbstractDocument.ContentElementName)) {
                    return new LabelView(elem);
                } else if (kind.equals(AbstractDocument.ParagraphElementName)) {
                    return new ParagraphView(elem);
                } else if (kind.equals(AbstractDocument.SectionElementName)) {
                    return new BoxView(elem, View.Y_AXIS);
                } else if (kind.equals(StyleConstants.ComponentElementName)) {
                    return new ComponentView(elem);
                } else if (kind.equals(StyleConstants.IconElementName)) {
                    return new IconView(elem);
                }
            }
 
            // default to text display
            return new LabelView(elem);
        }
```      
Then the created view is added to parent. Parent’s layout becomes invalid and views are relayed out.

The article gives information about model data reflection by views structure. Next article shows how model's data - Documnet
structures can be stored adn loaded via custom reader and writer.

