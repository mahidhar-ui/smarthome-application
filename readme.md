SmartHome IoT Assistant

A Retrieval-Augmented Generation (RAG) chatbot that answers questions about a SmartHome IoT platform's
documentation. Ask it about system architecture, device management, automation rules, security,
troubleshooting, or API design — it searches the actual documents and generates a grounded answer.

Built with: FastAPI, Streamlit, ChromaDB, scikit-learn (TF-IDF), Groq (LLaMA 3.3 70B), Docker.


What is in this repo

smart-home-application/
├── streamlit_app_tfidf.py   # Option A: Streamlit chat UI, TF-IDF search (used by default in Docker)
├── streamlit_app.py         # Option A-alt: Streamlit chat UI, ChromaDB semantic search
├── main.py                  # Option B: FastAPI backend API
├── Dockerfile                # Container definition (runs streamlit_app_tfidf.py by default)
├── requirements.txt          # Pinned Python dependencies
├── .env                       # Local secrets — never committed (gitignored)
├── .gitignore
└── _data/                     # Platform documentation the RAG system searches
    ├── 01_smart_home_architecture_deep_dive.txt
    ├── 02_smart_home_device_management_deep_dive.txt
    ├── 03_smart_home_automation_deep_dive.txt
    ├── 04_smart_home_security_deep_dive.txt
    ├── 05_smart_home_troubleshooting_deep_dive.txt
    └── 06_smart_home_api_data_deep_dive.txt


How it works

User types a question
        |
        v
TF-IDF / ChromaDB searches _data/ documents
(finds top 3 most relevant paragraphs)
        |
        v
Those paragraphs are sent to Groq LLM as context
        |
        v
LLM generates a grounded answer
        |
        v
Answer + source documents + raw chunks (with similarity scores) displayed to user

Documents are chunked by paragraph at startup and indexed in memory (either a TF-IDF matrix or a ChromaDB
collection, depending on which app file you run). No external database needed — everything resets on each
restart.


Prerequisites

- Python 3.11+
- A Groq API key — free at https://console.groq.com
- Git


Local Setup

Step 1: Clone the repo

    git clone https://github.com/YOUR_USERNAME/smart-home-application.git
    cd smart-home-application

Step 2: Create and activate a virtual environment

    python -m venv .venv
    source .venv/bin/activate           # Mac / Linux
    # .venv\Scripts\activate            # Windows

Step 3: Install dependencies

    pip install -r requirements.txt

Step 4: Add your Groq API key

Create a .env file in the project root:

groq_api_key=gsk_xxxxxxxxxxxxxxxxxxxxxxxx

Get your key at https://console.groq.com — it is free to sign up.


Running Locally

Option A: Streamlit UI — TF-IDF (recommended, lightweight)

No model download, starts in under 2 seconds, uses keyword-based search.

    streamlit run streamlit_app_tfidf.py

Open: http://localhost:8501

Option A-alt: Streamlit UI — ChromaDB (semantic search)

Understands synonyms and meaning, not just exact word matches. Needs ~300MB RAM.

    streamlit run streamlit_app.py

Open: http://localhost:8501

You will see:
- A green confirmation that documents were indexed
- A chat input at the bottom
- Answers with source filenames and a collapsible "View retrieved chunks" section
  (TF-IDF version also shows a similarity score per chunk)

Option B: FastAPI backend only

Runs a pure JSON API with no visual interface. Use this to understand how a backend API works, or if you want to
build your own frontend later.

    uvicorn main:app --reload --port 8000

Open: http://localhost:8000/docs

The /docs page is FastAPI's built-in interactive UI (Swagger). To test:
1. Click POST /chat
2. Click Try it out
3. Change the question in the request body
4. Click Execute
5. See the JSON response below

To call it from Python:

    import requests

    response = requests.post(
        "http://localhost:8000/chat",
        json={"question": "What are the core resources in the SmartHome API?"}
    )
    print(response.json())


Deploying to Render (Free)

Render is a cloud platform that runs Docker containers for free (with some limits). The Dockerfile in this repo is
already configured for Render.

Step 1: Push your code to GitHub

    # Inside the project folder, with venv active:

    git init
    git branch -m main
    git add .
    git commit -m "Initial deploy: SmartHome IoT Assistant"

    # Set remote with your GitHub PAT for authentication
    # Replace YOUR_USERNAME and YOUR_PAT with your actual values
    git remote add origin https://YOUR_USERNAME:YOUR_PAT@github.com/YOUR_USERNAME/smart-home-application.git

    git push -u origin main

Getting a GitHub PAT (Personal Access Token):
1. Go to https://github.com/settings/tokens/new
2. Note: smart-home-deploy
3. Expiration: 90 days
4. Scopes: tick repo
5. Click Generate token — copy it immediately

Step 2: Create a Render account

Go to https://render.com and sign up (free — no credit card needed for basic use).

Step 3: Create a new Web Service
1. Click New → Web Service
2. Connect your GitHub account when prompted
3. Select the smart-home-application repository
4. Fill in the settings:

Setting          Value
Name             smart-home-application
Region           Singapore (or closest to you)
Branch           main
Runtime          Docker
Instance Type    Free

5. Click Create Web Service

Step 4: Add your Groq API key as a secret

Never put API keys in code or in the repo. Render injects them as environment variables.
1. On your service page, click Environment
2. Click Add Environment Variable
3. Key: groq_api_key
4. Value: your actual Groq API key (gsk_...)
5. Click Save Changes

Render will automatically redeploy with the secret available.

Step 5: Wait for the build (~3-5 minutes)

Render will:
- Pull your code from GitHub
- Build the Docker image (installs all packages from requirements.txt)
- Start the container running streamlit run streamlit_app_tfidf.py

Watch the build logs on the Render dashboard. You will see:

Ready — N document chunks indexed.
You can now view your Streamlit app in your browser.

Step 6: Access your live app

Once deployed, your app is live at:

https://smart-home-application.onrender.com

Share this URL — anyone can open it and chat with the SmartHome assistant.


Updating the app

Every time you push to GitHub, Render automatically rebuilds and redeploys:

    # Make your changes, then:
    git add .
    git commit -m "Update: describe what changed"
    git push


Troubleshooting

Problem                                  Likely cause                              Fix
Error loading ASGI app                   Running uvicorn from the wrong folder     cd into the project folder first
_data/ folder not found                  Running from wrong directory              Make sure _data/ is next to the script
groq_api_key not found                   .env file missing or wrong key name       Check .env has groq_api_key=gsk_...
Push rejected (auth failed)              PAT expired or wrong scope                Create a new classic PAT with repo scope
Render build fails                       Check Render logs tab                     Usually a missing package or wrong port
App loads but answers are wrong          LLM answering from memory, not docs       Check _data/ files were committed to GitHub
Cold start on Render (30s delay)         Free tier spins down after 15min idle     Normal behaviour — first request is slow
ResolutionImpossible during pip install  Version pins conflicting (e.g. pydantic vs chromadb)  Loosen the strict pin, pin chromadb to a modern release


Key concepts covered

Concept                              Where to see it
RAG pipeline                         load_rag() in streamlit_app_tfidf.py / streamlit_app.py
TF-IDF + cosine similarity search    vectorizer.transform() + cosine_similarity() in ask_rag()
ChromaDB vector search               collection.query() in ask_rag() (ChromaDB version)
Groq API call                        groq_client.chat.completions.create()
Streamlit session state              st.session_state.messages
Streamlit caching                    @st.cache_resource on load_rag()
FastAPI request/response schema      ChatRequest / ChatResponse in main.py
Docker containerisation              Dockerfile
Environment secrets                  .env locally, Render dashboard in production


Notes on the free Render tier

- The app spins down after 15 minutes of inactivity — the first request after idle takes ~30 seconds to wake up
- Free tier has 512MB RAM — the TF-IDF version fits comfortably; the ChromaDB version may exceed this and crash
- To keep it always awake, upgrade to Render's paid tier or use an uptime monitor like
  https://uptimerobot.com to ping it every 10 minutes

Live demo: 