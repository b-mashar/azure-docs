---
title: Use AI to understand Blob data
titleSuffix: Azure Search
description: Add semantic, natural language processing and image analysis to Azure blobs using an AI enrichment pipeline in Azure Search.

manager: nitinme
author: HeidiSteen
ms.author: heidist
ms.service: search
ms.topic: conceptual
ms.date: 10/09/2019
---

# Use AI to understand Blob data

Data in Azure Blob storage is often a variety of unstructured content such as images, long text, PDFs, and Office documents. By using the AI capabilities in Azure Search, you can understand and extract valuable information from blobs in a variety of ways. Examples of applying AI to blob content include:

+ Extract text from images using optical character recognition (OCR)
+ Produce a scene description or tags from a photo
+ Detect language and translate text into different languages
+ Process text with named entity recognition (NER) to find references to people, dates, places, or organizations 

While you might need just one of these AI capabilities, it’s common to combine multiple of them into the same pipeline (for example, extracting text from a scanned image and then finding all the dates and places referenced in it). 

AI enrichment creates new information, captured as text, stored in fields. Post-enrichment, you can access this information from a search index through full text search, or send enriched documents back to Azure storage to power new application experiences that include exploring data for discovery or analytics scenarios. 

In this article, we view AI enrichment through a wide lens so that you can quickly grasp the entire process, from transforming raw data in blobs, to queryable information in either a search index or a knowledge store.

## What it means to "enrich" blob data

*AI enrichment* is part of the indexing architecture of Azure Search that integrates built-in AI from Microsoft or custom AI that you provide. It helps you implement end-to-end scenarios where you need to process blobs (both existing ones and new ones as they come in or are updated), crack open all file formats to extract images and text, extract the desired information using various AI capabilities, and index them in an Azure Search index for fast search, retrieval and exploration. 

Inputs are your blobs, in a single container, in Azure Blob storage. Blobs can be almost any kind of text or image data. 

Output is always an Azure Search index, used for fast text search, retrieval, and exploration in client applications. Additionally, output can also be a *knowledge store* that projects enriched documents into Azure blobs or Azure tables for downstream analysis in tools like Power BI or in data science workloads.

In between is the pipeline architecture itself. The pipeline is based on the *indexer* feature, to which you can assign a *skillset*, which is composed of one or more *skills* providing the AI. The purpose of the pipeline is to produce *enriched documents* that enter as raw content but pick up additional structure, context, and information while moving through the pipeline. Enriched documents are consumed during indexing to create inverted indexes and other structures used in full text search or exploration and analytics.

## How to get started

You can start directly in your storage account portal page. Click **Add Azure Search** and create a new Azure Search service or select an existing one. If you already have an existing search service in the same subscription, clicking **Add Azure Search** opens the Import data wizard so that you can immediately step through indexing, enrichment, and index definition.

Once you add Azure Search to your storage account, you can follow the standard process to enrich data in any Azure data source. Assuming you already have blob content, you can use the Import data wizard in Azure Search for an easy initial introduction to AI enrichment. This quickstart explains the steps: [Create an AI enrichment pipeline in the portal](cognitive-search-quickstart-blob.md). 

In the following sections, we'll explore more components and concepts.

## Begin with Blob indexers

AI enrichment is an add-on to an indexing pipeline, and in Azure Search, those pipelines are built on top of an *indexer*. An indexer is a data-source-aware subservice equipped with internal logic for sampling data, reading metadata data, retrieving data, and serializing data from native formats into JSON documents for subsequent import. Indexers are often used by themselves for import, separate from AI, but if you want to build an AI enrichment pipeline, you will need an indexer and a skillset to go with it. In this section, we'll focus on the indexer itself.

Blobs in Azure Storage are indexed using the [Azure Search Blob storage indexer](search-howto-indexing-azure-blob-storage.md). You invoke this indexer by setting the type, and by providing connection information that includes an Azure Storage account along with a blob container. Unless you've previously organized blobs into a virtual directory, which you can then pass as a parameter, the Blob indexer pulls from the entire container.

An indexer does the "document cracking", and after connecting to the data source, it's the first step in the pipeline. For blob data, this is where PDF, office docs, image, and other content types are detected. Document cracking with text extraction is no charge. Document cracking with image extraction is charged at rates you can find on the Azure Search [pricing page](https://azure.microsoft.com/pricing/details/search/).

Although all documents will be cracked, enrichment only occurs if you explicitly provide the skills to do so. For example, if your pipeline consists exclusively of text analytics, any images in your container or documents will be ignored.

The Blob indexer comes with configuration parameters and supports change tracking if the underlying data provides sufficient information. You can learn more about the core functionality in [Azure Search Blob storage indexer](search-howto-indexing-azure-blob-storage.md).

## Add AI

*Skills* are the individual components of AI processing that you can use standalone or in combination with other skills for sequential processing. 

+ Built-in skills are backed by Cognitive Services, with image analysis based on Computer Vision, and natural language processing based on Text Analytics. A few examples are [OCR](cognitive-search-skill-ocr.md), [Entity Recognition](cognitive-search-skill-entity-recognition.md), and [Image Analysis](cognitive-search-skill-image-analysis.md). You can review the full list of built-in skills in [Predefined skills for content enrichment](cognitive-search-predefined-skills.md).

+ Custom skills are custom code, wrapped in an interface definition that allows for integration into the pipeline. In customer solutions, it's common practice to use both, with custom skills providing open-source, third-party, or first-party AI modules.

A *skillset* is the collection of skills used in a pipeline, and it's invoked after the document cracking phase makes content available. An indexer can consume exactly one skillset, but that skillset exists independently of an indexer so that you can reuse it in other scenarios.

Custom skills might sound complex but can be simple and straightforward in terms of implementation. If you have existing packages that provide pattern matching or classification models, the content you extract from blobs could be passed to these models for processing. Since AI enrichment is Azure-based, your model should be on Azure also. Some common hosting methodologies include using [Azure Functions](cognitive-search-create-custom-skill-example.md) or [Containers](https://github.com/Microsoft/SkillsExtractorCognitiveSearch).

Built-in skills backed by Cognitive Services require an [attached Cognitive Services](cognitive-search-attach-cognitive-services.md) all-in-one subscription key that gives you access to the resource. An all-in-one key gives you image analysis, language detection, text translation, and text analytics. Other built-in skills are features of Azure Search and require no additional service or key. Text shaper, splitter, and merger are examples of helper skills that are sometimes necessary when designing the pipeline.

If you use only custom skills and built-in utility skills, there is no dependency or costs related to Cognitive Services.

<!-- ## Order of operations

Now we've covered indexers, content extraction, and skills, we can take a closer look at pipeline mechanisms and order of operations.

A skillset is a composition of one or more skills. When multiple skills are involved, the skillset operates as sequential pipeline, producing dependency graphs, where output from one skill becomes input to another. 

For example, given a large blob of unstructured text, a sample order of operations for text analytics might be as follows:

1. Use Text Splitter to break the blob into smaller parts.
1. Use Language Detection to determine if content is English or another language.
1. Use Text Translator to get all text into a common language.
1. Run Entity Recognition, Key Phrase Extraction, or Sentiment Analysis on chunks of text. In this step, new fields are created and populated. Entities might be location, people, organization, dates. Key phrases are short combinations of words that appear to belong together. Sentiment score is a rating on continuum of negative (0) to positive (1) sentiment.
1. Use Text Merger to reconstitute the document from the smaller chunks. -->

## How to use AI-enriched content

The output of AI enrichment is either a search index on Azure Search, or a knowledge store in Azure Storage.

In Azure Search, a search index is used for interactive exploration using free text and filtered queries in a client app. Enriched documents created through AI are formatted in JSON and indexed in the same way all documents are indexed in Azure Search, leveraging all of the benefits an indexer provides. For example, during indexing, the blob indexer refers to configuration parameters and settings to utilize any field mappings or change detection logic. Such settings are fully available to regular indexing and AI enriched workloads. Post-indexing, when content is stored on Azure Search, you can build rich queries and filter expressions to understand your content.

In Azure Storage, a knowledge store has two manifestations: a blob container, or tables in Table storage. 

+ A blob container captures enriched documents in their entirety, which is useful if you want to feed into other processes. 

+ In contrast, Table storage can accommodate physical projections of enriched documents. You can create slices or layers of enriched documents that include or exclude specific parts. For analysis in Power BI, the tables in Azure Table storage become the data source for further visualization and exploration.

An enriched document at the end of the pipeline differs from its original input version by the presence of additional fields containing new information that was extracted or generated during enrichment. As such, you can work with a combination of original and created content, regardless of which output structure you use.

## Next steps

There’s a lot more you can do with AI enrichment to get the most out of your data in Azure Storage, including combining Cognitive Services in different ways, and authoring custom skills for cases where there’s no existing Cognitive Service for the scenario. You can learn more by following the links below.

> [!div class="nextstepaction"]
> [AI enrichment overview](cognitive-search-concept-intro.md) 
> [Create a skillset](cognitive-search-defining-skillset.md)
> [Map nodes in an annotation tree](cognitive-search-output-field-mapping.md)
