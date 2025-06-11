# Docling Document Exploration

#### Conducted by: Ryan Fernandes, June 2025

<br>

## Methodology

The specifications for the Docling Document data type are outlined in the [docling/docling-core](https://github.com/docling-project/docling-core) repository. Within this repository, I reviewed the [pydantic](https://www.geeksforgeeks.org/introduction-to-python-pydantic-library/) definition of Docling Documents within the [types/doc](https://github.com/docling-project/docling-core/tree/main/docling_core/types/doc) subdirectory. 

In my exploration, I first read through each file in this subdirectory, taking notes of functions that may be useful for the purposes of document review/revision and paying attention to how certain functions are implemented under the hood. These notes are summarized below. The full, unfiltered notes are available in ```notes.md```.

Next, I used the BofA ```01_Advantage_Savings``` document converted from PDF to highlight the structure of Docling Documents in pydantic and json format, as well as the provided methods to add/update/delete items in Docling Documents. These examples are available in the ```docling_doc_structure.ipynb``` notebook.

Lastly, I converted two documents from the BofA PoC that the team has identified issues with in the past. In the ```docling_error_revision.ipynb``` notebook, I provide prototypes for fixing these issues using built-in Docling capabilities, as well as ideas for how this could be integrated into a UI.

<br>

## Files Overview: [docling_core/types/doc](https://github.com/docling-project/docling-core/tree/main/docling_core/types/doc)

| File Name | Lines | Main Ideas |
|-|-|-|
| \_\_init__.py | 32 | Packaging of models in other files |
| base.py | 435 | Defines classes/methods for denoting page locations and spans, most importantly ```BoundingBox```|
| **document.py** | 4320 | The main file of interest, especially for the purposes of editing Docling Documents. Provides pydantic class definitions for a variety of elements that make up Docling Documents (eg. ```TextItem```, ```TableItem```). Lines 1615-4320 contain the pydantic definition of ```DoclingDocument```, including all public methods available on Docling Documents |
| labels.py | 262 | Helps to keep other files organized with enum classes for different types of labels. For example: ```GroupLabel.ORDERED_LIST``` |
| page.py | 1274 | Defines models for pages of documents, with specific and extended models for pages from PDF documents. |
| tokens.py | 295 | Similar to ```labels.py```, contains enum classes for DocTags. For example, ```TableToken.OTSL_ECEL = <ecel>``` |
| utils.py | 75 | Provides 3 util functions: 2 are related to HTML tags and language direction (LTR or RTL), 1 calculates relative file paths between two files |

<br>

## Main Takeaways


### 1. Interacting With and Saving Docling Documents

The Docling Document is readily accessible after conversion by accessing the ```document``` property of ```converter.convert(source)```, as shown below:

```python
from docling.document_converter import DocumentConverter

source = 'docling-document/files/01_Advantage_Savings.pdf'
converter = DocumentConverter()
result = converter.convert(source)
docling_doc = result.document

print(type(docling_doc))
```

```
<class 'docling_core.types.doc.document.DoclingDocument'>
```

Upon accessing the Docling Document instance, many built-in methods for manipulation become available. If Docling Documents must be transferred between files, this can be done in a lossless manner by using the built-in ```save_as_json(filename: str)``` and ```load_from_json()``` methods:

```python
from docling_core.types.doc.document import DoclingDocument

FILE_NAME = 'docling-document/files/example.json'

# Check original length
orig_len = len(docling_doc.export_to_markdown())
print(f'Document length before: {orig_len}')

# Save document as JSON
docling_doc.save_as_json(FILE_NAME)

# Re-upload as a new Docling Document
new_docling_doc = DoclingDocument.load_from_json(FILE_NAME)
final_len = len(new_docling_doc.export_to_markdown())
print(f'Document length after: {final_len}')
```

```
Document length before: 5229
Document length after: 5229
```

Notably, this lossless storage of information **does not exist once documents are converted to Markdown**. Metadata is lost in the conversion to Markdown, making it far more difficult to fix errors in conversion than when the data is in DoclingDocument format.

Lastly, when the DoclingDocument is exported to JSON, it is perfectly possible to add/edit the raw content of the JSON file and re-upload as a DoclingDocument. The demonstration below shows that when adding an extra text item to the bottom of the DoclingDocument JSON, there are no errors in re-upload, and the changes are reflected in the resulting data structure. To make the edits, I simply opened a JSON editor, but it is equally possible to make these changes in a programmatic manner:

```python
from docling_core.types.doc.document import DoclingDocument

FILE_NAME = 'docling-document/files/example.json'

# Check original length
orig_len = len(docling_doc.export_to_markdown())
print(f'Document length before: {orig_len}')

# Upload edited version of the JSON as a Docling Document
new_docling_doc = DoclingDocument.load_from_json(FILE_NAME)
final_len = len(new_docling_doc.export_to_markdown())
print(f'Document length after: {final_len}')
```

```
Document length before: 5229
Document length after: 5301
```

### 2. Public Methods to Modify Docling Documents

A more structured and safer alternative to editing Docling Documents by editing the raw JSON, there exist plenty of methods to add, insert, and delete items.

**Adding items** can be done in a variety of ways, as there are different methods of specifying where the item should be added. The 3 most applicable methods are ```append_child_item()```, ```insert_item_after_sibling()```, and ```insert_item_before_sibling()```. All of these methods accept two parameters, both of which are instances of ```NodeItem```, Docling's general class for specifying the content that exists in a document. The first parameter is ```new_item```, the item to be added to the document. Note that this item must already be constructed as an instance of ```NodeItem``` before being passed to this function. The second parameter is either the parent (first method, optional) or the sibling (second two methods, required), which specifies where the new item should be placed in the document. If no parent is specified in ```append_child_item()```, the default parent is ```body```, placing the item at the top content layer of the document.

A couple examples of adding items to desired locations are outlined below:

**1\. Adding a child item based on its parent**

In the ```01_Advantage_Savings``` document, there is a list of items explaining how to avoid the Monthly Maintenance Fee. The code below adds an item to this list group.

```python
# Get the parent node item

from docling_core.types.doc.document import RefItem

targetRef = RefItem(cref="#/groups/0")
parent = targetRef.resolve(doc=docling_doc)

print(parent)
```

```
self_ref='#/groups/0' parent=RefItem(cref='#/body') children=[RefItem(cref='#/texts/11'), RefItem(cref='#/texts/12'), RefItem(cref='#/texts/13'), RefItem(cref='#/texts/14')] content_layer=<ContentLayer.BODY: 'body'> name='list' label=<GroupLabel.LIST: 'list'>
```

```python
# Create the new child item

from docling_core.types.doc.document import ListItem

new_ref = "#/extras/1" # Must match a regex validation of  "#/{word}/{num}
text = "This is my new list item"
orig = "This is my new list item"

new_item = ListItem(
    self_ref=new_ref,
    text=text,
    orig=orig
)
```

```python
# Add the item and observe the change to the document

docling_doc.append_child_item(
    child=new_item, 
    parent=parent
)

print(parent.children)
```

```
[RefItem(cref='#/texts/11'), RefItem(cref='#/texts/12'), RefItem(cref='#/texts/13'), RefItem(cref='#/texts/14'), RefItem(cref='#/texts/39')]
```

**2\. Adding a child item based on its siblings**

Instead of appending our new list item to the end of the existing list, say that we wanted it to be this first element. This type of insertion is done with ```insert_item_after_sibling()``` and ```insert_item_before_sibling()```. We continue the example from above:

```python
# Get the sibling NodeItem for the first item in the list

firstRef = parent.children[0]
firstItem = firstRef.resolve(doc=docling_doc)

print(firstItem)
```

```
self_ref='#/texts/11' parent=RefItem(cref='#/groups/0') children=[] content_layer=<ContentLayer.BODY: 'body'> label=<DocItemLabel.LIST_ITEM: 'list_item'> prov=[ProvenanceItem(page_no=1, bbox=BoundingBox(l=154.8, t=635.624, r=465.39, b=625.614, coord_origin=<CoordOrigin.BOTTOMLEFT: 'BOTTOMLEFT'>), charspan=(0, 70))] orig='· Maintain a minimum daily balance of $500 or more in your account, OR' text='· Maintain a minimum daily balance of $500 or more in your account, OR' formatting=None hyperlink=None enumerated=False marker='-'
```

```python
# Insert our new item before the sibling

docling_doc.insert_item_before_sibling(
    new_item=new_item,
    sibling=firstItem
)

print(parent.children)
```

```
[RefItem(cref='#/texts/40'), RefItem(cref='#/texts/41'), RefItem(cref='#/texts/11'), RefItem(cref='#/texts/12'), RefItem(cref='#/texts/13'), RefItem(cref='#/texts/14'), RefItem(cref='#/texts/39')]
```

Before moving on, there are a couple points to note from these examples:

1. **Docling modifies the new_item under the hood.** Notice how the cref's of the two new items in ```parent.children``` are "#/texts/39" and "#/texts/40," even though we had originally set the item's ref to "#/extras/1." Any method that contributes a NodeItem to the DoclingDocument will first modify that NodeItem to have an appropriate self_ref in the format of "#/{type}/{index}". This means that we do not need to worry about appropriately setting self_ref when creating items, nor do we have to worry about self_ref duplicates. However, Docling still stores the NodeItems in the resources section of the datatype by reference, so one can change self_ref after adding the NodeItem to the document, and the change will be reflected. Be advised that this will not affect accessing items through the body, as these are all stored as new RefItems (rather than a NodeItem reference) that point to an index of an attribute rather than a key.

2. **RefItems are very powerful for accessing NodeItems.** Ref items can be instantiated with one cref parameter, or accessed from ```children``` properties within the DoclingDocument. Their ```.resolve(document)``` method makes it easy to get existing NodeItems that must be passed as parameters to other functions. However, this ```.resolve()``` functionality only finds NodeItems based on the "location" specified in their cref, not by matching the cref with self_ref. Thus, it is important to beware that an object's self_ref cannot always be used as a cref to locate it.

**Deleting and replacing items** are implemented in a similar way to the demonstrated addition methods. In fact, the item replacement method simply adds an item and then deletes the old item using the public methods. Thus, I will leave out their implementations here and reference those interested to the ```docling_doc_structure.ipynb``` notebook. The methods are ```delete_items()```, which takes one paremter, node_items, as a list of NodeItems, and ```replace_item```, which takes the parameters new_item and old_item as NodeItems. Note that since the delete_items() method takes NodeItems as inputs rather than RefItems, it ends up creating RefItems from the self_ref attribute on each NodeItem. Coupled with note #2 from above, this makes it **extremely dangerous** to change the self_ref attribute of NodeItems once they have been added to documents. In turn, it is also very dangerous to add a single NodeElement more than once (either to the same document or different documents), as this will, by default, update the self_ref attribute. Demonstrations of this are in the ```docling_doc_structure.ipynb``` notebook.

**To add items without instantiating a NodeItem**, Docling provides a collection of "add" methods. These methods do not take a NodeItem as a parameter, but rather the values necessary to create the type of NodeItem requested. For instance, the ```add_title``` method only requires that a string for the text of the title be passed (with other optional parameters as well). The methods then instantiate the type of NodeItem required, add it to the DoclingDocument as a RefItem and NodeItem, and return the created NodeItem as the output of the function. In practice, this has about the same functionality as ```append_child_item()```, but with more of the complexity abstracted away. It also provides safeguards against self_ref distortion, as it guarantees that each item added to the document is a completely new instance of a NodeItem. However, there are only "add" functions, not "insert" or "update" functions to mimic ```insert_item_after_sibling()```, ```insert_item_before_sibling()```, and ```replace_item()```. **These functions implementations could be a contribution that our team makes to docling-core.**

Overall, there is a robust collection of public methods available to use to edit Docling Documents. However, the biggest obstacles will be in correctly creating the NodeItems need to be added to the document and accessing the existing NodeItems that serve as "locators" within the document. To determine how to best do this, **we must first define the user experience that we want for revising document conversion**. Then, we can determine how to implement existing methods or contribute new ones.

### 3. Displaying and Reviewing Docling Documents

Having reviewed the class definitions of Docling Documents raises some questions about how we should encourage conversions to be reviewed. The Docling-Serve UI utilizes the Docling-Rendered format that is availble in the [docling-ts](https://github.com/docling-project/docling-ts) library for Typescript projects. However, this format can be a bit misleading, as it simply overlays a png of the original document with the Bounding Boxes of items that Docling classified and tooltip annotations of the content extracted from each Bounding Box. While this display succeeds in showing how Docling grouped the document, it is harder (in my opinion) to quickly spot formatting/content errors. 

Additionally, the view misrepresents what the DoclingDocument data structure actually looks like and how it is ingested by future stages. Further investigation would need to be done to confirm this, but from speculation, I don't believe that the Hybrid Chunker utilizes the page png's, nor does our authoring/seed data/SDG stages. Furthermore, I'm uncertain how the Bounding Box data influences the other stages, especially if we are not using DocTags as the plain text representation of the document. Thus, it may be more useful for the user to get a view similar to the Markdown Preview, in which they can see a presentation that more closely aligns with what will be used. Extracted content is also directly visible, rather than being extracted away, which makes it easier to spot errors. However, converting to Markdown has its own loss issues, as mentioned earlier. Thus, what would be useful is a **raw DoclingDocument preview** that sits somewhere between seeing the DocTags and seeing the png with overlays. This is another option for contribution that could be considered.

Lastly, I keep floating an idea about omission/hallucination detection. After looking into the omission issue that I ran into and reviewing the way Docling Documents are structured, it seems that the omission I ran into was an issue with the transform from DoclingDocuments to markdown format, rather than the Docling Document itself. Thus, it could be useful to get the raw text of a Docling Document and compare it to the markdown file as a check for accurate conversion, especially if the markdown file is to be used somewhere else. Due to the DocTags aligning very closely with the structure of Docling Documents, the ```export_to_doctags()``` method can be used as a basis to get the raw text content of a Docling Document: 

```python
import re

print(re.sub(r"<.*?>", "", docling_doc.export_to_doctags(), flags=re.DOTALL).strip())
```

From this, **conversion/export checks** can be developed. Note that Docling documents also have an ```export_to_text()``` method, but this name is actually quite misleading. In reality, the ```export_to_text()``` method simply wraps the ```export_to_markdown()``` function, which is not the behavior we are after. There are many other functions for export, including ```export_to_element_tree()```, ```export_to_html()``` and ```save_as_yaml()```. Of these, only ```save_as_json()``` and ```save_as_yaml()``` are designed to be completely lossless, meaning that the resulting file can be reuploaded using ```load_from_json()``` and ```load_from_yaml()``` to get the original Docling Document without any changes. For DocTags, there is the pair of ```export_to_doctags()``` and ```load_from_doctags```, but there is some loss when converting to this representation, as not all metadata appears to be stored/inferrable.

## Ideas for Future Tasks/Investigations:

From the results of this investigation, I believe we could benefit from the following:

- Investigate the Markdown transformer to identify where omissions are introduced and submit a pr with a fix
- Investigate the Hybrid Chunker to see what data from Docling Documents are actual used
- Investigate the format of chunks passed to SDG and what data is included (plain text, markdown, DocTags)
- Create additional methods for DoclingDocuments that allow more robust editing on top of the layer of safeguards that the data type imposes
- Investigate the document conversion pipeline and if we can start at certain stages (eg. force certain bounding boxes and convert with those boxes defined)
- Determine options for presenting Docling Documents for review and which best aligns with user needs (eg. raw DoclingDocument preview)
- **Discuss the pros/cons of creating an opinionated, Docling pipeline that leverages the ability to store Docling Documents in a lossless manner as JSON/YAML, as opposed to generalizing to markdown for "plug and play" abilities with other converters/chunkers**