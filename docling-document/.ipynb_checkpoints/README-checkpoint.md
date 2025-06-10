## Docling Document Exploration
Conducted by: Ryan Fernandes, June 2025

<br>

### Methodology

The specifications for the Docling Document data type are outlined in the [docling/docling-core](https://github.com/docling-project/docling-core) repository. Within this repository, I reviewed the [pydantic](https://www.geeksforgeeks.org/introduction-to-python-pydantic-library/) definition of Docling Documents within the [types/doc](https://github.com/docling-project/docling-core/tree/main/docling_core/types/doc) subdirectory. 

In my exploration, I first read through each file in this subdirectory, taking notes of functions that may be useful for the purposes of document review/revision and paying attention to how certain functions are implemented under the hood. These notes are summarized below. The full, unfiltered notes are available in ```notes.md```.

Next, I used a simple example document converted from markdown to highlight the structure of Docling Documents in pydantic and json format, as well as the provided methods to add/update/delete items in Docling Documents. These examples are available in the ```docling_doc_structure.ipynb``` notebook.

Lastly, I converted two documents from the BofA PoC that the team has identified issues with in the past. In the ```docling_error_revision.ipynb``` notebook, I provide prototypes for fixing these issues using built-in Docling capabilities, as well as ideas for how this could be integrated into a UI.

<br>

### Files Overview: docling_core/types/doc

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

### Main Takeaways


#### 1. Interacting with and Saving Docling Documents

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











Misc Ideas:

Text cells sourced from OCR