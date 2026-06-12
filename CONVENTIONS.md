# DEVELOPMENT AND CODING CONVENTIONS - AGENT INSTRUCTOR

## 1. PYTHON LANGUAGE STANDARDS & TYPING

*   **PEP 8 Compliance:** All code must adhere to PEP 8 style guidelines for readability and consistency.
*   **Strict Type Hinting:** Every function and method signature must include explicit type hints for all arguments and return values, using the `typing` module and standard types (e.g., `str`, `Path`, `list`). This is crucial for code clarity and static analysis.
*   **Prohibition of `Any`:** The use of `typing.Any` should be avoided. Use specific types wherever possible.
*   **Naming Conventions:**
    *   `snake_case`: For all variables, functions, and methods (e.g., `run_rag_localization`).
    *   `PascalCase`: For all classes (if any are added).
    *   `UPPER_SNAKE_CASE`: For all global constants (e.g., `ROOT_DIR`, `DEFAULT_CONFIG`).

## 2. ARCHITECTURE AND CODE STRUCTURE

*   **Functional Modularity:** The orchestrator is a single script, but logic must be separated into distinct functions with clear, single responsibilities. For example, keep configuration loading, RAG processing, and external CLI calls in separate, well-named functions.
*   **No Infrastructure Leakage:** Core logic functions should not be cluttered with presentation details. For instance, the RAG logic in `run_rag_localization` should return data structures (like a list of file paths), and a different function should handle printing that output to the console.
*   **Configuration in `config.json`:** All configurable parameters for the RAG process (file paths, model names, thresholds) must be defined in `config.json`. Avoid hardcoding these values directly in the `instructor.py` script. A `DEFAULT_CONFIG` dictionary should only be used as a fallback.

## 3. DEPENDENCY MANAGEMENT & ENVIRONMENT

*   **Virtual Environment Isolation:** To prevent library conflicts and keep system environments clean, a local Python virtual environment (`.venv`) must always be used for development and runtime execution.
*   **Self-Aware Launcher:** The execution launcher `agent-instructor` must be used to run the application, as it automatically detects and targets the local `.venv` Python interpreter without requiring manual shell activation.
*   **Python Libraries:** Core dependencies like `sentence-transformers` and `numpy` are managed via `requirements.txt` and installed via `pip` within the `.venv`. Any new library additions must be justified, documented, and appended to `requirements.txt`.
*   **External CLI Tools:** The project depends on `fabric-ai` being globally available. This is a critical dependency. The installation requirement (`pipx install fabric-ai`) must be checked for and clearly communicated in error messages if the tool is missing.

## 4. LOGGING AND ERROR HANDLING

*   **Console Output via Helpers:** All console output must be routed through the provided helper functions: `print_step()`, `print_success()`, `print_warning()`, and `print_error()`. The use of the standard `print()` function directly within the main application logic is forbidden to ensure consistent, color-coded user feedback.
*   **Pre-flight Checks:** The script must perform checks for critical dependencies and files at startup (e.g., `fabric` CLI, `CONVENTIONS.md`). If a check fails, the script must exit immediately using `sys.exit(1)` after printing a clear error message via `print_error()`.
*   **Handle External Process Failures:** When calling external processes like `fabric`, always check the return code. If the process returns a non-zero exit code, capture `stderr`, log it using `print_error()`, and terminate the script. Do not allow the script to continue with potentially corrupted or incomplete data.
*   **Explicit Exception Handling:** Avoid broad `except Exception:` clauses. Catch specific, expected exceptions (e.g., `json.JSONDecodeError`, `FileNotFoundError`) and provide a specific, helpful error message.

## 5. FILE-BASED CONTRACTS

*   The system's operation is dependent on a set of files in the project root (`CONVENTIONS.md`, `CUSTOM-PROMPT.txt`, etc.). These are considered part of the system's "API."
*   Any function reading these files should assume they exist. The initial check in `main()` is the designated gatekeeper for their presence.
*   All file I/O must explicitly specify `encoding="utf-8"` to ensure cross-platform compatibility.
