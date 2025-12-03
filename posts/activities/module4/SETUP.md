# 📌 flow_along

This repository contains workflows and resources to build **domain-specific chatbots** using **n8n**.  
Each use case demonstrates how to upload documents, extract and embed knowledge, and query through Telegram or Whatsapp or n8n chat.  

---

## 🚀 Use Cases

### 1. 🎓 Syllabus Advisor Bot  
**Domain:** Education  
- **Upload:** Subject-wise syllabus PDFs  
- **Embedding Target:** Pinecone  
- **Telegram Query Example:**  
  > “What is the marks split for Maths?”

---

### 2. 🛠 API Manual Helper  
**Domain:** DevTools  
- **Upload:** API manuals (PDFs)  
- **Embedding Target:** Extract API methods, parameters, and examples  
- **Telegram Query Example:**  
  > “What’s the use of GET /users API?”

---

### 3. 📝 Meeting Summary Bot  
**Domain:** Corporate Communications  
- **Upload:** Meeting transcripts  
- **Embedding Target:** Speaker actions & decisions  
- **Telegram Query Example:**  
  > “What did Ravi say about the budget?”

---

### 4. 👔 Resume Finder Bot  
**Domain:** Human Resources  
- **Upload:** Resumes (Google Drive or Local)  
- **Embedding Target:** Skills and experience  
- **Telegram Query Example:**  
  > “Find Python + 3 YOE resumes”

---

### 5. 📌 Jira Docs Helper  
**Domain:** DevOps / Product Management  
- **Upload:** PRDs or Jira tickets  
- **Embedding Target:** Stories & tasks  
- **Telegram Query Example:**  
  > “Which story involves frontend development?”

---

## ⚙️ Tech Stack
- **n8n** – Workflow automation  
- **OpenAI / LLM** – Q&A and semantic search  
- **Pinecone** – Vector database for embeddings  
- **Telegram Bot API** – User interface for querying  
- **Google Drive / HTTP** – Document source  

---

## 🗂 Workflow Overview
1. **Document Upload** → Upload syllabus, API manuals, transcripts, resumes, or Jira docs.  
2. **Text Extraction** → Convert PDF, DOCX, or transcripts into plain text.  
3. **Chunking & Embedding** → Split text into smaller parts and embed into **Pinecone**.  
4. **Telegram Query** → User asks questions → workflow retrieves context → LLM generates answers.  

---

## 📥 Getting Started
1. Clone the repository:  
Import workflows (.json) into n8n.

Configure environment variables:

OPENAI_API_KEY

PINECONE_API_KEY

TELEGRAM_BOT_TOKEN or WHATSAPP

Upload relevant documents for each use case.

Start querying via Telegram bot.

✅ Example Queries
Education: “What is the marks split for Maths?”

DevTools: “What’s the use of GET /users API?”

Corp Comms: “What did Ravi say about the budget?”

HR: “Find Python + 3 YOE resumes”

DevOps: “Which story involves frontend development?”

🙌 Contribution
Feel free to fork, enhance workflows, and raise pull requests.
Each use case can be extended with additional domains and data sources.



