# Prompt Specification

This document defines the prompt used to classify conversations using OpenAI.

The prompt determines:

- issue_category
- confidence_score
- resolution_status
- conversation_status

## Prompt Goals

- Identify the main intent of the message
- Classify conversation type
- Extract structured labels

## Example Input

Customer message:
"Здравствуйте, когда будет доставка?"

## Example Output

issue_category: delivery_question
confidence_score: 0.92
resolution_status: unresolved
conversation_status: open
