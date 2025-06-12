# Docling MCP Exploration

#### Conducted by: Ryan Fernandes, June 2025

<br>

## Summary

The [docling-mcp](https://github.com/docling-project/docling-mcp) repo appears to be in its initial stages, with some extended functionality missing. However, there are already useful tools in the repository that integrate into clients like [Claude for Desktop](https://claude.ai/download). 

For the purposes of our project, we would likely build a client around the Docling-MCP server that allows users to converse with a model in order to refine document conversion, rather than editing the Docling Document themselves in a UI. If this were implemented, it could give us the ability to simultaneously search for/fix conversion errors across multiple documents. However, we must consider if a chat-based editing experience is better for users than a UI, or if the two could work in tandem.

Additionally, we must consider what style of editing would be most suitable for the MCP server. Would results be better when asking the model to reconstruct a Docling Document from scratch, but with a small change, or when asking the model to use tools to change an existing Docling Document? These questions do not have clear answers, and depending on the answer may lead us to contribute certain tools to the exiting Docling-MCP project.

Lastly, we should explore configuring a client other than Claude for Desktop. This would allow us to integrate the tools in Docling-MCP as a part of endpoints/services and within our examples notebook.

## Current Tools

There are 13 tools currently in the Docling-MCP project

```is_document_in_local_cache```

```convert_pdf_document_into_json_docling_document_from_uri_path```

```convert_attachments_into_docling_document```

```create_new_docling_document```

```export_docling_document_to_markdown```

```save_docling_document```

```add_title_to_docling_document```

```add_section_heading_to_docling_document```

```open_list_in_docling_document```

```close_list_in_docling_document```

```add_listitem_to_list_in_docling_document```

```add_table_in_html_format_to_docling_document```

```export_docling_document_to_vector_db```

```search_documents```

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