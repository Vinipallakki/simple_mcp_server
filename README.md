
# Zoo MCP Server on Cloud Run

This project shows how to build and deploy a **secure Model Context Protocol (MCP)** server using **FastMCP** on **Google Cloud Run**. It includes tools for querying zoo animals by species and getting details for individual animals. The server is set up to require authentication so only authorized clients can invoke its tools.

---

## 🚀 What you'll learn

- How to build a simple MCP server with FastMCP  
- How to containerize it and deploy to Cloud Run  
- Securing the service endpoint so that only authenticated requests are allowed  
- Connecting a client (e.g. Gemini CLI) to your remote MCP server  

---

## 📋 Project structure

```

.
├── Dockerfile
├── pyproject.toml
├── server.py
└── README.md

````

- `server.py` — Python MCP server code with tools for animal lookup  
- `Dockerfile` — To containerize the application  
- `pyproject.toml` — Dependency and project config  
- `README.md` — This file  

---

## 🔧 Prerequisites

- A Google Cloud account  
- A Google Cloud project with billing enabled  
- `gcloud` SDK installed and configured  
- (Optional) Gemini CLI if you want to test from CLI  

---

## ⚙️ Setup Guide

Below are the high-level steps to setup and deploy the MCP server.

### 1. Enable required APIs

```bash
gcloud services enable \
  run.googleapis.com \
  artifactregistry.googleapis.com \
  cloudbuild.googleapis.com
````

### 2. Initialize the Python project

```bash
mkdir zoo-mcp-server && cd zoo-mcp-server

# Create project config (assuming using `uv` for project scaffolding)
uv init --description "Zoo MCP server on Cloud Run" --bare --python 3.13
```

Add `fastmcp` as a dependency. For example in `pyproject.toml`:

```toml
[project]
name = "zoo-mcp-server"
version = "0.1.0"
description = "Zoo MCP Server example"
requires-python = ">=3.13"

[dependencies]
fastmcp = "2.11.1"
```

### 3. Write the server code (`server.py`)

Here’s a simplified example:

```python
import asyncio
import logging
import os
from typing import List, Dict, Any

from fastmcp import FastMCP

logger = logging.getLogger(__name__)
logging.basicConfig(
    format="[%(levelname)s]: %(message)s",
    level=logging.INFO
)

mcp = FastMCP("Zoo Animal MCP Server")

# In-memory data store
ZOO_ANIMALS = [
    {"species": "lion", "name": "Leo", "age": 7, "enclosure": "The Big Cat Plains", "trail": "Savannah Heights"},
    {"species": "penguin", "name": "Waddles", "age": 2, "enclosure": "The Arctic Exhibit", "trail": "Polar Path"},
    # … add more animals …
]

@mcp.tool()
def get_animals_by_species(species: str) -> List[Dict[str, Any]]:
    logger.info(f">>> Tool: 'get_animals_by_species' called for '{species}'")
    return [animal for animal in ZOO_ANIMALS if animal["species"].lower() == species.lower()]

@mcp.tool()
def get_animal_details(name: str) -> Dict[str, Any]:
    logger.info(f">>> Tool: 'get_animal_details' called for '{name}'")
    for animal in ZOO_ANIMALS:
        if animal["name"].lower() == name.lower():
            return animal
    return {}

if __name__ == "__main__":
    port = int(os.getenv("PORT", 8080))
    logger.info(f"Starting MCP server on port {port}")
    asyncio.run(
        mcp.run_async(
            transport="http",
            host="0.0.0.0",
            port=port,
        )
    )
```

### 4. Dockerize

Create `Dockerfile`:

```dockerfile
# Use official Python image
FROM python:3.13-slim

# Copy uv tool (if using uv scaffolding)
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/

# Copy project
COPY . /app
WORKDIR /app

# Ensure logs appear promptly
ENV PYTHONUNBUFFERED=1

# Install dependencies
RUN uv sync

# Expose port
EXPOSE $PORT

# Command to run the server
CMD ["uv", "run", "server.py"]
```

### 5. Deploy to Cloud Run

```bash
gcloud run deploy zoo-mcp-server \
  --no-allow-unauthenticated \
  --region=<YOUR_REGION> \
  --source=. \
  --labels=env=prod,project=zoo-mcp
```

* The flag `--no-allow-unauthenticated` ensures that only authenticated requests can invoke the MCP server.
* Make sure to choose your preferred region.

---

## 🔐 Setting up access & client

Once deployed, you’ll want to allow your user (or service account) to invoke the service:

```bash
gcloud projects add-iam-policy-binding $GOOGLE_CLOUD_PROJECT \
  --member=user:$(gcloud config get-value account) \
  --role='roles/run.invoker'
```

You can also retrieve an `ID_TOKEN` for use with clients:

```bash
export ID_TOKEN=$(gcloud auth print-identity-token)
```

### Configuring Gemini CLI (or other MCP clients)

In your client config (e.g. `~/.gemini/settings.json`), add:

```json
{
  "mcpServers": {
    "zoo-remote": {
      "httpUrl": "https://<YOUR_CLOUD_RUN_SERVICE_URL>/mcp/",
      "headers": {
        "Authorization": "Bearer <YOUR_ID_TOKEN>"
      }
    }
  },
  "selectedAuthType": "cloud-shell",
  "hasSeenIdeIntegrationNudge": true
}
```

---

## 🧰 Logs & Debugging

* To see if tool calls are working:

  ```bash
  gcloud run services logs read zoo-mcp-server --region=<YOUR_REGION> --limit=10
  ```

* A common issue: **authentication errors** — make sure your client’s bearer token is valid. If the ID token times out, re-fetch it.

---

## 🧼 Cleanup

If you want to avoid ongoing charges:

```bash
gcloud projects delete $GOOGLE_CLOUD_PROJECT
```

Or manually delete:

* The Cloud Run service
* Artifact Registry repository storing your container images

---

## 📌 References

* Codelab: *How to deploy a secure MCP server on Cloud Run* — Google Developers ([Google Codelabs][1])
* FastMCP documentation ([Google Codelabs][1])

---

**Happy building!** 🎉
If you need help adjusting for your own use-case (different tools, data source, region, etc.), I can help you tweak this.

[1]: https://codelabs.developers.google.com/codelabs/cloud-run/how-to-deploy-a-secure-mcp-server-on-cloud-run "How to deploy a secure MCP server on Cloud Run  |  Google Codelabs"
