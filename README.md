# 🦉 LPU UMS AI Assistant (RAG Chatbot)

An intelligent, production-grade Retrieval-Augmented Generation (RAG) chatbot designed for the **University Management System (UMS)**. This assistant helps students and staff navigate university life by providing instant, accurate answers about admissions, fees, courses, placements, and more.

---

## ✨ Key Features
- **Intelligent RAG Pipeline**: Leverages **LlamaIndex** with **Amazon Bedrock** for precise context-aware answering.
- **Premium UI/UX**: A modern, responsive interface built with vanilla CSS, featuring glassmorphism and smooth micro-animations.
- **Multi-Model Support**: Integrated with **Claude 3 (Amazon Bedrock)** and **GPT-4o (GitHub Models)** for high-quality inference.
- **Enterprise Security**: Production-ready security settings including SSL enforcement, CSRF protection, and secure session management.
- **Containerized Deployment**: Fully dockerized and optimized for **AWS App Runner**.

---

## 📸 Demo Screenshots

<div align="center">
  <img src="static/images/demo1.jpeg" alt="Chatbot answering chancellor query" width="45%" />
  &nbsp;
  <img src="static/images/demo2.jpeg" alt="Chatbot answering emergency numbers query" width="45%" />
</div>

---


## 🧭 v2 Architecture (AWS design)

The next version is designed as a production-style AWS deployment with a private network, a VPN-only sandbox, and every request recorded. It is a **design**; the running version is v1, described in the next section.
Full design spec: [`docs/superpowers/specs/2026-11-03-rag-v2-design.md`](docs/superpowers/specs/2026-11-03-rag-v2-design.md)

**Request path**: nginx edge in a public subnet, RAG stack in a private subnet:

![v2 request path](docs/architecture/aws-architecture.png)

**Team access**: AWS Client VPN → Warpgate bastion → prod and sandbox, with CloudWatch and VPC Flow Logs:

![v2 team access and sandbox](docs/architecture/team-access.png)

**Highlights**
- **Hybrid retrieval:** BM25 sparse + dense vectors in **Qdrant**, fused with Reciprocal Rank Fusion (k = 60).
- **Self-hosted embeddings:** **EmbeddingGemma 2** (text-only, CPU), so queries never leave the VPC and there is no per-token embedding bill.
- **Contextual retrieval at ingestion:** Amazon Bedrock **Nova Micro** writes a short context for each chunk.
  - The **whole document is a cached prompt prefix**, so later chunks read it from cache instead of re-running prefill.
  - That makes cached input 75% cheaper.
- **Incremental indexing:**
  - content-hash IDs, so unchanged re-runs write nothing;
  - retries with backoff, resumable runs, a delete guard and a per-collection run lock.
- **Network:**
  - **Public subnet:** nginx + certbot edge, plus a NAT instance.
  - **Private subnets:** prod and a **sandbox** that replicates it.
  - **NAT Gateway:** switched on only to pull images and the model.
- **Team access:**
  - **AWS Client VPN** with one certificate per person;
  - **Warpgate** bastion with per-user logins, recording every SSH session and sandbox HTTP request;
  - SSH only through VPN → Warpgate.
- **Observability:** CloudWatch logs, metrics, alarms and a dashboard, plus VPC Flow Logs for every network flow.
- **Security:**
  - IMDSv2;
  - secrets in SSM Parameter Store;
  - least-privilege IAM, with each host writing only its own logs and Bedrock limited to two models.
- **Cost-aware:** instances are stopped when idle, the VPN is detached outside work hours, and budget alerts are set.

Editable diagrams: [`docs/architecture/aws-architecture.drawio`](docs/architecture/aws-architecture.drawio) (two pages).

---

## 🏗 System Architecture (v1, current)

```mermaid
graph TD
    subgraph Client_Layer [Client Layer]
        User((User))
        Browser[Web Browser]
    end

    subgraph App_Layer [Application Layer - AWS App Runner]
        Django[Django Web Server]
        Auth[Auth System]
        ChatUI[Chat Interface]
        subgraph Logic [RAG Logic - LlamaIndex]
            Agent[Multi-Agent Orchestrator]
            Retriever[Amazon KB Retriever]
        end
    end

    subgraph AI_Data_Layer [AI & Data Layer]
        GH_Models[GitHub Models - GPT-4o]
        Bedrock[Amazon Bedrock Knowledge Base]
        Supabase[(Supabase - PostgreSQL)]
    end

    User <--> Browser
    Browser <--> ChatUI
    ChatUI <--> Django
    Django <--> Auth
    Django <--> Agent
    Agent <--> GH_Models
    Agent <--> Retriever
    Retriever <--> Bedrock
    Bedrock <--> Supabase
```

## 🤖 Agentic Workflow

This diagram represents the high-level orchestration of our AI agents within the assistant:

```mermaid
graph TD
    A[User Query] --> B[Planner Agent]
    B --> C{Retriever Agent <br/> Needed?}
    C -- Yes --> D[Retriever Agent]
    D --> E[Generator Agent]
    C -- No --> E
    E --> F[Response]
```

### Agency Components:
- **Planner Agent**: Analyzes the intent of the user query to decide if it requires factual retrieval from university documents (`RETRIEVE`), general conversation (`DIRECT`), or should be rejected as out-of-context (`REJECT`).
- **Retriever Agent**: Interfaces with the **Amazon Bedrock Knowledge Base** to fetch high-relevance chunks of documentation.
- **Generator Agent**: Synthesizes the final response by combining the user's intent, the retrieved context, and the conversation history into a clear, professional answer.
- **Orchestrator**: Manages the state and transition between agents, ensuring robust fallback logic if any single component fails.

## 🔄 RAG Workflow

This diagram illustrates the decision logic behind every response, including the robust fallback mechanism.

```mermaid
flowchart TD
    A[User Message Received] --> B{Initialize Orchestrator?}
    B -- Success --> C[Query Multi-Agent System]
    B -- Failure --> D[Initialize Raw Retriever]
    
    C --> E{LLM Responded?}
    E -- Yes --> F[Display AI Response]
    E -- No --> D
    
    D --> G{Data Found?}
    G -- Yes --> H[Display Raw Snippets <br/> with Warning]
    G -- No --> I[Display Friendly Error]
    
    F --> J[Save to Database]
    H --> J
    I --> J
```

## 📡 Sequence Diagram

```mermaid
sequenceDiagram
    participant U as User
    participant D as Django (View)
    participant O as Orchestrator (Multi-Agent)
    participant K as Bedrock KB
    participant L as LLM (GPT-4o)

    U->>D: Send Message
    D->>O: Process Request
    O->>K: Retrieve Context
    K-->>O: Context Chunks
    O->>L: Context + Query
    L-->>O: Structured Answer
    O-->>D: response_text
    D-->>U: Display Answer
```

---

## 🛠️ Tech Stack

### Frontend
- **HTML5/CSS3**: Modern, mobile-first design system.
- **JavaScript (ES6+)**: Custom chat logic and UI interactions.
- **Marked.js**: High-performance Markdown rendering for bot responses.

### Backend
- **Django**: Robust Python web framework.
- **Gunicorn**: Production WSGI HTTP server.
- **WhiteNoise**: Efficient static file serving for containerized environments.

### AI & Data
- **RAG Engine**: LlamaIndex (Multi-Agent Orchestrator).
- **LLMs**: Claude 3 (AWS Bedrock), GPT-4o (GitHub Models).
- **Vector Database**: Amazon Bedrock Knowledge Bases.
- **Database**: 
  - **Local**: SQLite
  - **Production**: Supabase (PostgreSQL)

---

## 🚀 Local Setup

### 1. Clone the Repository
```bash
git clone https://github.com/MAVIcVICKY/LPU_RAG.git
cd LPU_RAG
```

### 2. Configure Environment Variables
Create a `.env` file in the root directory and add the following (see `.env.example` for reference):
```env
DJANGO_SECRET_KEY=your-secret-key
DJANGO_DEBUG=True
DB_NAME=your-db-name
DB_USER=your-db-user
DB_PASSWORD=your-db-password
DB_HOST=your-db-host
DB_PORT=5432

# AWS Credentials
AWS_ACCESS_KEY_ID=your-access-key
AWS_SECRET_ACCESS_KEY=your-secret-key
AWS_REGION=ap-south-1

# Knowledge Base
AWS_KB_ID=your-knowledge-base-id
```

### 3. Install Dependencies
```bash
pip install -r requirements.txt
```

### 4. Run Migrations & Start Server
```bash
python manage.py migrate
python manage.py runserver
```

---

## 🐳 Deployment (Docker & AWS)

> [!IMPORTANT]
> All commands below must be executed from the `LPU_RAG` directory (where the `Dockerfile` is located).

### Push to Amazon ECR
Retrieve an authentication token and authenticate your Docker client to your registry:
```powershell
(Get-ECRLoginCommand).Password | docker login --username AWS --password-stdin 804594767592.dkr.ecr.ap-south-1.amazonaws.com
```

Build, tag, and push your image:
```bash
docker build -t rag-bot .
docker tag rag-bot:latest 804594767592.dkr.ecr.ap-south-1.amazonaws.com/rag-bot:latest
docker push 804594767592.dkr.ecr.ap-south-1.amazonaws.com/rag-bot:latest
```

---

## 🎨 Design Philosophy
The UI follows a **clean, modern aesthetic** inspired by premium SaaS applications:
- **Responsive Layout**: Designed to look like a native application on mobile while providing a full-featured dashboard on desktop.
- **Visual Feedback**: Subtle hover states, smooth scroll behaviors, and slide-up animations for messages.
- **Accessibility**: Clear contrast ratios and legible typography using the **Roboto** font family.

---

## 📄 License
© 2026 University Management System AI Portal. All Rights Reserved.
