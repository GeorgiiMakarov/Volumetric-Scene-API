# Volumetric-Scene-API

A robust and scalable backend solution for managing and processing volumetric 3D data (like `.splat`, `.glb`, `.gltf` files), designed for easy integration with frontend viewers or Mixed Reality (MR) SDKs. This project demonstrates a complete architectural approach, from secure user authentication and efficient file uploads directly to AWS S3, to asynchronous background processing for data conversion and validation.

---

## Features

* **User Authentication:** Secure user registration and login using JWT (JSON Web Tokens).
* **File Uploads:** Accepts `.splat`, `.glb`, and `.gltf` files.
* **Scalable Storage:** Files are stored securely and efficiently on AWS S3.
* **Asynchronous Processing:** Long-running tasks (e.g., file validation, conversion) are handled in the background by Celery workers.
* **Data Persistence:** Utilizes PostgreSQL for storing user, upload, and task metadata.
* **Containerized Deployment:** Entire stack orchestrated with Docker Compose for easy setup and scalability.

---

## Tech Stack

This project is built using a modern and powerful set of technologies:

* **Backend Framework:** FastAPI (Python) - High-performance, easy-to-use web framework.
* **Asynchronous Task Queue:** Celery with Redis broker - For background processing.
* **Database:** PostgreSQL with SQLAlchemy ORM - Reliable and robust data storage.
* **Cloud Storage:** AWS S3 (`boto3`) - For scalable and persistent file storage.
* **Authentication:** JWT (JSON Web Tokens) - Secure user authentication.
* **Containerization:** Docker & Docker Compose - For reproducible development and deployment environments.
* **Python Utilities:** `python-dotenv`, `uvicorn`, `psycopg2-binary`, `pydantic`.

---

## Folder Structure

The project follows a clear and modular structure to maintain separation of concerns:
volumetric-scene-api/
├── app/
│   ├── main.py              # FastAPI application entry point
│   ├── models.py            # SQLAlchemy database models
│   ├── db.py                # Database engine and session setup
│   ├── config.py            # Application-wide configurations
│   ├── tasks.py             # Celery task definitions
│   ├── s3_utils.py          # Utilities for S3 interaction
│   └── scene_processor.py   # Core logic for 3D scene processing (e.g., conversion)
├── Dockerfile               # Docker build instructions for the application
├── docker-compose.yml       # Docker Compose setup for all services
├── requirements.txt         # Python dependencies
├── .env.example             # Example environment variables
└── README.md                # This project documentation

---

## Getting Started

Follow these steps to set up and run the Volumetric Scene API locally.

### Prerequisites

* Docker Desktop (includes Docker Engine and Docker Compose)

### Installation

1.  **Clone the repository:**
    ```bash
    git clone [https://github.com/YourGitHubUsername/volumetric-scene-api.git](https://github.com/YourGitHubUsername/volumetric-scene-api.git) # Replace YourGitHubUsername
    cd volumetric-scene-api
    ```

2.  **Configure Environment Variables:**
    Create a `.env` file in the root directory of the project based on `.env.example`.
    ```bash
    cp .env.example .env
    ```
    Open `.env` and fill in your details, particularly your AWS credentials and S3 bucket name.
    ```dotenv
    # .env
    POSTGRES_USER=user
    POSTGRES_PASSWORD=pass
    POSTGRES_DB=volumetric # As per docker-compose.yml
    SECRET_KEY=supersecretkey_replace_with_a_long_random_string # IMPORTANT!
    ALGORITHM=HS256
    ACCESS_TOKEN_EXPIRE_MINUTES=30
    REDIS_URL=redis://redis:6379/0

    # AWS S3 Configuration
    AWS_ACCESS_KEY_ID=YOUR_AWS_ACCESS_KEY_ID_HERE
    AWS_SECRET_ACCESS_KEY=YOUR_AWS_SECRET_ACCESS_KEY_HERE
    AWS_REGION_NAME=your-aws-region # e.g., us-east-1
    S3_BUCKET_NAME=your-splat-bucket-name
    ```
    **IMPORTANT:** Never commit your actual `.env` file to version control. The `.gitignore` should exclude it.

3.  **Build and Run with Docker Compose:**
    ```bash
    docker-compose up --build
    ```
    This command will:
    * Build the Docker images for your `api` (FastAPI) and `worker` (Celery) services.
    * Start PostgreSQL (`db`) and Redis (`redis`) containers.
    * Run FastAPI application accessible at `http://localhost:8000`.
    * Start the Celery worker for background tasks.

    *(**Note:** Ensure your database models are created on startup. You might need to add Alembic migrations or a simple startup script to `main.py` to create tables.)*

---

## API Usage

The API provides endpoints for authentication and file uploads. You can access the interactive API documentation (Swagger UI) at `http://localhost:8000/docs`.

### Authentication

* **Sign Up:** `POST /auth/signup`
    ```bash
    curl -X POST "http://localhost:8000/auth/signup" -H "Content-Type: application/json" -d '{
      "username": "testuser",
      "password": "testpassword",
      "email": "test@example.com"
    }'
    ```
* **Login:** `POST /auth/login`
    ```bash
    curl -X POST "http://localhost:8000/auth/login" -H "Content-Type: application/json" -d '{
      "username": "testuser",
      "password": "testpassword"
    }'
    # Response will include an access_token. Use this as a Bearer token for protected endpoints.
    ```

### File Upload

* **Upload .splat, .glb, .gltf:** `POST /upload/`
    Requires a `Bearer` token in the `Authorization` header.
    ```bash
    curl -X POST "http://localhost:8000/upload/" \
      -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
      -H "accept: application/json" \
      -H "Content-Type: multipart/form-data" \
      -F "file=@/path/to/your/model.splat" # Replace with your file path
    ```
    Files will be uploaded to AWS S3, and a background Celery task (`process_scene_async`) will be initiated for further processing.

---

## Core Configuration & Infrastructure Files

These files define the environment and how different parts of the application communicate.

### `Dockerfile`

This Dockerfile builds the Python application image.

```dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app/ ./app

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]

docker-compose.yml
Defines and runs the multi-container Docker application.
# docker-compose.yml
version: "3.9"
services:
  api:
    build: .
    ports:
      - "8000:8000"
    depends_on:
      - redis
      - db
    environment: # These should ideally come from .env in production
      - AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
      - AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}
      - AWS_BUCKET_NAME=${AWS_BUCKET_NAME}
      - REDIS_URL=${REDIS_URL}
      - DATABASE_URL=postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@db:5432/${POSTGRES_DB}
      - SECRET_KEY=${SECRET_KEY}
      - ALGORITHM=${ALGORITHM}
      - ACCESS_TOKEN_EXPIRE_MINUTES=${ACCESS_TOKEN_EXPIRE_MINUTES}

  redis:
    image: redis:7
    ports:
      - "6379:6379"

  db:
    image: postgres:15
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    ports:
      - "5432:5432"
    volumes: # IMPORTANT: Add this for persistent DB data
      - postgres_data:/var/lib/postgresql/data

  worker:
    build: .
    command: celery -A app.tasks worker --loglevel=info # Points to app/tasks.py for Celery app
    depends_on:
      - redis
      - api # Worker might need API for config or DB access

volumes: # Define named volume for PostgreSQL data
  postgres_data:

requirements.txt
Python dependencies for the project.
# requirements.txt
fastapi
uvicorn[standard]
pydantic
boto3
sqlalchemy
psycopg2-binary
celery
redis
python-multipart
python-dotenv

Key Python Scripts (Code Examples)
Here are snippets from the core Python files that implement the API's functionality.
app/config.py
Handles loading environment variables for application configuration.
# app/config.py
import os
from dotenv import load_dotenv
load_dotenv() # Load environment variables from .env file

AWS_ACCESS_KEY_ID = os.getenv("AWS_ACCESS_KEY_ID")
AWS_SECRET_ACCESS_KEY = os.getenv("AWS_SECRET_ACCESS_KEY")
AWS_BUCKET_NAME = os.getenv("AWS_BUCKET_NAME")
REDIS_URL = os.getenv("REDIS_URL")
DATABASE_URL = os.getenv("DATABASE_URL")
# Add other configurations as needed (SECRET_KEY, ALGORITHM, ACCESS_TOKEN_EXPIRE_MINUTES)

Note: It's recommended to use Pydantic BaseSettings for robust configuration management in FastAPI projects, as discussed previously. This snippet reflects your provided image.
app/db.py
Sets up the SQLAlchemy engine and session for database interactions.
# app/db.py
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from .config import DATABASE_URL # Assuming DATABASE_URL is defined in config.py

engine = create_engine(DATABASE_URL)

SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

app/models.py
Defines the SQLAlchemy model for VolumetricScene (or Upload as discussed previously, it seems you renamed it to VolumetricScene).
# app/models.py
from sqlalchemy import Column, Integer, String, DateTime
from sqlalchemy.ext.declarative import declarative_base
from datetime import datetime

Base = declarative_base() # Or from app.db.base import Base if structured differently

class VolumetricScene(Base):
    __tablename__ = "scenes" # Or "uploads"

    id = Column(Integer, primary_key=True, index=True)
    filename = Column(String, index=True)
    s3_url = Column(String) # URL or key in S3
    format = Column(String) # e.g., ".splat", ".glb"
    uploaded_at = Column(DateTime, default=datetime.utcnow)

app/s3_utils.py
Utility functions for interacting with AWS S3.
# app/s3_utils.py
import boto3
from .config import AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_BUCKET_NAME # Or import settings object

def upload_to_s3(filename, data):
    s3 = boto3.client("s3",
                      aws_access_key_id=AWS_ACCESS_KEY_ID,
                      aws_secret_access_key=AWS_SECRET_ACCESS_KEY) # Region name might be missing here

    s3.put_object(Bucket=AWS_BUCKET_NAME, Key=filename, Body=data)
    # Returns an S3 public URL (ensure your bucket policy allows public reads if you use this directly)
    return f"https://{AWS_BUCKET_NAME}[.s3.amazonaws.com/](https://.s3.amazonaws.com/){filename}"
Note: The s3_utils.py provided in your image is simplified. For a robust solution, you'd likely pass region_name to boto3.client as well, and upload_to_s3 might be an async function if called from FastAPI's async routes. Also, handling UploadFile objects directly (as in previous discussions) is more common than filename, data.
app/tasks.py
Defines Celery tasks for background processing.
# app/tasks.py
from celery import Celery
from .config import REDIS_URL # Or import settings object
from .scene_processor import process_scene # Assuming scene_processor.py exists and has this function

celery = Celery("tasks", # This name "tasks" should match the module Celery loads (app.tasks)
                broker=REDIS_URL)

@celery.task
def process_scene_async(filename, s3_url, format):
    print(f"Processing {filename} at {s3_url}")
    # Call the actual processing logic from scene_processor.py
    return process_scene(s3_url, format)

Note: You'll need to create app/scene_processor.py with the process_scene function that handles the actual 3D data processing (e.g., using Open3D).
app/scene_processor.py
Placeholder for the core 3D scene processing logic.
# app/scene_processor.py
def process_scene(s3_url, format):
    # Placeholder for future .splat/.glb optimization,
    # preview generation, etc.
    print(f"Processing scene at {s3_url} with format {format}")
    return True

app/main.py
The main FastAPI application file, defining API routes.
# app/main.py
from fastapi import FastAPI, UploadFile, File
from .tasks import process_scene_async
from .s3_utils import upload_to_s3 # Assuming s3_utils.py handles direct S3 upload

app = FastAPI()

@app.post("/upload/")
async def upload_scene(file: UploadFile = File(...)):
    contents = await file.read() # Read file content
    
    # Upload to S3
    s3_url = upload_to_s3(file.filename, contents) # Use original filename for S3 key initially
    
    # Extract format from filename
    file_format = file.filename.split('.')[-1]
    
    # Enqueue background processing task
    process_scene_async.delay(file.filename, s3_url, file_format)
    
    return {"status": "uploaded", "s3_url": s3_url}

Note: This main.py snippet is simplified. It lacks user authentication, dependency injection for database sessions, and proper error handling, which are crucial for a production-ready API. Your initial app/api/auth.py and app/api/upload.py and app/services structure is more robust. This example reflects the image you provided.
Live Demo / Preview
▶️ Watch a quick demo on YouTube (Coming Soon!)
(Once you have a frontend or video showcasing the API in action, replace this with the actual link.)
Contributing
We welcome contributions! Please see our CONTRIBUTING.md for details on how to get started.
License
This project is licensed under the MIT License - see the LICENSE file for details.
