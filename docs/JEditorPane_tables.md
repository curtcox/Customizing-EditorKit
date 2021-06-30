# Tables in the JEditorPane/JTextPane.
By [Stanislav Lapitsky]()

Modern text editors require structured representation of text (tables).
The tables consist of rows and columns and framed by a colored border (black in the article)
and may contain paragraphs of text and images as well as nested tables. 
The cells (and rows) automatically increase their height according to height of content.

The article shows how to add tables in the JEditorPane/JTextPane.
The solution is based on StyledEditorKit extension but can be implemented in any other kit 
(e.g. RTFEditorKit from SUN doesn’t support tables in JEditorPane but may be extended to render them).
The new image illustrates the example frame.
[image]

The example consists of two parts – model (DefaultStyledDocument based) and Views (TableView, RowView, CellView) extend SUN’s BoxView class.

# Model implementation (DefaultStyledDocument extension).

StyledEditorKit uses DefaultStyledDocument class as a model of JEditorPane content.
The model is represented as a tree of Element class instances each of them contains own AttributeSet
with different parameters. For example text Element uses Font Family, Font Size, Effects like Bold, Italic etc.
Paragraph Element has own alignment, insets, TabStops. Basically the hierarchy is organized in 3 levels.

* Root
  *  Paragraph
    *   Text.

To add JEditorPane's tables we should change the structure to be like this:
* Root
  * Paragraph
  *  * Text
  * Table
  *  * Row(s)
  *  *  * Cell(s)
  *  *  *  * Paragraph(s)
  *  *  *  *  * Text

DefaultStyleDocument provides a method to create advanced structures.
protected void insert(int offset, ElementSpec[] data)
where ElementSpec[] describes structure changes. So we define insertTable() method where we will create the array of ElementSpec and change document’s structure. The method signature is
protected void insertTable(int offset, int rowCount, int[] colWidths)
We pass offset in the document where the table will be inserted, row count and width of the table’s columns. Size of the array defines column count in the table.

ElementSpec[] will be following. Closing tag (to close paragraph before table, table start tag, rows tags, table end tag and start a paragraph after table. Row tags in turns contains list of cells tags and each cell contains an empty paragraph with “\n” text. We also define names of the tags. The names will be used in ViewFactory to create proper views for each element.
So the code is
```java
    protected void insertTable(int offset, int rowCount, int[] colWidths) {
        try {
            SimpleAttributeSet attrs = new SimpleAttributeSet();

            ArrayList tableSpecs = new ArrayList();
            tableSpecs.add(new ElementSpec(attrs, ElementSpec.EndTagType)); //close paragraph tag

            SimpleAttributeSet tableAttrs = new SimpleAttributeSet();
            tableAttrs.addAttribute(ElementNameAttribute, ELEMENT_NAME_TABLE);
            ElementSpec tableStart = new ElementSpec(tableAttrs, ElementSpec.StartTagType);
            tableSpecs.add(tableStart); //start table tag

            fillRowSpecs(tableSpecs, rowCount, colWidths);

            ElementSpec tableEnd = new ElementSpec(tableAttrs, ElementSpec.EndTagType);
            tableSpecs.add(tableEnd); //end table tag

            tableSpecs.add(new ElementSpec(attrs, ElementSpec.StartTagType)); //open new paragraph after table

            ElementSpec[] spec = new ElementSpec[tableSpecs.size()];
            tableSpecs.toArray(spec);

            this.insert(offset, spec);
        }
        catch (BadLocationException ex) {
            ex.printStackTrace();
        }
    }

    protected void fillRowSpecs(ArrayList tableSpecs, int rowCount, int[] colWidths) {
        SimpleAttributeSet rowAttrs = new SimpleAttributeSet();
        rowAttrs.addAttribute(ElementNameAttribute, ELEMENT_NAME_ROW);
        for (int i = 0; i < rowCount; i++) {
            ElementSpec rowStart = new ElementSpec(rowAttrs, ElementSpec.StartTagType);
            tableSpecs.add(rowStart);

            fillCellSpecs(tableSpecs, colWidths);

            ElementSpec rowEnd = new ElementSpec(rowAttrs, ElementSpec.EndTagType);
            tableSpecs.add(rowEnd);
        }

    }

    protected void fillCellSpecs(ArrayList tableSpecs, int[] colWidths) {
        for (int i = 0; i < colWidths.length; i++) {
            SimpleAttributeSet cellAttrs = new SimpleAttributeSet();
            cellAttrs.addAttribute(ElementNameAttribute, ELEMENT_NAME_CELL);
            cellAttrs.addAttribute(PARAM_CELL_WIDTH, new Integer(colWidths[i]));
            ElementSpec cellStart = new ElementSpec(cellAttrs, ElementSpec.StartTagType);
            tableSpecs.add(cellStart);

            ElementSpec parStart = new ElementSpec(new SimpleAttributeSet(), ElementSpec.StartTagType);
            tableSpecs.add(parStart);
            ElementSpec parContent = new ElementSpec(new SimpleAttributeSet(), ElementSpec.ContentType, "\n".toCharArray(), 0, 1);
            tableSpecs.add(parContent);
            ElementSpec parEnd = new ElementSpec(new SimpleAttributeSet(), ElementSpec.EndTagType);
            tableSpecs.add(parEnd);
            ElementSpec cellEnd = new ElementSpec(cellAttrs, ElementSpec.EndTagType);
            tableSpecs.add(cellEnd);
        }

    }
```    
# Views implementation (BoxView extensions).

When model is defined we have to render the elements with proper views. So we create TableView, RowView and CellView.
CellView is the main one all the rest are just containers. CellView has inner insets to avoid border and text overlapping.
We also redefine getPreferredSpan(0 method of the cell to provide proper width.
The width is stored as an Attribute so if measured axis is X_AXIS we just extract the attribute value and return it in opposite case use super call.

```java
public class CellView extends BoxView {
    public CellView(Element elem) {
        super(elem, View.Y_AXIS);
        setInsets((short)2,(short)2,(short)2,(short)2);
    }
    public float getPreferredSpan(int axis) {
        if (axis==View.X_AXIS) {
            return getCellWidth();
        }
        return super.getPreferredSpan(axis);
    }
    public float getMinimumSpan(int axis) {
        return getPreferredSpan(axis);
    }
    public float getMaximumSpan(int axis) {
        return getPreferredSpan(axis);
    }
    public float getAlignment(int axis) {
        return 0;
    }

    public int getCellWidth() {
        int width=100;
        Integer i=(Integer)getAttributes().getAttribute(TableDocument.PARAM_CELL_WIDTH);
        if (i!=null) {
            width=i.intValue();
        }
        return width;
    }
```    
RowView is just cell container (cells are laid out horizontally).
TableView is rows container (rows are laid out vertically).
RowView paints vertical borders to split cells and TableView paints horizontal borders to split rows.
```java
public class TableView extends BoxView {
    public TableView(Element elem) {
        super(elem, View.Y_AXIS);
    }

    public float getMinimumSpan(int axis) {
        return getPreferredSpan(axis);
    }

    public float getMaximumSpan(int axis) {
        return getPreferredSpan(axis);
    }

    public float getAlignment(int axis) {
        return 0;
    }

    protected void paintChild(Graphics g, Rectangle alloc, int index) {
        super.paintChild(g, alloc, index);
        g.setColor(Color.black);
        g.drawLine(alloc.x, alloc.y, alloc.x + alloc.width, alloc.y);
        int lastY = alloc.y + alloc.height;
        if (index == getViewCount() - 1) {
            lastY--;
        }
        g.drawLine(alloc.x, lastY, alloc.x + alloc.width, lastY);
    }

public class RowView extends BoxView {
    public RowView(Element elem) {
        super(elem, View.X_AXIS);
    }

    public float getPreferredSpan(int axis) {
        return super.getPreferredSpan(axis);
    }

    protected void layout(int width, int height) {
        super.layout(width, height);
    }

    public float getMinimumSpan(int axis) {
        return getPreferredSpan(axis);
    }

    public float getMaximumSpan(int axis) {
        return getPreferredSpan(axis);
    }

    public float getAlignment(int axis) {
        return 0;
    }

    protected void paintChild(Graphics g, Rectangle alloc, int index) {
        super.paintChild(g, alloc, index);
        g.setColor(Color.black);
        int h = (int) getPreferredSpan(View.Y_AXIS) - 1;
        g.drawLine(alloc.x, alloc.y, alloc.x, alloc.y + h);
        g.drawLine(alloc.x + alloc.width, alloc.y, alloc.x + alloc.width, alloc.y + h);
    }
```    
All the 3 views minimum and maximum span is defined to return preferred span. Alignment is also set to 0.
Without the definition BoxView distributes extrac space from container and content will be centered.
It looks bad when one cell’s height is bigger than neighbor cell.
ViewFactory is changed to return appropriate views for all elements like this.
```java
class TableFactory implements ViewFactory {
    public View create(Element elem) {
        String kind = elem.getName();
        if (kind != null) {
            if (kind.equals(AbstractDocument.ContentElementName)) {
                return new LabelView(elem);
            }
            else if (kind.equals(AbstractDocument.ParagraphElementName)) {
                return new ParagraphView(elem);
            }
            else if (kind.equals(AbstractDocument.SectionElementName)) {
                return new BoxView(elem, View.Y_AXIS);
            }
            else if (kind.equals(StyleConstants.ComponentElementName)) {
                return new ComponentView(elem);
            }
            else if (kind.equals(TableDocument.ELEMENT_NAME_TABLE)) {
                return new TableView(elem);
            }
            else if (kind.equals(TableDocument.ELEMENT_NAME_ROW)) {
                return new RowView(elem);
            }
            else if (kind.equals(TableDocument.ELEMENT_NAME_CELL)) {
                return new CellView(elem);
            }
            else if (kind.equals(StyleConstants.IconElementName)) {
                return new IconView(elem);
            }
        }

        // default to text display
        return new LabelView(elem);
    }
}
```
You can download the JEditorPane's tables example .jar file here.

Appendix
Here is the full source code of the JEditorPane's tables example.
