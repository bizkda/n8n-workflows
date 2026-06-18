# Apple RAG Chatbot — n8n Workflow

An AI-powered chatbot that answers questions about Apple Inc.'s Q4 and full-year FY2019 financial statements using Retrieval-Augmented Generation (RAG). Documents are ingested from Google Drive, embedded into Pinecone, and retrieved at query time by an AI Agent.

---

## How it works

The workflow has two pipelines:

### 1. Knowledge base (ingestion)
Triggered automatically when a new file is added to a watched Google Drive folder. The file is downloaded, chunked, embedded using Ollama, and stored in a Pinecone vector index.

```
Google Drive Trigger → Download file → Pinecone Vector Store
                                              ↑
                               Embeddings Ollama (nomic-embed-text)
                               Default Data Loader
```

### 2. RAG chatbot (retrieval)
Activated when a user sends a chat message. The AI Agent uses the vector store as a tool to look up relevant financial data before answering.

```
Chat Trigger → AI Agent
                  ├── Ollama Chat Model (minimax-m3:cloud)
                  ├── Simple Memory (conversation history)
                  └── Answer questions with vector store [Tool]
                              ├── Pinecone Vector Store1 (namespace: applen8n)
                              ├── Embeddings Ollama1 (nomic-embed-text)
                              └── Ollama Chat Model1 (minimax-m3:cloud)
```

---

## Prerequisites

| Service | Details |
|---|---|
| n8n | Self-hosted instance |
| Google Drive | OAuth2 credentials configured |
| Pinecone | API key, index named `n8nv2` |
| Ollama | Running locally, with `nomic-embed-text` and `minimax-m3:cloud` pulled |

---

## Setup

### 1. Import the workflow
In n8n, go to **Workflows → Import** and paste the workflow JSON.

### 2. Configure credentials
Update these credential references in the workflow:

- **Google Drive** (`googleDriveOAuth2Api`) — connect your Google account
- **Pinecone** (`pineconeApi`) — add your Pinecone API key
- **Ollama** (`ollamaApi`) — point to your Ollama instance URL

### 3. Create the Pinecone index
Create an index named `n8nv2` in your Pinecone dashboard with:
- **Dimensions:** `768` (matches `nomic-embed-text` output)
- **Metric:** `cosine`
- **Namespace:** `applen8n`

### 4. Pull Ollama models
```bash
ollama pull nomic-embed-text
ollama pull minimax-m3:cloud
```

### 5. Upload your document
Add your PDF (e.g. Apple Q4 FY2019 financial report) to the Google Drive folder:
`https://drive.google.com/drive/folders/1SJtEzP2GHVQBsNqFtzqzzlxojvPoiZ54`

The ingestion pipeline triggers automatically and loads the document into Pinecone.

### 6. Activate the workflow
Toggle the workflow to **Active** in n8n. The chat interface will be available via the Chat Trigger webhook.

---

## Configuration reference

### AI Agent — system message
The agent is pre-configured to answer questions about Apple's FY2019 financials:

> You are a financial analyst assistant with access to Apple Inc.'s Q4 and full-year FY2019 financial statements (fiscal year ended September 28, 2019). You can answer questions about revenue by product, revenue by region, profitability, balance sheet, and cash flows. Always cite specific figures in USD millions.

### Vector store tool — description
The tool description tells the agent when to query Pinecone:

> Apple Inc. condensed consolidated financial statements for Q4 and full-year FY2019. Includes income statement, balance sheet, and cash flow statement. All figures in USD millions.

### Key parameters

| Node | Parameter | Value |
|---|---|---|
| Pinecone (ingest) | Index | `n8nv2` |
| Pinecone (ingest) | Namespace | `applen8n` |
| Pinecone (retrieval) | Index | `n8nv2` |
| Pinecone (retrieval) | Namespace | `applen8n` |
| Embeddings | Model | `nomic-embed-text:latest` |
| Chat model | Model | `minimax-m3:cloud` |
| Vector store QA | Limit | 4 (chunks retrieved per query) |

---

## Example questions

Once running, try asking:

- *What was Apple's total revenue in Q4 FY2019?*
- *How did iPhone sales change year-over-year?*
- *Which region generated the most revenue?*
- *What was Apple's net income for the full year?*
- *How much cash did Apple have on the balance sheet?*
- *How much did Apple spend on share buybacks?*

---

## Troubleshooting

**Agent doesn't use the vector store**
Make sure the "Answer questions with a vector store" node is connected to the AI Agent's `Tool` slot (not just floating with a dashed line). Also verify the tool description is filled in — the agent decides whether to call a tool based on that text.

**Empty results from Pinecone**
Run the ingestion pipeline first by uploading a file to the Google Drive folder. Confirm the namespace `applen8n` exists in your Pinecone index before querying.

**Ollama model errors**
Verify both `nomic-embed-text` and `minimax-m3:cloud` are available by running `ollama list`. The Ollama base URL in n8n credentials must be reachable from your n8n instance.

**Google Drive trigger not firing**
The trigger polls every minute. Ensure the workflow is set to **Active** and the watched folder ID matches your Drive folder.

---

## Workflow settings

| Setting | Value |
|---|---|
| Execution order | v1 |
| Timezone | Africa/Algiers |
| Caller policy | Workflows from same owner |
| Binary mode | Separate |
