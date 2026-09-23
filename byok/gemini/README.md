# Gemini request vectors

These Gemini 3.8 Flash fixtures use `thinkingLevel` without `thinkingBudget`, and no `temperature`: Gemini 3.8 removed sampling controls. Google documents that combining both controls in one request returns HTTP 400. The Desktop adapter already selects the control by model family; these vectors pin that request shape for image, PDF and text inputs.

Source: [Google Gemini 3 developer guide](https://ai.google.dev/gemini-api/docs/generate-content/gemini-3), checked September 8, 2026.

The `sha256` field hashes the compact, sorted-key JSON in `request`. Endpoint, model, prompts, response schemas and attachment bodies are unchanged.
