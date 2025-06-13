# Docling MCP Exploration

#### Conducted by: Ryan Fernandes, June 2025

<br>

## Summary

The [docling-mcp](https://github.com/docling-project/docling-mcp) repo appears to be in its initial stages, with some extended functionality missing. However, there are already useful tools in the repository that integrate into clients like [Claude for Desktop](https://claude.ai/download). 

For the purposes of our project, we would likely build a client around the Docling-MCP server that allows users to converse with a model in order to refine document conversion, rather than editing the Docling Document themselves in a UI. If this were implemented, it could give us the ability to simultaneously search for/fix conversion errors across multiple documents. However, we must consider if a chat-based editing experience is better for users than a UI, or if the two could work in tandem.

Additionally, we must consider what style of editing would be most suitable for the MCP server. Would results be better when asking the model to reconstruct a Docling Document from scratch, but with a small change, or when asking the model to use tools to change an existing Docling Document? These questions do not have clear answers, and depending on the answer may lead us to contribute certain tools to the exiting Docling-MCP project.

Lastly, we should explore configuring a client other than Claude for Desktop. This would allow us to integrate the tools in Docling-MCP as a part of endpoints/services and within our examples notebook.

## Current Tools

There are 15 tools currently in the Docling-MCP project:

**[Conversion](https://github.com/docling-project/docling-mcp/blob/main/docling_mcp/tools/conversion.py)**

```is_document_in_local_cache``` - This is a utility function that can be utilized by the model before taking other actions on a document. It checks if a cache key (string) that is used to access existing documents is present in the local cache.

```convert_pdf_document_into_json_docling_document_from_uri_path``` - This function can accept either relative paths or URI's to remote files (PDF's). It will generate a cache key based on the source of the document, and will return if this key already exists. If the cache key is new, the tool converts the PDF to a Docling Document and stores it in the local document cache. Lastly, it adds a text item as furniture (not visible in markdown conversion) to the body of the document that specifies the original source of the converted document. This text item is added to the stack cache (reference for last item added to a document)

```convert_attachments_into_docling_document``` - This function is supposed to support conversions for PDF files attached in Claude Desktop. However, it does not appear to have this functionality added as of yet.

**[Generation](https://github.com/docling-project/docling-mcp/blob/main/docling_mcp/tools/generation.py)**

```create_new_docling_document``` - This function instantiates a blank Docling Document that other tools can add to. The function creates a cache key for the document using the prompt passed into the tool and adds this prompt as a furniture text item (not visible in markdown conversion) to the body of the document. This text item is added to the stack cache as a starting anchor for where to add content to the document.

```export_docling_document_to_markdown``` - This function simply executes the ```export_to_markdown()``` function on the Docling Document with the given cache key. It returns this markdown export as a string, formatted alongside the document's key. This function was posed to the client in prompts as a way to check/see the results of other tools that it has applied to a document.

```save_docling_document``` - This function saves the JSON and markdown versions of the Docling Document with the specified cache key in the cache directory (on the disk). By default, this cache directory sits within the docling-mcp directory, but its path can be specified as an environment variable. The files are named with their cache key. There don't appear to be checks for file overwrite.

```add_title_to_docling_document``` - This function implements the ```add_title()``` function and appends a title with the specified text to the Docling Document. Titles cannot live inside of lists, so there is a check to ensure that the top of the local stack cache is not a list/ordered list. After adding the title, the stack cache is updated to have this title as the top item.

```add_section_heading_to_docling_document``` - This function implements the ```add_heading()``` function and appends a section heading with the specified text/heading level to the Docling Document. Headings cannot live inside of lists, so there is a check to ensure that the top of the local stack cache is not a list/ordered list. After adding the section heading, the stack cache is updated to have this heading as the top item.

```add_paragraph_to_docling_document``` - This function implements the ```add_text()``` function and appends a "paragraph" with the specified text to the Docling Document. Paragraphs cannot live inside of lists, so there is a check to ensure that the top of the local stack cache is not a list/ordered list. After adding the paragraph, the stack cache is updated to have this text item as the top item.


```open_list_in_docling_document``` - This function implements the ```add_group()``` function and appends a new list to the body of the Docling Document. Once the group is opened, it is appended to the top of the stack cache. There are no checks to see if another group is open, and the functionality of this tool will place groups after one other when multiple are open, rather than nesting them.

```close_list_in_docling_document``` - This function runs ```pop()``` on the stack cache, which will remove groups that have been placed at the top of the cache. By removing the group from the cache, list items will no longer be added to it, effectively "closing" the group.

```add_listitem_to_list_in_docling_document``` - This function takes three arguments: the document key, text for the list item, and marker for the list item (bullet, number, etc). The function will ensure that a group item sits at the top of the cache. If it does, the function calls ```add_list_item()``` with the group item at the top of the cache set as the parent. This funtion does not touch the stack cache after adding its item, leaving the group item "open."

```add_table_in_html_format_to_docling_document``` - This tool uses the Docling document converter to convert an HTML table passed as a string to a Docling table. It then accesses the data from this Docling table and appends it to the body of the Docling Document stored in the cache. If there is a group open, no error will be thrown, but the table will be added to the end of the body, not to the group. This function seems a bit less refined than the others, and it is not used in the example Claude Desktop integrations.

**[Applications](https://github.com/docling-project/docling-mcp/blob/main/docling_mcp/tools/applications.py)**

```export_docling_document_to_vector_db``` - This function takes a document key as a string and accesses the Docling Document associated with that key. It then uses ```VectoreStoreIndex```, ```StoreContext```, and ```Document``` from ```llama_index.core``` in order to create an index with a vector database of the document. This index is stored in the local index cache under the key "milvus_index".

```search_documents``` - This function accepts a query string and queries the index in the local index cache under the key "milvus_index." It then returns the response from this query, if there is one. The docstring defines the querying as "using semantic similarity to find content that best matches the query, rather than simple keyword matching."

## Client Performance

I tested two prompts that trigger tools from the Docling-MCP server, both provided as examples in the Docling-MCP repository. Here are my impressions from both:

**#1: Converting documents**

Above, there are two tools for converting documents: ```convert_pdf_document_into_json_docling_document_from_uri_path```, and ```convert_attachments_into_docling_document```. I triggered the first tool by sending the prompt: "Convert the PDF document at /Users/ryan/Documents/code/claude-access/02_BofA_CoreChecking_en_ADA.pdf into DoclingDocument and return its document-key." Immediately, the ```convert_pdf_document_into_json_docling_document_from_uri_path``` tool was triggered, and the conversion completed as normal. At the end of the conversion, the document-key to access the document from the local_document_cache was provided. 

Following this conversion, I prompted Claude Desktop: "Please save the DoclingDocument with that key to my disk." This successfully triggered the ```save_docling_document``` tool as intended, and a ```_cache``` directory was created where my DoclingDocument was stored as Markdown and JSON. 

Lastly, I did not want the file names to be the document keys, so I sent the prompt: "Please move these documents from the _cache directory to the converted directory in Users/ryan/Documents/code/claude-access, and rename them to match the name of the original PDF." This triggered the ```move_file``` tool from Anthropic's ```filesystem``` server, which correctly moved/renamed the files.

Overall, the document conversion process was smooth and had no errors. However, there is little-to-no room for customizing the settings of conversion, and for the pipeline/product that we are building, it seems unnecessarily complex to make users interact with an LLM to convert documents. With that being said, these tools could be used as a single step when answering a different prompt that does not explicitly direct document conversion.

**#2. Creating documents**

In order to test the document creation capabilities provided by the Docling-MCP server, I again used the default prompt provided in the Docling-MCP repo:

"I want you to write a Docling document. To do this, you will create a document first by invoking create_new_docling_document. Next you can add a title (by invoking add_title_to_docling_document) and then iteratively add new section-headings and paragraphs. If you want to insert lists (or nested lists), you will first open a list (by invoking open_list_in_docling_document), next add the list_items (by invoking add_listitem_to_list_in_docling_document). After adding list-items, you must close the list (by invoking close_list_in_docling_document). Nested lists can be created in the same way, by opening and closing additional lists.
During the writing process, you can check what has been written already by calling the export_docling_document_to_markdown tool, which will return the currently written document. At the end of the writing, you must save the document and return me the filepath of the saved document.
The document should investigate the impact of tokenizers on the quality of LLM's."

When I entered this prompt, the model immediately knew to use the ```create_new_docling_document``` tool, which gave it a blank DoclingDocument and a document-key for the local_document_cache.

From there, the model was able to utilize all of the tools available in ```generation.py```, with the exception of ```add_table_in_html_format_to_docling_document```, which it did not utilize. Granted, this tool was not explicitly mentioned in the prompt. At one point, the model made an incorrect call to the ```add_paragraph_to_docling_document``` tool, as it mislabeled the 'paragraph' parameter in the call as 'parameter.' However, when the model received the error message response, it quickly made a revised call to the tool with the correct parameter name and continued on, which was very impressive. 

Once it was done generating, the model used the ```export_docling_document_to_markdown``` to "check" the quality of the document so far. In its output, the model noted that the document looked "comprehensive and well-structured," and moved on to the next step. However, I wonder how the model would respond if the markdown version of the document was not as expected. How would it approach editing the Docling Document with only the tools available?

Lastly, the model correctly used the ```save_docling_document``` tool, and the new document was saved to the _cache directory. The generated document was indeed of high quality, although it was quite simple given that only text item tools were provided to it. I wonder how it would perform/if it would be capable of using more complex tools to construct more complex documents.

## Next Steps

There is currently a draft PR from Peter that adds some document manipulation tools to the MCP server. However, this PR is 2 months old and is still a draft. It also only adds functionality for getting the "anchors" (cref's and text for each NodeItem) of the document, getting the text of an item based on its anchor, updating the text of an item, and deleting items based on their anchors. There is also a draft PR from Peter that was last updated 2 weeks ago that will add Llama Stack examples for connecting to the Docling-MCP server. Given this information, there are some potential next steps:

- Connect an MCP client in the notebook/hosted as a service to the Docling-MCP server
- **Contribute more tools for document manipulation to the Docling-MCP repo**
- Try to get document editing to work with the current tools that are provided (use the "recreate document with changes" strategy)
- **Explore connecting this LLM editing approach to a Rendered Docling Document-esque UI where users can select the NodeItems they want and describe the changes that should be made/the issues that exist**