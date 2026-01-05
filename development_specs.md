# Development Specifications: Human-Centric RAG System PoC

This document outlines the technical specifications for the Proof-of-Concept (PoC) of the Human-Centric Retrieval-Augmented Generation (RAG) System for RFP handling.

## 1. Project Setup

- **Language:** Python 3.10+
- **Framework:** FastAPI
- **Dependencies:**
  - `fastapi`
  - `uvicorn`
  - `pydantic`
  - `anthropic` (or a similar library for the chosen LLM)
- **Directory Structure:**
  ```
  .
  ├── main.py             # FastAPI application
  ├── agents.py           # AI agent functions
  ├── schemas.py          # Pydantic schemas
  ├── store.py            # Knowledge Module store
  ├── data/
  │   ├── modules.json    # Knowledge Module database
  │   └── rfps/           # Sample RFP documents
  └── tests/
      ├── test_agents.py
      └── test_store.py
  ```

## 2. Knowledge Module Schema

The `KnowledgeModule` schema will be defined in `schemas.py` using Pydantic for validation and serialization.

```python
from pydantic import BaseModel, Field
from typing import List, Optional
from datetime import datetime

class KnowledgeModule(BaseModel):
    module_id: str = Field(..., description="Stable, human-readable ID (e.g., 'capability-data-encryption-001')")
    title: str = Field(..., description="A concise, human-readable title for the module.")
    content: str = Field(..., description="The atomic claim, capability, or fact.")
    source_document_id: str = Field(..., description="Identifier for the source document.")
    source_excerpt: str = Field(..., description="The exact text from the source document that supports the content.")
    confidence_level: float = Field(..., ge=0.0, le=1.0, description="Confidence score from 0.0 to 1.0.")
    created_at: datetime = Field(default_factory=datetime.utcnow)
    tags: List[str] = Field(default_factory=list, description="Keywords or categories for discoverability.")
```

## 3. Knowledge Module Store

The `KnowledgeModuleStore` will be a simple class in `store.py` that handles persistence. For the PoC, it will read from and write to a JSON file.

```python
import json
from typing import List, Dict
from schemas import KnowledgeModule

class KnowledgeModuleStore:
    def __init__(self, db_path: str = "data/modules.json"):
        self.db_path = db_path
        self._modules: Dict[str, KnowledgeModule] = self._load()

    def _load(self) -> Dict[str, KnowledgeModule]:
        try:
            with open(self.db_path, "r") as f:
                data = json.load(f)
                return {mid: KnowledgeModule(**km) for mid, km in data.items()}
        except (FileNotFoundError, json.JSONDecodeError):
            return {}

    def add(self, module: KnowledgeModule):
        self._modules[module.module_id] = module
        self._save()

    def get(self, module_id: str) -> KnowledgeModule | None:
        return self._modules.get(module_id)

    def get_all(self) -> List[KnowledgeModule]:
        return list(self._modules.values())

    def _save(self):
        with open(self.db_path, "w") as f:
            json.dump({mid: km.dict() for mid, km in self._modules.items()}, f, indent=2, default=str)
```

## 4. AI Agent Functions

The AI agent functions will be defined in `agents.py`. Each function will wrap a call to a language model, providing a structured prompt and parsing the structured JSON response.

```python
from schemas import KnowledgeModule, RetrievalResult, DraftAnswer
from store import KnowledgeModuleStore
from typing import List

def distill_document_agent(raw_document: str) -> List[KnowledgeModule]:
    """Distills a raw document into a list of Knowledge Modules."""
    # Implementation will involve a structured prompt to an LLM
    # to extract atomic facts and their provenance.
    pass

def retrieve_modules_agent(question: str, module_store: KnowledgeModuleStore) -> RetrievalResult:
    """Selects relevant Knowledge Modules to answer a question."""
    # Implementation will involve a structured prompt to an LLM,
    # providing the question and a summary of available modules.
    pass

def assemble_answer_agent(question: str, selected_modules: List[KnowledgeModule]) -> DraftAnswer:
    """Assembles a draft answer using only the provided modules."""
    # Implementation will involve a strict, structured prompt to an LLM,
    # forbidding any external knowledge.
    pass
```
### Supporting Schemas for Agents
```python
from pydantic import BaseModel, Field
from typing import List

class SelectedModule(BaseModel):
    module_id: str
    rationale: str

class RetrievalResult(BaseModel):
    selected_modules: List[SelectedModule]
    raw_query: str

class DraftAnswer(BaseModel):
    draft_text: str
    citations: List[str] # List of module_ids
```

## 5. FastAPI Endpoints

The FastAPI application will be defined in `main.py`.

```python
from fastapi import FastAPI, HTTPException
from schemas import KnowledgeModule, RetrievalResult, DraftAnswer
from agents import distill_document_agent, retrieve_modules_agent, assemble_answer_agent
from store import KnowledgeModuleStore

app = FastAPI()
module_store = KnowledgeModuleStore()

@app.post("/ingest", response_model=List[KnowledgeModule])
async def ingest_document(document: dict):
    """Endpoint to ingest a new document."""
    raw_text = document.get("text")
    if not raw_text:
        raise HTTPException(status_code=400, detail="Document text is required.")

    distilled_modules = distill_document_agent(raw_text)
    for module in distilled_modules:
        module_store.add(module)
    return distilled_modules

@app.post("/ask", response_model=dict)
async def ask_question(question: dict):
    """Endpoint to ask a question and get a draft answer."""
    user_question = question.get("question")
    if not user_question:
        raise HTTPException(status_code=400, detail="Question is required.")

    retrieval_result = retrieve_modules_agent(user_question, module_store)

    if not retrieval_result.selected_modules:
        return {"message": "No relevant modules found."}

    selected_module_objects = [module_store.get(sm.module_id) for sm in retrieval_result.selected_modules]

    draft_answer = assemble_answer_agent(user_question, selected_module_objects)

    return {
        "user_question": user_question,
        "retrieval_result": retrieval_result,
        "draft_answer": draft_answer
    }
```

## 6. UI Data Structure

The `/ask` endpoint will return a JSON object that can be directly rendered by a minimal frontend. This JSON serves as the "Glass Box UI" data payload.

**Example Payload:**
```json
{
  "user_question": "What are our data encryption policies?",
  "retrieval_result": {
    "selected_modules": [
      {
        "module_id": "policy-encryption-at-rest-001",
        "rationale": "This module directly addresses the company's policy on data encryption at rest."
      },
      {
        "module_id": "policy-encryption-in-transit-001",
        "rationale": "This module covers the policy for data encryption while in transit."
      }
    ],
    "raw_query": "data encryption policies"
  },
  "draft_answer": {
    "draft_text": "We encrypt all customer data at rest using AES-256 [policy-encryption-at-rest-001]. Data in transit is protected using TLS 1.2+ [policy-encryption-in-transit-001].",
    "citations": [
      "policy-encryption-at-rest-001",
      "policy-encryption-in-transit-001"
    ]
  }
}
```

## 7. Sample Data

- **RFP Documents:** A few sample text files (or mocked snippets) will be placed in the `data/rfps/` directory. These should contain clear, extractable facts.
- **Pre-distilled Modules:** The `data/modules.json` file can be pre-populated with a few example `KnowledgeModule` objects to facilitate testing of the retrieval and assembly stages.

## 8. Logging Strategy

- **Agent Inputs/Outputs:** Log the full JSON input and output of each agent call to ensure inspectability.
- **Endpoint Calls:** Log incoming requests and outgoing responses.
- **Errors:** Log any exceptions that occur within the agents or endpoints.

This structured logging will be crucial for the 5-minute demo to showcase the system's determinism and transparency.
