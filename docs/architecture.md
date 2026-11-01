# VideoMind AI — System Architecture

## 1. Project Overview

VideoMind AI is a multimodal Retrieval-Augmented Generation (RAG) system integrated with a Chrome extension for YouTube.

The main objective is to provide users with an AI-powered assistant that understands the context of the currently playing YouTube video and answers questions related to its content.

Users can enable the Chrome extension while watching a YouTube video and interact with it through a chatbot-style interface.

### Problem Statement

Users often face difficulties understanding educational videos because of language barriers, complex technical explanations, lengthy content, or unfamiliar concepts.

VideoMind AI aims to solve these problems by allowing users to:

- Ask questions about the currently playing YouTube video.
- Understand complex concepts through contextual explanations.
- Translate video explanations into languages such as Hindi.
- Get answers grounded in the video's transcript.
- Navigate to relevant video timestamps through citations.
- Interact through text or voice.
- Eventually understand visual information from video frames.

### Example Use Case

Suppose a user is watching an English-language machine learning lecture but is more comfortable with Hindi.

The workflow is:

1. The user opens a YouTube video.
2. The user enables the VideoMind AI Chrome extension.
3. The extension identifies the currently playing video.
4. The backend obtains an available, permitted transcript.
5. The user asks: "Explain gradient descent in Hindi."
6. The RAG pipeline retrieves relevant transcript segments.
7. The LLM generates a contextual explanation in Hindi.
8. The response includes references to relevant video timestamps.

The objective is to make YouTube content easier to understand through an intelligent, video-aware assistant.

## 2. Architecture Style — Modular Monolith

VideoMind AI follows a Modular Monolith architecture.

A modular monolith is a software architectural approach in which the application is deployed as a single system but internally organized into independent modules with clearly defined responsibilities and interfaces.

Instead of developing multiple independently deployed microservices, we maintain a unified backend application with well-separated modules.

### Planned Modules

- **Ingestion Module:** Handles video metadata, transcript acquisition, cleaning and chunking.
- **RAG Module:** Coordinates retrieval, context construction and answer generation.
- **Retrieval Module:** Handles dense retrieval, BM25, RRF fusion and reranking.
- **LLM Gateway Module:** Provides a unified interface for different LLM providers.
- **Authentication Module:** Manages user authentication and authorization.
- **Evaluation Module:** Measures retrieval quality, answer faithfulness and system performance.

### Why Modular Monolith?

We choose this architecture because:

- It reduces deployment and infrastructure complexity.
- It provides clear separation of responsibilities.
- It makes debugging and testing easier.
- It supports independent development of internal modules.
- It is suitable for our initial development scope.
- Individual modules can potentially be extracted into independent services if future scaling requirements justify it.

The objective is to maintain clean internal boundaries without introducing unnecessary distributed-system complexity.

## 3. Core Technologies

### FastAPI

FastAPI is the Python backend framework used to expose APIs and handle HTTP requests.

Responsibilities:
- Accept requests from the Chrome extension.
- Handle video ingestion requests.
- Expose chatbot and retrieval endpoints.
- Manage request validation.
- Return responses to the frontend.

### PostgreSQL

PostgreSQL is the primary relational database and source of truth for application data.

It stores:
- User information.
- Video metadata.
- Transcript and chunk metadata.
- Processing job status.
- Application configurations.
- Relevant relationships between users and videos.

### Qdrant

Qdrant is a vector database designed for high-performance vector storage and similarity search.

Responsibilities:
- Store transcript embeddings.
- Perform semantic similarity searches.
- Retrieve relevant transcript chunks.
- Support metadata-based filtering.

It plays a central role in the dense retrieval component of our RAG pipeline.

### Redis

Redis is an in-memory data store used for multiple infrastructure responsibilities.

Responsibilities:
- Celery message brokering.
- Caching frequently accessed information.
- Temporary state management.
- Rate limiting.

### Celery

Celery is a distributed task queue used to execute time-consuming background operations asynchronously.

Responsibilities:
- Transcript processing.
- Text cleaning and chunking.
- Embedding generation.
- Video ingestion workflows.
- Other long-running background operations.

This prevents time-consuming ingestion tasks from blocking normal API requests.

### LLM Gateway

The LLM Gateway is an abstraction layer between the application and external or local language model providers.

It provides a unified interface for communicating with providers such as OpenAI, Gemini, Claude and Ollama.

Responsibilities:
- Provider abstraction.
- Model selection and routing.
- Retry mechanisms.
- Automatic fallback.
- Usage and cost tracking.
- Latency monitoring.
- Rate limits and budget controls.

The objective is to avoid tightly coupling the application to a single LLM provider.

## 4. Video Ingestion Workflow

When a user submits a YouTube video URL, the following process takes place:

1. The Chrome extension sends the video URL and relevant video information to FastAPI.
2. FastAPI validates the request.
3. The backend creates an ingestion job and records its status in PostgreSQL.
4. Celery receives the background task through Redis.
5. The ingestion module obtains the permitted transcript.
6. The transcript is cleaned and normalized.
7. The transcript is divided into meaningful chunks while preserving timestamps and metadata.
8. An embedding model converts the text chunks into vector representations.
9. The embeddings and associated metadata are stored in Qdrant.
10. PostgreSQL maintains the video's metadata and processing status.

Once ingestion is complete, the video becomes available for contextual question answering.

Content acquisition will use user-provided transcripts, owner-authorized captions and other permitted source adapters. We will not depend on undocumented YouTube scraping.

## 5. RAG Query Workflow

The RAG pipeline is responsible for retrieving relevant video content and generating contextually accurate answers.

### Initial Dense Retrieval Concept

At a basic level, the user query is converted into an embedding. Qdrant compares the query embedding with stored transcript embeddings and retrieves semantically similar chunks.

However, VideoMind AI will implement an advanced hybrid retrieval pipeline.

### Planned Hybrid RAG Workflow

1. **Query Rewriting:** Reformulates the user's question when necessary to improve retrieval.
2. **Dense Retrieval:** Uses embeddings to identify semantically relevant transcript chunks.
3. **BM25 Retrieval:** Performs keyword-based retrieval.
4. **RRF Fusion:** Combines the rankings produced by dense and keyword retrieval.
5. **Cross-Encoder Reranking:** Reorders the retrieved chunks according to their relevance to the query.
6. **Parent Expansion:** Retrieves additional surrounding context when necessary.
7. **Context Construction:** Prepares the relevant transcript segments for the LLM.
8. **LLM Generation:** Generates an answer using the retrieved context.
9. **Citation Validation:** Validates references against the retrieved video segments.
10. **Response Generation:** Returns the answer along with relevant video timestamps.

The number of retrieved chunks will be configurable and determined through evaluation rather than permanently fixed at three or four.

### Why Hybrid RAG?

Dense retrieval understands semantic relationships, while BM25 is effective at matching exact keywords.

Combining both approaches improves the chances of retrieving relevant information, particularly when users ask questions using different terminology from the original video.

## 6. Multimodal Capabilities

VideoMind AI is designed to evolve beyond transcript-based question answering.

Planned multimodal capabilities include:

- **Text:** Transcript-based retrieval and question answering.
- **Translation:** Generating explanations in the user's preferred language.
- **Images and Video Frames:** On-demand visual understanding and authorized video-frame analysis.
- **Audio:** Speech recognition and audio-based interaction.
- **Voice Assistant:** Push-to-talk input and spoken AI responses.

These capabilities will be implemented incrementally, starting with transcript-based RAG.

## 7. Technology Decisions

The selected technologies provide a practical foundation for developing the application.

- FastAPI provides a lightweight Python API framework.
- PostgreSQL provides reliable structured data storage.
- Qdrant provides specialized vector search capabilities.
- Redis and Celery support asynchronous background processing.
- The LLM Gateway enables provider flexibility.
- The Modular Monolith keeps the architecture manageable while preserving clear internal boundaries.

The architecture prioritizes maintainability, modularity, evaluation and the ability to extend the system as the project evolves.