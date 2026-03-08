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

A[Customer Messages] --> B[Conversation Extraction]
B --> C[Make Automation]
C --> D[OpenAI Classification]
D --> E[JSON Parser]
E --> F[Airtable Dataset]
```
