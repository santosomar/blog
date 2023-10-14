---
title: "AI BOMs: Ensuring Transparency and Traceability"
thumbnail: /blog/assets/agents-js/thumbnail.png
authors:
  - user: santosomar
---

# AI BOMs: Ensuring Transparency and Traceability

As the adoption of artificial intelligence (AI) and machine learning (ML) systems becomes more widespread, the need for transparency, and traceability in AI development has never been more crucial. Supply chain security is top-of-mind for many individuals in the industry. This is why AI Bill of Materials (AI BOMs) are so important. But what exactly are AI BOMs, and why are they so important? Let's dive in.

## What is an AI BOM?
Much like a traditional Bill of Materials in manufacturing that lists out all the parts and components of a product, an AI BOM provides a detailed inventory of all components of an AI system. But, what about Software Bill of Materials (SBOMs)? How are they different from AI BOMs? In the case of SBOMs, they are used to document the components of a software application. However, AI BOMs are used to document the components of an AI system, including the model, training data, and more.

[Ezi Ozoani](https://huggingface.co/Ezi), [Marissa Gerchick](https://huggingface.co/Marissa), and [Margaret Mitchell](https://huggingface.co/meg) introduced the concept of AI Model Cards in a blog post in 2022. Since then, AI BOMs continue to evolve. Manifest (a supply chain security company) also introduced [an AI BOM concept](https://github.com/manifest-cyber/ai-bom) that is being suggested to be included in [OWASP's CycloneDX](https://cyclonedx.org/) and the Linux Foundation also created [a project to standardize AI BOMs](https://spdx.dev/learn/areas-of-interest/ai/). 

I created a proposed [JSON schema](https://github.com/manifest-cyber/ai-bom/pull/31) for the AI BOM elements that Manifest introduced. This JSON schema describes the structure of an AI BOM document and defines which fields are required and which are optional, as well as the expected data types for each field. You can use this schema to validate any AI BOM documents to ensure they meet the specification outlined.

A visual representation of the AI BOM schema can be found below:
![AI BOM schema visualization](https://imgur.com/a/tzROBGr)

I created [a visualizer tool](https://aibomviz.aisecurityresearch.org/) that can be used to visualize the AI BOM schema, as shown above.

## Why AI BOMs are Essential

1. **Transparency and Trust**: AI BOMs ensure that every element used in an AI solution is documented. This transparency fosters trust among users, developers, and stakeholders.

2. **Supply Chain Security and Quality Assurance**: With a detailed BOM, developers and auditors can assess the quality, reliability, and security of an AI system.

3. **Troubleshooting**: In cases of system failures or biases, AI BOMs can facilitate the quick identification of the problematic component.


## Main Components of an AI BOM

- **Model Details**: This includes the model's name, version, type, creator, and more. 
- **Model Architecture**: Details about the model's training data, design, input and output types, base model, and more.
- **Model Usage**: Outlines the model's intended usage, prohibited uses, and potential misuse.
- **Model Considerations**: Information about the model's environmental and ethical implications.
- **Model Authenticity or Attestations**: A digital endorsement by the model's creator to vouch for the AI-BOM's authenticity.

## AI BOM Proposed Schema
The following is the proposed JSON schema based on Manifest's original AI BOM concept. This schema is currently being reviewed and suggestions are welcomed.

```json
{
    "$schema": "http://json-schema.org/draft-07/schema#",
    "title": "AI BOM",
    "type": "object",
    "properties": {
      "ModelDetails": {
        "type": "object",
        "properties": {
          "Name": { "type": "string" },
          "Version": { "type": "string" },
          "Type": { "type": "string" },
          "Author": { "type": "string" },
          "Licenses": { "type": "array", "items": { "type": "string" } },
          "Libraries": { "type": "array", "items": { "type": "string" }, "required": false },
          "Source": { "type": "string" },
          "BOMGeneration": {
            "type": "object",
            "properties": {
              "Timestamp": { "type": "string" },
              "Method": { "type": "string" },
              "GeneratedBy": { "type": "string" }
            },
            "required": false
          },
          "OtherReferences": { "type": "array", "items": { "type": "string" }, "required": false },
          "Tags": { "type": "array", "items": { "type": "string" }, "required": false }
        },
        "required": ["Name", "Version", "Type", "Author", "Licenses", "Source"]
      },
      "ModelArchitecture": {
        "type": "object",
        "properties": {
          "Datasets": {
            "type": "array",
            "items": {
              "type": "object",
              "properties": {
                "Name": { "type": "string" },
                "Source": { "type": "string" },
                "Usage": { "type": "string" }
              },
              "required": ["Name", "Source"]
            }
          },
          "Architecture": { "type": "string", "required": false },
          "ArchitectureFamily": { "type": "string", "required": false },
          "ParentModel": { "type": "object", "properties": {
              "Name": { "type": "string" },
              "Version": { "type": "string" },
              "Source": { "type": "string" }
            }, "required": false
          },
          "BaseModel": { "type": "object", "properties": {
              "Name": { "type": "string" },
              "Version": { "type": "string" },
              "Source": { "type": "string" }
            }, "required": false
          },
          "Input": { "type": "string" },
          "Output": { "type": "string" },
          "Hardware": { "type": "string", "required": false },
          "Software": { "type": "string", "required": false },
          "SoftwareRequiredForExecution": { "type": "boolean" }
        },
        "required": ["Datasets", "Input", "Output", "SoftwareRequiredForExecution"]
      },
      "Usage": {
        "type": "object",
        "properties": {
          "IntendedUse": { "type": "string" },
          "OutOfScopeUsage": { "type": "string" },
          "MisuseOrMaliciousUse": { "type": "string" }
        },
        "required": ["IntendedUse", "OutOfScopeUsage", "MisuseOrMaliciousUse"]
      },
      "Considerations": {
        "type": "object",
        "properties": {
          "EnvironmentalImpact": { "type": "string", "required": false },
          "EthicalConsiderations": { "type": "string", "required": false }
        },
        "required": []
      },
      "Authenticity": {
        "type": "object",
        "properties": {
          "Authenticity": { "type": "string", "required": false }
        },
        "required": []
      }
    },
    "required": ["ModelDetails", "ModelArchitecture", "Usage"]
  }

```

## The Future of AI BOMs
With AI becoming an integral part of our lives and potential regulations in the horizons, AI BOMs are set to play a pivotal role in ensuring that AI models are developed and deployed responsibly. As AI systems become more complex, the need for AI BOMs will only grow, serving as a testament to the commitment to ethical AI practices.

## References and Further Reading:
- [Manifest AI BOM concept](https://github.com/manifest-cyber/ai-bom) that is being suggested to be included in 
- [OWASP's CycloneDX](https://cyclonedx.org/)
- [SPDX AI](https://spdx.dev/learn/areas-of-interest/ai/)
- [AI BOM schema visualization tool](https://aibomviz.aisecurityresearch.org/)

