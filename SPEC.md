# SOFTWARE DESIGN DOCUMENT (SDD) - AGENT INSTRUCTOR

## 1. SYSTEM OBJECTIVE

### 1.1 Purpose Statement
The primary objective of **Agent Instructor** is to automate the generation of detailed, context-aware development instructions. It functions as a command-line orchestrator that analyzes a local codebase to identify relevant files for a given task (either a new feature or a bug fix) and then uses an external AI tool (`fabric-ai`) to compile a comprehensive `INSTRUCTIONS.md` file.

### 1.2 High-Level Technical Objectives
*   **Context-Aware Code Analysis:** Implement a local Retrieval-Augmented Generation (RAG) pipeline to dynamically identify and rank source code files based on their semantic relevance to a user-provided task description.
*   **Declarative Configuration:** Allow fine-grained control over the analysis process (file inclusion/exclusion, model selection, etc.) through a simple `config.json` file.
*   **External AI Integration:** Interface with the `fabric-ai` command-line tool, passing it a structured context containing project conventions, relevant code snippets, and the specific task objective.
*   **Stateless, Deterministic Operation:** Operate as a pure command-line utility that reads from the local file system, executes its logic, and writes a single output file (`INSTRUCTIONS.md`) without maintaining any persistent state or requiring a database.
*   **Mode-Based Instruction Generation:** Support distinct operational modes for `'feature'` development and `'bug'` fixing, using different input prompts and context for each scenario.

---

## 2. ARCHITECTURE AND DESIGN RULES

The system is designed as a **stateless command-line orchestration script**. Its architecture is simple and functional, prioritizing a clear, sequential flow over complex patterns like DDD or Hexagonal Architecture, which are not necessary for this use case.

```
       [User via CLI]
              |
              v
+-----------------------------+
|   agent-instructor (Bash)   |
+-----------------------------+
              |
              v
+-----------------------------+
|    instructor.py (Python)   |
|  (Orchestration & RAG Core) |
+-----------------------------+
              |
              v
|---------------------------------|
| 1. Load config.json             |
| 2. Load CONVENTIONS.md          |
| 3. Run RAG to find files        |
| 4. Read relevant file content   |
| 5. Prepare payload              |
|---------------------------------|
              |
              v
+-----------------------------+
|      fabric-ai (Ext. CLI)     |
+-----------------------------+
              |
              v
    [INSTRUCTIONS.md (Output)]
```

### 2.1 Dependency and Environment
1.  **Python Core & Virtual Environment:** The main logic resides in `instructor.py`. It requires Python 3.8+ and specific libraries (`sentence-transformers`, `numpy`). To keep dependencies isolated, the orchestrator is designed to run within a local virtual environment (`.venv`). The launcher script (`agent-instructor`) is self-aware and automatically prioritizes execution via `.venv/bin/python` if it exists.
2.  **External CLI Tool:** The system has a hard dependency on the `fabric-ai` CLI tool, which must be installed (e.g., via `pipx`) and available in the system's PATH. The script verifies its presence on startup.
3.  **Local AI Models:** The RAG process downloads and uses a `sentence-transformers` model locally. The model name is specified in `config.json`.
4.  **Global Execution Setup:** To run the orchestrator system-wide, the installation folder can be added to the user's shell PATH (e.g., in `.bashrc` or `.zshrc`):
    ```bash
    export PATH="/Volumes/ExtDrive/Home/rzavala/Documents/Projects/Koiosoft/agent-instructor:$PATH"
    ```
5.  **File-Based Contracts:** The orchestrator relies on the presence of specific files in the root directory to function:
    *   `CONVENTIONS.md`: Contains project-wide coding standards.
    *   `CUSTOM-PROMPT.txt`: System prompt for 'feature' mode.
    *   `CUSTOM-PROMPT-LOG.txt`: System prompt for 'bug' mode.
    *   `sdd.log`: Optional file containing error logs for 'bug' mode.

### 2.2 Project Topology and Directory Structure
The file hierarchy is minimal and centered around the main script and its configuration.

```
.
├── agent-instructor         # Bash script to launch the orchestrator.
├── instructor.py            # Main Python orchestration script with RAG logic.
├── config.json              # Declarative configuration for the RAG process.
├── CONVENTIONS.md           # (Required) Project coding and design rules.
├── CUSTOM-PROMPT.txt        # (Required) System prompt for feature generation.
├── CUSTOM-PROMPT-LOG.txt    # (Required) System prompt for bug fixing.
├── INSTRUCTIONS.md          # (Output) The generated instruction file.
└── sdd.log                  # (Optional) Log file used as context in bug-fixing mode.
```

---

## 3. DATA DESIGN AND PERSISTENCE

This system is **stateless** and does not use a database. All data is handled in-memory during a single execution or read from/written to the local filesystem.

*   **Configuration (`config.json`):** A JSON file defines the parameters for the RAG process. A default, fallback configuration is hardcoded in `instructor.py` in case this file is missing.
*   **Input Context:** All context (conventions, prompts, relevant code) is read from files and aggregated into a single payload for the `fabric-ai` tool.
*   **Output (`INSTRUCTIONS.md`):** The final output is written to a single markdown file, overwriting any previous content.
*   **Logging:** The script logs its a summary of its payload to a timestamped file in a `logs/` directory for debugging and traceability.

---

## 4. CORE COMPONENTS

### 4.1 RAG Localization (`run_rag_localization`)
*   **Purpose:** To find the most relevant files in the codebase for a given task.
*   **Engine:** Uses the `sentence-transformers` library to calculate the cosine similarity between the task description (the "problem text") and the content of each file in the project.
*   **Process:**
    1.  Recursively finds all files matching the `include_extensions` from `config.json`, while respecting `exclude_dirs` and `exclude_extensions`.
    2.  Encodes the user's task description into a vector embedding.
    3.  Encodes the content of each file into vector embeddings.
    4.  Compares the task embedding with all file embeddings to find the `top_n` files with the highest similarity score.
    5.  Returns a list of relative paths to these top files.

### 4.2 Fabric Integration (`call_fabric`)
*   **Purpose:** To generate the final `INSTRUCTIONS.md` using the `fabric-ai` tool.
*   **Process:**
    1.  Constructs a dynamic `system_prompt` and a `user_payload`. The payload contains the project conventions, the content of the relevant code files identified by the RAG process, and the specific objective.
    2.  Invokes the `fabric` CLI as a subprocess.
    3.  Pipes the `user_payload` to the subprocess's `stdin`.
    4.  Captures the `stdout` from `fabric`, which contains the generated markdown instructions.
    5.  Handles any errors from the subprocess.

### 4.3 Main Orchestrator (`main`)
*   **Purpose:** To control the end-to-end workflow.
*   **Process:**
    1.  Parses command-line arguments (`--mode`).
    2.  Loads the `rag_config` and `CONVENTIONS.md`.
    3.  Verifies the `fabric` CLI is available.
    4.  Determines the "problem text" based on the mode (`feature` or `bug`).
    5.  Calls `run_rag_localization` to get the list of relevant files.
    6.  Assembles the code context from those files.
    7.  Calls `call_fabric` with the complete context to get the final instructions.
    8.  Writes the result to `INSTRUCTIONS.md`.

---

## 5. CODING AND QUALITY STANDARDS

### 5.1 Code Quality and Style
*   **Typing:** The code uses Python type annotations (`pathlib.Path`, `str`, etc.) for function signatures to improve clarity.
*   **Style:** The code follows general PEP 8 standards for formatting and naming.
*   **Modularity:** The script is broken down into distinct functions with clear responsibilities (e.g., `load_rag_config`, `verify_fabric`, `run_rag_localization`).
*   **User Feedback:** The script provides clear, color-coded feedback to the user's console for each step of the process (`print_step`, `print_success`, `print_error`).

### 5.2 Error Handling
*   The script performs critical pre-flight checks, such as verifying the existence of `fabric-ai` and `CONVENTIONS.md`, exiting gracefully with an informative error message if a check fails.
*   JSON parsing errors in `config.json` are caught and reported.
*   Errors from the `fabric` subprocess are captured from `stderr` and displayed to the user.
