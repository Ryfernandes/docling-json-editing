## Docling Document Editing Notes

### Exploration 1: Docling-Core Repo

[(Reference)](https://docling-project.github.io/docling/concepts/docling_document/)

**Docling Document contains two types of items:**

DocItem (express data structures based on type and reference to parents/children):
- Texts
- Tables
- Pictures
- Key-value items

NodeItems (only contain references to parents/children):
- Body
- Furniture (headers, footers, etc.)
- Groups (divs)

[(Reference)](https://www.geeksforgeeks.org/introduction-to-python-pydantic-library/)

**Pydantic Data Structures:**

- Helps to ensure that Python data is in desired/regulated format
- Helps to define and validate data models easier
- Features:
    - Type validation
    - Data parsing
    - Error handling
    - Field validators (custom validation logic aside from types)
    - Permorance/Integration

*Install:*
```
pip install pydantic
```

- Pydantic models are Python classes that inherit from Base Model
- Each attribute is a field in the data, and the type annonations define the expected type
- Pydantic will raise errors if incorrect data types are provided

*Simple example:*
```python
from pydantic import BaseModel

class UserProfile(BaseModel):
    name: str
    age: int
    email: str

user = UserProfile(name="Ryan", age=19, email="ryan@email.com")
```

*Set defaults:*
```python
from pydantic import BaseModel

class UserProfile(BaseModel):
    name: str = 21
    age: int
    email: str

user = UserProfile(name="Ryan", email="ryan@email.com")
```

*Field validators:*
```python
from pydantic import BaseModel

class UserProfile(BaseModel):
    name: str = 21
    age: int
    email: str

    @field_validator('age'):
    def check_age(cls, value):
        if value < 21:
            raise ValueError('Age must be at least 21')
        return value

user = UserProfile(name="Ryan", age=19, email="ryan@email.com")
```

*Nested Models:*
```python
from pydantic import BaseModel

class Address(BaseModel):
    street: str
    city: str

class UserProfile(BaseModel):
    name: str = 21
    age: int
    address: Address

    @field_validator('age'):
    def check_age(cls, value):
        if value < 21:
            raise ValueError('Age must be at least 21')
        return value

address = Address(street="Whitney Circle", city="Edgartown")
user = userProfile(name="Ryan", age=19, email="ryan@email.com")
```

*Parse Data from JSON*
```python
from pydantic import BaseModel

class UserProfile(BaseModel):
    name: str = 21
    age: int
    email: str

    @field_validator('age'):
    def check_age(cls, value):
        if value < 21:
            raise ValueError('Age must be at least 21')
        return value

data = '{"name": "Ryan", "age": 19, "email": "ryan@email.com"}'

user = UserProfile.model_validate_json(data)
```

*Serialize Data to JSON*
```python
from pydantic import BaseModel

class UserProfile(BaseModel):
    name: str = 21
    age: int
    email: str

user = UserProfile(name="Ryan", email="ryan@email.com")

json_data = user.json()
```

*Optional Fields*
```python
from typing import Optional
from pydantic import BaseModel

class UserProfile(BaseModel):
    name: str
    age: Optional[int] = None
    email: str
```

[(Reference)](https://github.com/docling-project/docling-core/blob/main/docling_core/types/doc/base.py)

**Docling-core: doc/base.py**

- Contains information about the BoundingBox Model, which has l, t, r, b (left, top, right, bottom) and coord_origin ("TOPLEFT" | "BOTTOMLEFT")
- BoundingBox has properties for area, width, methods for computing/checking overlaps, class method for generating BoundingBox for union of boxes
- Many multi-BoundingBox methods require that the BoudningBoxes have the same CoordOrigin

[(Reference)](https://github.com/docling-project/docling-core/blob/main/docling_core/types/doc/labels.py)

**Docling-core: doc/labels.py**

- Contains enums to with different types of labels for content, groups, images, code, etc
- Some types of label have colors associated with the different potential values

[(Reference)](https://github.com/docling-project/docling-core/blob/main/docling_core/types/doc/tokens.py)

**Docling-core: doc/tokens.py**

- Provides pydantic models and classes for turning the Docment labels into LLM-friendly tokens/DocTags
- Specifies custom table representation with tags
- Specifies ability to specifiy locations on the page with four DogTags "<loc_#>"
- Section headers are specified as DocTags by level "<section_header_level_#>"

[(Reference)](https://github.com/docling-project/docling-core/blob/main/docling_core/types/doc/utils.py)

**Docling-core: doc/utils.py**

- Unhandled edge cases in relative path code with calculating nearest ancestor?
- This file has 3 util functions to be used in the Docling project
- One calculates relative file paths and is potentially used for image references?
- The other two have to do with generating html tags appropriately depending on the direction of the language being passed into the tags

[(Reference)](https://github.com/docling-project/docling-core/blob/main/docling_core/types/doc/page.py)

**Docling-core: doc/page.py**

- Overall, defines the structure of pages, with specific models for PDF documents (metadata, different page boundaries, table of contents, etc)
- Positions are relative to the page they are on, not the entire document (converting from TOPLEFT to BOTTOMLEFT uses page height)
- Text cells have tags that indicate whether they were sourced from ocr (could maybe sort by this to see the effect of OCR and what documents it is relevant for?)
- Bitmap: any resource that is just a collection of rows of pixel values
- What is the "angle" of a rectangle/page and how is this useful in calculations shown?
- Segmented page has charcacter, word and textline cells, as well as BitMap resources and an optional parameter for an image of the page
- For PDF pages/documents, the geometry/bounding boxes are more complex
- There is the ability to get all text cells/resources within a bounding box
- Pdf page model is able to be loaded from json and dumped to json
- Able to extract text from cells within a specified bounding box for a PDF page
- Very big function to draw a PDF page as a "PIL image"
    - Python image library
- PDF metadata is extracted from the XML and put in a model
- Pypdfium objects seem to have the method model_validate_json() that makes it possible to construct a model from JSON
- When loading from JSON, it seems to be done from a separate file with a filename reference
- Seems like most high-level document types have save_as_json() and load_from_json()

[(Reference)](https://github.com/docling-project/docling-core/blob/main/docling_core/types/doc/document.py)

**Docling-core: doc/document.py**

- PictureClassificationClass has a classification class string and a float for confidence
- The PictureClassificationData has a list of all predicted classes and confidences (PictureClassificationClass)
- There are classes that sepecify specific types of data that one would get from a chart, with exact values, that are then combined with axis labels and a title to create a chart/graph
- TableData supports merged cells by initializing the table as a collection of blank cells from row and column count and then adding the full cell object in any cell that matches the correct row/col indexes
- Document origin is useful for tracking changes with mimetype, binary_hash, filename, and uri (if provided)
- For any ImageRef, the PIL image can be returned (generated if needed)
- ImageRef can be created as well from PIL images with the uri set as a Base64 encoded data: uri
- A DocTags page is just a string of DocTags with an optional image and a model_config
- A DocTags document is just a list of DocTags pages and can be constructed as well as two lists of DocTags and images
- Provenance items have a page number, bounding box, and character span
- RefItems will be very important for determining and editing parents/children

Node Items:

- Contain a self_ref in the form of a string, rather than RefItems
- A parent and children are specified with RefItems or RefItem lists, so is there a way to use self_ref to more safely construct these
- Yes, the method get_ref returns a ref to the object as a RefItem
- Confused by _get_parent_ref()... what is the stack? (list of integers), it seems to be propagating down the tree instead of up it as I would expect
- When deleting child, it seems like the child is just decoupled from its parent (so it won't display in document?). However, the content will remain in the Docling Document object
- It seems that any functions with a stack are meant to be invoked from the top of the tree and be capable of working their way down to the target level
- Not sure what the .resolve() function does on a RefItem
- Update child seems to allow a ref to a new object, could be useful in editing
- Add sibling seems to be like add child, but with an insert instead of an appedn operation
- There are three main types of of Group Items:
    - Unordered lists
    - Ordered lists
    - Inline groups
- DocItems extend NodeItems and have a list of ProvenanceItem's
- Text items have orig and text properties, where orig is the original, untreated representation, and text is "sanitized"
- The DocTagsDocSerializer appears to be responsible for turning the model into Doc Tags (at least for text)
- For most Doc Items, there appear to be functions in place to get the raw text
- Formula and Code items are their own types of items that extend FloatingItem
- PictureItems have methods to export to a variety of different formats (markdown, html, etc)
- ** Tables can be exported to pandas dataframes, which could be very useful for review
- Tables are exported to markdown in the MarkdownDocSerializer, which appears to be where the conversion from DoclingDoc's to markdown is specified, possibly where our errors are coming in
- A lot fo the code before 1600 seems to be focused on the export of Docling Document elements to other formats, so that it is conveniently modularized for the Docling Document implementation
- PageItems have image parameters, which I assume is a screenshot of the page on which the overlays are placed for document review
    - Thus, while this enables convenient features for document conversion review, displaying updates live while editing becomes more difficult
- Docling Documents have an origin parameter that can specify where they came from if converted. However, this is optional for if the documents were synthesized rather than converted
- Docling Documents have the following parts/objects
    - Body
    - Groups
    - Texts
    - Pictures
    - Tables
    - Key Value Items
    - Form Items
- Will need to see groups in a Docling Document body to understand their function

**Public Manipulation Methods (Line 1650 - 1720):**

- Stack does not need to be specified by the user, but rather is calculated after a parent is specified
- append_child_item()
  - Takes parameters of a child NodeItem and an optional parent NodeItem (would need to be determined first)
- insert_item_after_sibling()
  - Another way of determining where to add the child item
- insert_item_before_sibling()
  - Same as above, but moves item an element before, rather than after
- delete_items()
  - This function deletes the supplied node_items from the DoclingDocument, as well as any children that they have
- replace_item()
  - Simply adds an item and then deletes the old one
- Overall, it seems that the main dificulty in creating editing functionality will be in creating instances of NodeItems and abstracting this away. However, this may be done in Docling-MCP

**Other Public Methods**

- Generally, it seems that add() just adds support for appending

**Can ordered lists be added other ways, like with insert_item?**

- add_ordered_list()
  - Will add an empty group for an ordered list wherever the parent is specified at
- add_unordered_list()
- add_inline_group()
- add_group()
  - If a label is specified, will just call the other methods
- add_list_item()
  - Must have a parent specified that is an OrderedList or UnorderedList
  - Able to be added without instantiating a ListItem, will be done for you
- add_text()
  - "cref" is the "#/{type}/{index}" that Docling uses
  - To physically add an item to a document in a certain place (reference, not content), all that is needed is 'parent.children.append(RefItem(cref=cref))'
- add_table()
  - To add a table, its data must be added as TableData
- add_picture()
  - Needs an input of an ImageRef
  - Outputs a PictureItem **remind what the difference is**
- add_title()
  - Only requires text
  - **Clarify to team based on data structures what must be provided and what each option does**
- add_code()
  - Text based
- add_formula()
  - Text based
- add_heading()
- add_key_values()
  - **Need to understand better what key_values actually are**
  - Takes GraphData as an input
- add_form()
  - Also GraphData input

**Outside of adding (appending)**

- num_pages()
  - Returns the number of pages in the document
- validate_tree()
  - Validates that all of the children are linked correctly to their parents
- iterate_items()
  - Returns a tuple of all of the NodeItems with their level
  - Traverses all children with _iterate_items_with_stack
  - **Very good base levl to give us the necessary information to be able to make edits/select and display values**
- print_element_tree()
  - Displays simple view of the element tree
- export_to_element_tree
  - Same formatting as print_element_tree, but just exported as a string instead of being printed
- save_as_json()
  - **VERY IMPORTANT METHOD (+below)**
  - Requires file name
  - Simple json dump with images dealt with as specified
- load_from_json()
  - Will validate structure, just requires file name and returns a Docling Document
- save_as_yaml()
- load_from_yaml()
- export_to_dict()
- save_as_markdown()
  - Wraps export_to_markdown()
- export_to_markdown()
  - Outputs a string
  - Lossy, can not convert directly back to a Docling Document
  - Many parameters that should be investigated in a separate spike
  - Uses MarkdownDocSerializer, MarkdownParams from docling_core.transforms.serializer.markdown
- export_to_text()
  - Wraps export_to_markdown()
- save_as_html()
  - Wraps export_to_html()
- export_to_html()
  - Outputs a string
- load_from_doctags()
  - Takes in a doctag_document, which is then output as a text file for the model to finetune/RAG/etc on
- extract_bounding_box()
  - Takes in a string and returns a bounding box object constructed using the first four <loc_...> tags found in the string
- extract_inner_text()
  - Takes in a string of a text chunk and strips all DocTags. Thus, this just gives the raw text content of the document **(could be very useful for omission/hallucination detection)**
- extract_caption()
  - Returns a caption item from doc.add_text() (on an empty document), as well as the BoundingBox object of the caption specified in the text chunk string
- otsl_parse_texts()
  - Utility functions related to parsing texts and tokens with the OTSL language
- otsl_extract_tokens_and_text()
  - Takes in a string and outputs a collection of all of the tokens present in the string and all of the texts
- parse_table_content()
  - Creates a full TableData object from an OTSL string by using the two methods above
- extract_chart_type()
  - Returns the last clabel (chart label) tag value that appears in an input string
- parse_key_value_item()
  - Skipped
- save_as_doctags()
  - Wraps export_to_doctags() and saves the result as a file
- export_to_doctags()
  - Uses DocTagsDocSerializer and DocTagsParams, outputs doc tags as a string
- add_page()
  - Creates and returns a new, blank PageItem with an optional image
- get_visualization()
  - Can enable a show_label flag
  - Returns an images list using the LayoutVisualizer and ReadingOrderVisualizer classes
- Some validators at the very end of the class definition
 
**A lot of these functions are just convenient wrappers of class instantiations and then linking the ref to the body/content to the corpus**

Hidden methods:

- _append_item() seems to take care of properly naming items and placing them in the "library"
  - However, it does not appear to place it in the body
- "path" refers to the name used to refer to the item (its ref)
- _pop_item() only works for the last item of a type
- _delete_items() has a lot of cleanup logic
- Other utility hidden methods throughout the project