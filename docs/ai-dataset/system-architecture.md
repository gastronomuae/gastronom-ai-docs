# System Architecture

## Data Flow

WhatsApp Web  
↓  
Chrome Extraction Script  
↓  
JSON Conversation Export  
↓  
Make.com Webhook  
↓  
OpenAI Classification  
↓  
JSON Parser  
↓  
Iterator  
↓  
Airtable Dataset

## Pipeline Diagram

```mermaid
flowchart TD
A[Customer Messages]
B[Conversation Extraction]
C[Make Automation]
D[OpenAI Classification]
E[JSON Parser]
F[Airtable Dataset]

A --> B --> C --> D --> E --> F
```
