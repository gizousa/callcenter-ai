# Call Center AI - Developer Guide

This is an AI-powered call center solution built with FastAPI, Azure Communication Services, and Azure OpenAI. The system handles real-time phone calls with intelligent conversation flow, claim processing, and multi-modal communication.

## Architecture Overview

The application follows a **modular, event-driven architecture** with clear separation between:

- **Event handlers** (`app/helpers/call_events.py`) - Process Communication Services webhooks
- **LLM orchestration** (`app/helpers/llm_worker.py`, `app/helpers/call_llm.py`) - Manage AI conversations
- **Persistence layer** (`app/persistence/`) - Abstract interfaces for storage (Cosmos DB, Redis, AI Search)
- **Models** (`app/models/`) - Pydantic models for data validation and serialization

Key architectural patterns:

- **Interface-based design**: All persistence layers implement abstract base classes (e.g., `ICache`, `IStore`, `ISearch`)
- **Configuration-driven**: Single YAML config file drives all Azure resource connections
- **Async-first**: Uses `asyncio`, `aiohttp`, and async Azure SDKs throughout
- **Telemetry integration**: OpenTelemetry spans wrap all major operations

## Core Data Flow

1. **Inbound calls** → Azure Communication Services → Event Grid → FastAPI webhooks (`/webhook/*`)
2. **Real-time audio** → WebSocket streaming → Speech-to-Text → LLM processing → Text-to-Speech
3. **Conversation state** stored in `CallStateModel` → persisted to Cosmos DB
4. **Claims data** extracted via LLM tools → structured in configurable schema

## Development Workflow

### Local Development

```bash
# Setup environment (uses uv for Python package management)
make install

# Run locally with auto-reload
make dev

# Sync remote Azure config to local
make name=my-rg-name sync-local-config
```

### Configuration Management

- **Local**: `config.yaml` (copy from `config-local-example.yaml`)
- **Remote**: Azure App Configuration + environment variables
- **Validation**: Pydantic models in `app/helpers/config_models/`

### Testing Patterns

- **Unit tests**: Mock Azure clients using custom mock classes (see `tests/conftest.py`)
- **LLM evaluation**: Uses DeepEval for conversation quality assessment
- **Async testing**: `pytest-asyncio` for async test functions

## Key Components to Understand

### Event-Driven Call Handling

- `on_call_connected()` - Initiates conversation flow
- `on_automation_recognize_error()` - Handles speech recognition failures with retry logic
- `on_audio_connected()` - Starts real-time audio streaming for low-latency responses

### LLM Integration

- **Dual LLM strategy**: Fast model (`gpt-4.1-nano`) for real-time, slow model (`gpt-4.1`) for complex reasoning
- **Tool calling**: LLM can execute predefined functions for claim updates, SMS sending, call transfers
- **Context management**: Conversation history maintained in `CallStateModel.messages`

### Audio Processing

- **Noise reduction**: `noisereduce` library for audio cleanup
- **Multi-language TTS**: Azure Cognitive Services with voice selection based on conversation language
- **Streaming optimizations**: Audio chunks processed in real-time to minimize latency

## Deployment & Infrastructure

### Azure Resources (Bicep)

- **Core**: `cicd/bicep/main.bicep` → `app.bicep` (modular deployment)
- **Container deployment**: GitHub Actions builds container → Azure Container Apps
- **Resource naming**: Uses sanitized instance names with consistent tagging

### Makefile Commands

```bash
make deploy name=my-rg-name          # Deploy to Azure
make logs name=my-rg-name            # View application logs
make version                         # Get semantic version
make tunnel-create                   # Setup dev tunnel for webhooks
```

## Project-Specific Conventions

### Code Organization

- **Helpers**: Pure functions and utilities, no state management
- **Models**: Pydantic models with validation, used for API contracts and database schemas
- **Persistence**: Interface implementations, each Azure service has its own module

### Error Handling

- **Graceful degradation**: Call continues even if non-critical services fail
- **Retry logic**: Uses `tenacity` for resilient external service calls
- **Structured logging**: `structlog` with OpenTelemetry correlation IDs

### Configuration Validation

All config is validated at startup using Pydantic. Missing or invalid Azure credentials cause immediate startup failure with clear error messages.

## Integration Points

- **Azure Communication Services**: Call automation, SMS, phone number management
- **Azure OpenAI**: Dual deployment strategy (fast/slow models) with embedding search
- **Azure AI Search**: RAG for training data and knowledge base
- **Azure Cosmos DB**: Conversation persistence with partitioning by phone number
- **Redis**: Caching layer for frequently accessed data and session state

## Development Tips

- **Mock Azure services** in tests using the patterns in `tests/conftest.py`
- **Use async context managers** for resource cleanup (see `@asynccontextmanager` in `main.py`)
- **Follow the interface pattern** when adding new persistence layers
- **Test conversation flows** using the YAML-based conversation definitions in `tests/conversations.yaml`
