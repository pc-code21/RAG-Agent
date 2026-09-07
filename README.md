# RAG Agent 

This repository contains two [n8n](https://n8n.io) workflows that together implement a Retrieval-Augmented Generation (RAG) pipeline:

1. **Upload File to Database** – Watches a Google Drive folder, downloads new files, embeds them, and stores them in Pinecone.
2. **RAG Agent** – A conversational AI agent that answers questions by retrieving relevant context from the Pinecone vector store.

---

## Architecture

### 1. Upload File to Database

```
Google Drive Trigger (fileCreated)
        │
        ▼
   Download File
        │
        ▼
 Pinecone Vector Store (insert)
        │
   ┌────┴─────┐
   ▼          ▼
Embeddings   Default Data
Google       Loader
Gemini       (Document)
```

- **Google Drive Trigger** — Fires whenever a new file is created in a watched Drive folder.
- **Download File** — Pulls the binary content of the newly created file.
- **Default Data Loader** — Parses/splits the downloaded document into chunks.
- **Embeddings Google Gemini** — Converts each chunk into a vector embedding using Google's Gemini embedding model.
- **Pinecone Vector Store** — Stores the embedded document chunks in a Pinecone index for later retrieval.

### 2. RAG Agent

```
When Chat Message Received
        │
        ▼
      AI Agent
   ┌────┼─────────────────────┐
   ▼    ▼                     ▼
Chat   Memory              Tool: Answer questions
Model  (Simple             with a vector store
(Open  Memory)                 │
Router)                   ┌────┴─────┐
                           ▼          ▼
                     Pinecone     OpenRouter
                     Vector       Chat Model1
                     Store        (Model)
                           │
                           ▼
                     Embeddings Google
                     Gemini (Embedding)
```

- **When Chat Message Received** — Chat trigger that starts the agent on each incoming user message.
- **AI Agent** — Orchestrates the conversation, decides when to call the retrieval tool, and generates the final response.
- **OpenRouter Chat Model** — The main LLM used by the AI Agent to converse with the user.
- **Simple Memory** — Keeps track of conversation history per session for context-aware replies.
- **Answer Questions with a Vector Store (Tool)** — A vector store tool the agent can call to retrieve relevant document chunks.
  - **Pinecone Vector Store** — The same index populated by the upload workflow; used here for similarity search.
  - **Embeddings Google Gemini** — Embeds the user's query so it can be matched against stored vectors.
  - **OpenRouter Chat Model1** — LLM used to synthesize an answer from the retrieved context.

---

## Prerequisites

- An [n8n](https://n8n.io) instance (self-hosted or cloud)
- A [Google Cloud](https://console.cloud.google.com) project with:
  - Google Drive API enabled (for the trigger and file download)
  - A Gemini API key (for embeddings)
- A [Pinecone](https://www.pinecone.io) account and index
- An [OpenRouter](https://openrouter.ai) API key (for the chat models)

---

## Setup

1. **Import the workflows**
   - In n8n, go to **Workflows → Import from File** and import both `Upload file to database.json` and `RAG Agent.json`.

2. **Configure credentials**
   - **Google Drive OAuth2** — connect your Google account for the trigger and file download nodes.
   - **Google Gemini (PaLM) API** — add your Gemini API key for the embeddings nodes.
   - **Pinecone API** — add your Pinecone API key and select/create the target index (make sure the vector dimension matches the Gemini embedding model's output).
   - **OpenRouter API** — add your OpenRouter API key for both chat model nodes.

3. **Set the watched Drive folder**
   - In the **Google Drive Trigger** node, select the folder you want to monitor for new files.

4. **Verify the Pinecone index**
   - Ensure both workflows point to the **same Pinecone index/namespace**, so documents uploaded by the first workflow are retrievable by the second.

5. **Activate the workflows**
   - Turn on **Upload file to database** so it starts watching Drive automatically.
   - Publish/activate **RAG Agent** so the chat trigger is live.

---

## Usage

1. Drop a new file into the watched Google Drive folder.
2. The **Upload file to database** workflow automatically downloads it, chunks it, embeds it, and upserts it into Pinecone.
3. Open the chat interface for the **RAG Agent** workflow (or connect it to your own front end) and ask questions.
4. The agent retrieves relevant chunks from Pinecone and uses the LLM to generate a grounded answer, while maintaining conversation memory.

---

## Notes

- Make sure the embedding model used at upload time (Gemini) matches the one used at query time, so vector spaces are compatible.
- Simple Memory is session-based; connect a persistent memory store if you need history across sessions/restarts.
- You can swap OpenRouter for any other chat model node supported by n8n without changing the overall architecture.
