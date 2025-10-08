
-----

1.  Create a new, empty folder for this setup. For example: `~/docker-gemini/`

2.  Inside that folder, create a file named `Dockerfile` (no extension) and paste the following content into it:

    ```dockerfile
    # Use the latest official Ubuntu image as the base
FROM ubuntu:latest

# Set environment variables to prevent interactive prompts during installation
ENV DEBIAN_FRONTEND=noninteractive

# Update package lists and install necessary dependencies for gcloud and nodejs
RUN apt-get update && apt-get install -y \
    curl \
    gnupg \
    apt-transport-https \
    ca-certificates \
    lsb-release \
    && rm -rf /var/lib/apt/lists/*

# --- Install Google Cloud SDK ---
RUN curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key --keyring /usr/share/keyrings/cloud.google.gpg add -
RUN echo "deb [signed-by=/usr/share/keyrings/cloud.google.gpg] https://packages.cloud.google.com/apt cloud-sdk main" | tee -a /etc/apt/sources.list.d/google-cloud-sdk.list
RUN apt-get update && apt-get install -y google-cloud-sdk

# --- NEW: Install Gemini CLI component ---
# The -q flag is for "quiet" to prevent interactive prompts during the build
RUN gcloud components install -q beta

# --- NEW: Install NodeJS (LTS version from NodeSource) ---
RUN curl -fsSL https://deb.nodesource.com/setup_22.x | bash -
RUN apt-get install -y nodejs
RUN npm install -g @google/gemini-cli

# Set the working directory inside the container
WORKDIR /workspace

# Set a default command to keep the container running
CMD ["tail", "-f", "/dev/null"]

    ```

-----

### \#\# 2. Build the Docker Image 🔨

Now, navigate to the folder containing your `Dockerfile` in your terminal and run the following command to build the image. We'll tag it as `gemini-ubuntu`.

```bash
docker build -t gemini-ubuntu .
```

This command reads the `Dockerfile` in the current directory (`.`) and creates a local Docker image named `gemini-ubuntu`. This might take a few minutes.

-----

### \#\# 3. Run the Container with a Volume Mount 🐳

This is the key step where you connect your host folder to the container. Run this command to start your container in the background:

**Important:** Make sure your host directory exists first.
`mkdir -p /home/halis/develop/gemini-workspace`

Now, run the container:

```bash
docker run -it -d --name gemini-cli -v /home/halis/develop/gemini-workspace:/workspace gemini-ubuntu
```

Let's break down that command:

  * `docker run`: The command to start a new container.
  * `-it`: Keeps the container's input/output interactive.
  * `-d`: Detached mode. It runs the container in the background.
  * `--name gemini-cli`: Gives your container a memorable and permanent name so you can easily start, stop, and connect to it.
  * `-v /home/halis/develop/gemini-workspace:/workspace`: This is the **volume mount**. It maps your host folder (`/home/halis/develop/gemini-workspace`) to the `/workspace` folder inside the container. **Any file you create or modify in one location will instantly appear in the other.**
  * `gemini-ubuntu`: The name of the image you built in the previous step.

-----

### \#\# 4. Authenticate and Install Gemini CLI 💎

Your container is now running. Let's get inside it to perform the one-time setup.

1.  **Enter the container's shell:**

    ```bash
    docker exec -it gemini-cli /bin/bash
    ```

    Your terminal prompt will change, indicating you are now the `root` user *inside the container*.

2.  **Authenticate with Google Cloud:**
    Run the authentication command. The `--no-launch-browser` flag is crucial because the container doesn't have a web browser.

    ```bash
    gcloud auth application-default login --no-launch-browser
    ```

      * This will print a long URL to your console.
      * **Copy** this URL.
      * **Paste** it into the web browser on your **host machine**.
      * Log in with your Google account and grant the necessary permissions.
      * After you approve, your browser will show an authorization code. **Copy** this code.
      * **Paste** the code back into your container's terminal and press Enter.

3.  **Install the Gemini CLI component:**
    The `gemini` command is part of the `beta` components of the `gcloud` CLI. Install it:

    ```bash
    gcloud components install beta
    ```

    You can also run `gcloud components update` to ensure everything is up to date.

You are now fully set up\! The authentication credentials are saved inside the container's filesystem and will persist as long as you don't delete the container.

-----

### \#\# 5. Use Gemini CLI on Your Host Files 🚀

Let's test it.

1.  **On your HOST machine**, create a test file in your shared folder:

    ```bash
    echo "Explain what a Docker volume mount is in simple terms." > /home/halis/develop/gemini-workspace/my-prompt.txt
    ```

2.  **Now, inside the CONTAINER's terminal** (the one you opened with `docker exec`), list the files to confirm you can see it:

    ```bash
    ls -l
    ```

    You should see `my-prompt.txt`.

3.  **Run the Gemini CLI** using the content of that file:

    ```bash
    gemini chat send "$(cat my-prompt.txt)"
    ```

You will see Gemini's response printed directly to your terminal\! Any files Gemini helps you create or modify inside the `/workspace` directory will be instantly available on your host machine at `/home/halis/develop/gemini-workspace`.

-----

### \#\# Your Daily Workflow

You've done the hard part. From now on, your daily process is simple:

  * **To start your workspace:**

    ```bash
    docker start gemini-cli
    ```

  * **To enter your workspace:**

    ```bash
    docker exec -it gemini-cli /bin/bash
    ```

  * **When you're done for the day, to stop the container:**

    ```bash
    docker stop gemini-cli
    ```

Your setup, authentication, and installed components are all saved. You never need to run the `docker run` command or authenticate again unless you explicitly delete the container with `docker rm gemini-cli`.