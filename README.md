# Bank Statement AI Automation

An n8n workflow that watches a local folder for bank statement files, keeps a Qdrant vector store in sync with the contents of that folder, and provides a chat interface for asking questions about the statements using Mistral AI.

## What This Does

The workflow has two parts that work together.

### File sync pipeline
Watches a folder on your machine for changes and keeps a Qdrant collection up to date automatically.

* A new file is added: it gets read, embedded, and inserted into Qdrant.
* An existing file is changed: the old vector for that file is deleted and a fresh one is inserted.
* A file is deleted: the matching vector is removed from Qdrant.

### Chat pipeline
A separate chat trigger lets you ask natural language questions about the contents of everything currently stored in Qdrant. It retrieves relevant chunks and passes them to Mistral for an answer.

## Prerequisites

* n8n running locally (via npm global install or Docker)
* A Qdrant Cloud account with a collection created (1024 dimensions, Cosine distance, since that matches Mistral's embedding output size)
* A Mistral AI account with an API key
* A local folder for your bank statements

## Setup

### 1. Install and run n8n
```
npm install n8n -g
```

Since this workflow uses the Local File Trigger and Read/Write Files From Disk nodes, both of which are restricted by default, start n8n with the following environment variables set:
```
$env:NODES_EXCLUDE="[]"
$env:N8N_RESTRICT_FILE_ACCESS_TO="C:\path\to\your\statements\folder"
n8n
```
Adjust the path to match your own folder. On Mac or Linux, use `export` instead of `$env:`.

### 2. Create your Qdrant collection
In the Qdrant Cloud dashboard, create a collection with:
* Vector size: 1024
* Distance metric: Cosine

Note the collection name, cluster URL, and API key. You will need all three.

### 3. Create a payload index
Qdrant requires an index on any field you plan to filter by. This workflow filters by filename, so run this once against your Qdrant instance before using the workflow (a temporary HTTP Request node in n8n works fine for this, or any HTTP client):
```
PUT https://YOUR_CLUSTER_URL:6333/collections/YOUR_COLLECTION/index
```
Body:
```json
{
  "field_name": "metadata.filter_by_filename",
  "field_schema": "keyword"
}
```

### 4. Get a Mistral API key
Sign up at console.mistral.ai, create a workspace, and generate an API key under API Keys.

### 5. Import the workflow
In n8n, go to Workflows, then Import from File, and select `bank-statement-automation.json`.

### 6. Configure credentials
The workflow needs two credentials set up in n8n:

**Qdrant Cloud account (Header Auth)**
Used by the HTTP Request nodes that manage the delete and search logic.
* Header name: `api-key`
* Header value: your Qdrant API key

**Qdrant account (Qdrant API credential type)**
Used by the Qdrant Vector Store nodes.
* URL: your Qdrant cluster URL
* API key: your Qdrant API key

**Mistral Cloud account**
Used by all Mistral nodes (embeddings and chat model).
* API key: your Mistral API key

### 7. Update the file paths and collection name
Open the following nodes and replace the placeholder values with your own:
* Local File Trigger: set Folder to Watch
* Edit Fields: update the `directory` value in the JSON output to match your folder, and set `qdrant_collection` to your actual collection name
* Every HTTP Request node (Existing Point, HTTP Request, Delete Existing Point, Delete Existing Point(Changed)): update the collection name in the URL
* Both Qdrant Vector Store nodes: update the Qdrant Collection field

### 8. Publish and test
Publish the workflow, then drop a text file into your watched folder. Check the Executions tab to confirm it ran successfully and check your Qdrant dashboard to confirm a point was added.

To test the chat side, click Open Chat in the editor and ask a question about the content of a file you added.

## Files In This Repository

* `bank-statement-automation.json`: the n8n workflow, ready to import
* `.env.example`: reference list of the values you will need to fill in during setup
* `README.md`: this file

## Notes On File Encoding

If you are creating test files manually, save them as plain UTF8 text. Files saved with unusual encodings, or files that are empty when the trigger fires, can produce garbled or empty content once embedded. The workflow uses an Extract From File node to read file contents as text, which handles standard UTF8 files reliably.

## Additional Use Cases

This same pattern (watch a folder, sync a vector store, chat over the contents) is not limited to bank statements. A few directions you could take it:

**Personal document assistant**
Point it at a folder containing contracts, insurance policies, or warranty documents. Ask things like "when does my car insurance renew" or "what is the cancellation policy on my lease" instead of digging through PDFs.

**Meeting notes and transcripts**
Drop meeting notes or call transcripts into the folder as they are exported. Ask the chat "what did we decide about the Q3 budget" or "summarize every mention of the client onboarding project."

**Research paper or article library**
Use it to hold PDFs or text exports of research papers, articles, or clippings you have saved. Ask comparative questions across multiple documents at once, such as "which of these papers mention the same dataset."

**Codebase or documentation search**
Point the folder at a directory of internal documentation or README files across projects. Ask "which service handles authentication" or "where is the deployment process documented" without manually searching each repo.

**Expense and receipt tracking**
Similar to the bank statement use case, but for receipts. Export or photograph receipts as text (using OCR beforehand if needed), drop them into the folder, and ask spending questions like "how much did I spend on software subscriptions this year."

**Support ticket or customer feedback analysis**
Feed exported support tickets or feedback forms into the folder. Ask "what are the most common complaints about the checkout flow" to get a natural language summary instead of reading each one.

**Legal document review**
For contracts or agreements saved as text, ask questions like "which of these contracts have an auto renewal clause" across a whole folder of documents at once.

In every case, the underlying mechanism is the same: the file sync branch keeps Qdrant current with whatever is in the folder, and the chat branch lets you query that content conversationally through Mistral. Swap the folder contents and adjust the metadata fields if you want more specific filtering (by client, by project, by date range, and so on).
