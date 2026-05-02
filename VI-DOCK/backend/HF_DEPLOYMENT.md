# Hugging Face Backend Deployment Guide for VI DOCK Pro 3.1

This guide walks you through deploying the VI DOCK backend on **Hugging Face Spaces** using Docker. Hugging Face provides a free, persistent tier for hosting ML and backend applications.

## Prerequisites
1.  A Hugging Face account ([Sign up here](https://huggingface.co/join)).
2.  Your code uploaded to GitHub (e.g., `https://github.com/messiay/simdock-pro.git`).

---

## 1. Create a New Hugging Face Space
1. Go to [Hugging Face Spaces](https://huggingface.co/spaces) and click **Create new Space**.
2. **Space Name**: Choose a name for your backend (e.g., `simdock-backend`).
3. **License**: Choose your preferred license (e.g., `MIT`).
4. **Select the Space SDK**: Select **Docker**.
5. **Docker Template**: Choose **Blank** (we will use our existing Dockerfile).
6. **Space Hardware**: **Free (CPU basic - 2vCPU - 16GB)** is sufficient for basic docking, though a Pro account is recommended if you expect heavy workloads.
7. Click **Create Space**.

---

## 2. Link Your Codebase
Once your space is created, you need to provide the code to run it. The easiest way is to clone your GitHub repository into the space.

### Option A: Using the Hugging Face Web UI (Recommended)
1. Go to the **Files** tab of your new Space.
2. If there's an existing `README.md`, edit it or leave it as is.
3. Click **Add file > Create new file** and create a file named `Dockerfile` in the root.
4. Copy the contents of the `backend/Dockerfile` from your project into this new file:
   ```dockerfile
   FROM python:3.9-slim

   # Install system dependencies
   RUN apt-get update && apt-get install -y \
       wget \
       build-essential \
       libpcre3 \
       libpcre3-dev \
       openbabel \
       git \
       && rm -rf /var/lib/apt/lists/*

   # Set up work directory
   WORKDIR /app

   # Clone the specific backend folder directly (for a complete deployment)
   # Or copy files if using the HF CLI. Assuming we are copying from the repo:
   RUN git clone https://github.com/messiay/simdock-pro.git /repo && \
       cp -r /repo/VI-DOCK/backend/* /app/ && \
       rm -rf /repo

   # Install Python requirements
   RUN pip install --no-cache-dir -r requirements.txt

   # Download Linux versions of Vina and Smina
   RUN mkdir -p /app/bin && \
       wget https://github.com/ccsb-scripps/AutoDock-Vina/releases/download/v1.2.5/vina_1.2.5_linux_x86_64 -O /app/bin/vina && \
       wget https://github.com/gnina/smina/releases/download/v2020.12.10/smina.static -O /app/bin/smina && \
       chmod +x /app/bin/vina /app/bin/smina

   # Set up user for Hugging Face (Running as user ID 1000)
   RUN useradd -m -u 1000 user && \
       mkdir -p "/app/VI DOCK_Projects" && \
       chown -R user:user /app

   USER user
   ENV HOME=/home/user \
       PATH=/home/user/.local/bin:$PATH \
       PYTHONPATH=$PYTHONPATH:/app

   # Expose port
   EXPOSE 7860

   # Start the API
   CMD ["uvicorn", "api.main:app", "--host", "0.0.0.0", "--port", "7860"]
   ```
5. Click **Commit new file to `main`**.
6. **Wait for the Build**: Go to the **App** tab. You'll see "Building". This might take a few minutes as it downloads OpenBabel and Vina.

---

## 3. Connect the Frontend
1. Once the Space status says **Running**, look for the three dots (`...`) in the top right corner of the App view and click **Embed this Space**.
2. Look for the **Direct URL** (it usually looks like `https://username-spacename.hf.space`).
3. Copy this URL.
4. **Update your Frontend Environment**:
   * **Local Development**: Open your `.env` file or `VI-DOCK/src/config.ts` and set `VITE_API_BASE_URL` to your Hugging Face URL.
   * **Vercel/Netlify Deployment**: Go to your project settings, find "Environment Variables", and update `VITE_API_BASE_URL` to the Hugging Face URL.

## ⚠️ Important Notes
* **Storage**: By default, Hugging Face Spaces are ephemeral. If the space restarts, any uploaded projects in `VI DOCK_Projects` will be lost. For permanent storage, you need to upgrade to a persistent storage tier in the Hugging Face Space settings.
* **Cold Starts**: If your Space receives no traffic for 48 hours, it may go to "sleep". It will automatically wake up when you access the URL, but the first request might take a minute.
