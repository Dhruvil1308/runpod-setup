# GuniVox V3 RunPod Deployment Guide

Based on the architecture of **GuniVox V3** and the VS Code SSH workflow you referenced, here is a detailed analysis and recommendation for deploying your system on RunPod.

---

## 1. System Analysis: Do you actually need a high-end GPU?

Let's look at what your system is actually running based on your `.env.local` and `server.py`:
*   **STT & TTS (Sarvam AI):** Handled via Cloud API. Uses 0 local compute.
*   **LLM Inference (OpenAI `gpt-4o-mini`):** Handled via Cloud API. Uses 0 local compute. *(Note: While Ollama code exists as a fallback, your system is explicitly locked to OpenAI for lowest latency).*
*   **Vector Search & Embeddings (FAISS + SentenceTransformers `all-MiniLM-L6-v2`):** This is the **only** local AI model. However, `all-MiniLM-L6-v2` is extremely small and lightweight. It runs perfectly fine on a standard CPU and takes almost no memory.
*   **Backend Server (FastAPI + SQLite):** Lightweight, runs on CPU.

**Conclusion:** Because you are using OpenAI and Sarvam AI APIs, **you do NOT need an expensive GPU like an RTX 4090**. Your system is almost entirely CPU-bound. 

---

## 2. Hardware Recommendations (Optimized for Cost)

Since your heavy lifting is done by external APIs, you should optimize for a stable network connection and standard compute rather than expensive VRAM. 

### Recommended Instance type
*   **Top Pick:** **RunPod CPU Instance** (or a standard VPS like DigitalOcean / AWS EC2)
    *   **Why:** You only need enough compute to run a FastAPI server and FAISS. A CPU pod will cost you pennies compared to a GPU pod.
*   **Alternative (If you strictly want a GPU Pod):** **NVIDIA RTX 3070 / RTX 4000 Ada / RTX A4000**
    *   **Why:** If you want to use a GPU instance just to be safe, pick the absolute cheapest one available (usually around $0.20/hr). The GPU will ensure the SentenceTransformer embeddings are generated in milliseconds, but anything more powerful is a waste of money.

### Recommended RAM & CPU
*   **System RAM:** **8 GB to 16 GB** 
    *   **Why:** 8GB is plenty for FastAPI, SQLite, and the FAISS vector index in RAM.
*   **vCPUs:** **4 to 8 vCPUs**
    *   **Why:** Sufficient for handling concurrent incoming HTTP requests from Vobiz webhook callbacks.

### Recommended Disk Space
*   **Container Disk:** **20 GB**
*   **Volume Disk (Persistent):** **20 GB**
    *   **Why:** You don't have massive local Ollama models. You only need space for the OS, Python libraries, and your SQLite database.

---

## 3. RunPod Template Selection

*   **Template:** **RunPod PyTorch 2.1.0** (or `runpod/pytorch:latest`)
*   **Why this template?**
    *   Even though you don't need heavy GPU compute, this template gives you full root SSH access and pre-installed Python, which makes it incredibly easy to connect via VS Code (as shown in your video).

**Port Configuration:**
When creating the pod, make sure to expose the ports you need:
*   **8000** (for your FastAPI HTTP Server)
*   *Note: If you use `ngrok` inside the pod, exposing ports is optional since ngrok creates a secure tunnel automatically.*

---

## 4. Step-by-Step Deployment Workflow

Based on the YouTube video, here is how you connect and deploy:

### Step A: Connect via VS Code
1. Start your RunPod instance (CPU or cheap GPU).
2. In RunPod, click **Connect** on your active pod, and copy the **SSH command** (e.g., `ssh root@x.x.x.x -p 12345 -i ~/.ssh/id_ed25519`).
3. Open VS Code locally, use the **Remote - SSH** extension, and paste the command.
4. Once connected, open a terminal in VS Code. You are now inside the RunPod instance!

### Step B: Setup the Environment
1. Clone your repository (or drag-and-drop the `GuniVox V3` folder into the VS Code explorer).
2. Navigate to your project folder:
   ```bash
   cd /workspace/GuniVox V3
   ```
3. Install Python dependencies:
   ```bash
   pip install -r requirements.txt
   ```

### Step C: Configure Environment Variables
Ensure your `.env.local` file is present in your workspace with your API keys:
```env
AI_PROVIDER=openai
OPENAI_MODEL=gpt-4o-mini
OPENAI_API_KEY=sk-proj-...
SARVAM_API_KEY=sk_...
VOBIZ_AUTH_ID=...
VOBIZ_AUTH_TOKEN=...
VOBIZ_FROM_NUMBER=...
```

### Step D: Run the Server
You can now start your FastAPI server. If you use ngrok to expose it to Vobiz:
```bash
# Start your server
uvicorn server:app --host 0.0.0.0 --port 8000
```
Then, in another terminal, run ngrok:
```bash
ngrok http 8000
```
Update your `BASE_URL` in `.env.local` (and restart your server) with the new ngrok URL so Vobiz webhooks route correctly to your RunPod instance.

## Summary
Because you have cleverly outsourced the heavy lifting to the **OpenAI** and **Sarvam AI APIs**, your RunPod deployment is incredibly lightweight. You do not need an expensive RTX 4090 or A6000. Save your money and use a **RunPod CPU instance** (or the absolute cheapest GPU available), and follow the standard SSH workflow to run your FastAPI app!
