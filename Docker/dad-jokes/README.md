# Dad Joke Dashboard

A lightweight, containerized dashboard that automatically fetches and displays a new "Dad Joke" every 30 seconds.

This project demonstrates the **Sidecar Pattern** in Docker, where a "Fetcher" service generates content and a "Web Server" service serves it, communicating via a shared volume. Both services run as **non-privileged users** for enhanced security.

## 🚀 Features

* **Auto-Refresh:** The HTML page automatically reloads every 30 seconds.
* **Separation of Concerns:** Nginx handles serving; a custom script handles logic.
* **Security:** Runs entirely as a non-root user (UID 101) to mitigate privilege escalation risks.
* **Lightweight:** Built on Alpine Linux.

## 📋 Prerequisites

* [Docker Desktop](https://www.docker.com/products/docker-desktop) or Docker Engine
* Docker Compose

## 📂 Project Structure

Ensure your files are organized exactly as shown below:

```text
dad-jokes/
├── README.md
├── docker-compose.yml
└── fetcher/
    ├── Dockerfile
    └── updater        
```
## 🛠️ Quick Start
Navigate to the project directory:

```Bash
cd dad-jokes
```
## Build and Start the containers:

```Bash

docker-compose up --build -d
```

## View the Dashboard: Open your web browser and go to: http://localhost:8080

Stop the containers:
```Bash
docker-compose down
```

## ⚙️ Configuration
Changing the Update Interval
To change how often the joke updates:

Open fetcher/updater.

Change sleep 30 to your desired seconds (e.g., sleep 60).

Update the HTML meta tag: <meta http-equiv="refresh" content="60">.

Rebuild the container: docker-compose up --build -d.

Troubleshooting
Problem: "exec user process caused: no such file or directory"

Cause: This usually happens if you created the updater script on Windows. Windows uses CRLF for line breaks, but Linux requires LF.

Fix: Open the file in your editor (VS Code, Notepad++, etc.) and change the End of Line sequence from CRLF to LF, then save.

Problem: Permission Denied

Cause: The container cannot write to the volume.

Fix: Ensure the Dockerfile includes the chown 101:101 /data step (as included in the source code).

## 🛡️ Security Details
This stack adheres to the principle of least privilege:

Nginx: Uses the nginxinc/nginx-unprivileged:alpine image, listening on port 8080 (internally).

Fetcher: The Dockerfile explicitly creates a user appuser with UID 101 and switches to it using USER 101.

Rootless: No process inside these containers runs as root.