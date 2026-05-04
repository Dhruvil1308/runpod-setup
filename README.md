# GuniVox V3 RunPod Deployment Guide

Based on the architecture of **GuniVox V3** and the VS Code SSH workflow you referenced, here is a detailed analysis and recommendation for deploying your system on a RunPod GPU instance for ultra-low latency voice RAG.

---

## 1. System Analysis: What Requires Compute?

GuniVox V3 has a hybrid architecture:
*   **STT & TTS (Sarvam AI):** Cloud API-based. These **do not** require local GPU resources.
*   **LLM Inference (Ollama - `jatas-qwen-rag:latest`):** This is the **heaviest component**. For a telephony agent, latency (Time-To-First-Token) must be incredibly low (under 500ms). Running this locally in Ollama demands a fast GPU.
*   **Vector Search & Embeddings (FAISS + SentenceTransformers `all-MiniLM-L6-v2`):** `all-MiniLM-L6-v2` is very small and fast. It can run on the CPU or take <1GB of VRAM on the GPU. FAISS operates entirely in RAM.
*   **Backend Server (FastAPI + SQLite):** Lightweight, runs on CPU.

Because your STT and TTS are handled via API, your RunPod GPU is dedicated *entirely* to LLM generation (Ollama) and text embeddings, which is perfect for maximizing speed.

---

## 2. Hardware Recommendations

To achieve the "very low latency" required for a voice agent, you need a GPU with high memory bandwidth to stream tokens as fast as possible.

### Recommended GPU (The Sweet Spot)
*   **Top Pick:** **NVIDIA RTX 4090 (24GB VRAM)**
    *   **Why:** The RTX 4090 offers incredible memory bandwidth and is currently one of the fastest GPUs for single-batch LLM inference (perfect for a voice agent). 
    *   **Capacity:** 24GB VRAM is more than enough to comfortably hold a 7B, 8B, or even 14B parameter Qwen model (quantized to 4-bit or 8-bit), plus the embedding model.
*   **Alternative:** **RTX 3090 (24GB VRAM)** or **RTX A5000 (24GB VRAM)**
    *   **Why:** Slightly cheaper per hour on RunPod, but still provides 24GB of VRAM and excellent inference speed.

### Recommended RAM & CPU
*   **System RAM:** **32 GB** 
    *   **Why:** When Ollama loads a model, it first loads it into system RAM before passing it to the GPU. FAISS also stores the entire vector database in RAM. 32 GB prevents any Out-Of-Memory (OOM) crashes.
*   **vCPUs:** **8 vCPUs**
    *   **Why:** Sufficient for FastAPI concurrent requests, SQLite DB operations, and background threads.

### Recommended Disk Space
*   **Container Disk:** **50 GB**
*   **Volume Disk (Persistent):** **100 GB**
    *   **Why:** Docker images, Python environments, and Ollama model weights (which can be 5GB–15GB each) eat up disk space quickly. A persistent volume ensures you don't have to re-download the model every time you restart the pod.

---

## 3. RunPod Template Selection

You need a template that allows you to run Python, FastAPI, and Ollama simultaneously, while also supporting the VS Code SSH connection you watched in the video.

*   **Template:** **RunPod PyTorch 2.1.0** (or latest stable PyTorch template, e.g., `runpod/pytorch:2.x.x-py3.10-cuda12.x.x-devel-ubuntu22.04`)
*   **Why this template?**
    *   It comes pre-installed with CUDA drivers, Python, and Jupyter.
    *   It gives you full root SSH access.
    *   It is a blank slate perfect for installing Ollama via terminal and running your `start_multi.sh` or `server.py` directly.

**Port Configuration:**
When creating the pod, make sure to expose the ports you need:
*   **8000** (for your FastAPI HTTP Server)
*   **11434** (for Ollama, though you can keep this local to the pod)
*   *Note: If you use `ngrok` inside the pod (as seen in your code), exposing ports is optional since ngrok creates a secure tunnel automatically.*

---

## 4. Step-by-Step Deployment Workflow

Based on your YouTube video, here is how you connect and deploy:

### Step A: Connect via VS Code
1. Start your RunPod instance with the hardware specified above.
2. In RunPod, click **Connect** on your active pod, and copy the **SSH command** (e.g., `ssh root@x.x.x.x -p 12345 -i ~/.ssh/id_ed25519`).
3. Open VS Code locally, use the **Remote - SSH** extension, and paste the command.
4. Once connected, open a terminal in VS Code. You are now inside the RunPod GPU!

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
   *(Ensure `sentence-transformers`, `faiss-cpu`, `fastapi`, `uvicorn`, `requests`, `python-dotenv` are in your requirements).*

### Step C: Install and Start Ollama
Since you are using Ollama as your AI Provider:
1. Install Ollama via the official script:
   ```bash
   curl -fsSL https://ollama.com/install.sh | sh
   ```
2. Start the Ollama server in the background (or use `tmux`/`screen`):
   ```bash
   ollama serve &
   ```
3. Pull your required model:
   ```bash
   ollama pull jatas-qwen-rag:latest
   ```
   *(If this is a custom model, ensure you build it or pull the base model like `qwen:7b`).*

### Step D: Configure Environment Variables
Create or edit the `.env.local` file in your workspace:
```env
AI_PROVIDER=ollama
OLLAMA_BASE_URL=http://localhost:11434
OLLAMA_MODEL=jatas-qwen-rag:latest
ENABLE_RAG=true
SARVAM_API_KEY=your_sarvam_key
VOBIZ_AUTH_ID=your_vobiz_id
VOBIZ_AUTH_TOKEN=your_vobiz_token
VOBIZ_FROM_NUMBER=your_vobiz_number
```

### Step E: Run the Server
You can now start your FastAPI server. If you use ngrok to expose it to Vobiz:
```bash
# Start your server (adjust the port/host as needed)
uvicorn server:app --host 0.0.0.0 --port 8000
```
Then, in another terminal, run ngrok:
```bash
ngrok http 8000
```
Update your `BASE_URL` in `.env.local` with the new ngrok URL so Vobiz webhooks route correctly to your RunPod instance.

## Summary
By using an **RTX 4090** on RunPod with the **PyTorch template**, you are keeping your heaviest component (the LLM) on top-tier hardware. Because Sarvam AI handles the audio generation in the cloud, the 4090 will respond with text tokens almost instantly, giving your users a seamless, low-latency conversational experience.
