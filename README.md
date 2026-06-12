# Agent Instructor

Agent Instructor is a command-line orchestrator that automates the generation of detailed, context-aware development instructions. It analyzes a local codebase using a local AI model (RAG) to find the most relevant files for a given task and then uses the `fabric-ai` CLI to compile a comprehensive `INSTRUCTIONS.md` file.

This tool is designed to provide a consistent and robust starting point for developers, ensuring that all instructions are grounded in the project's actual code and conventions.

## How It Works

The process is executed in a single, sequential flow:

1.  **Initialization**: The script loads its configuration from `config.json` and verifies that all external dependencies are available.
2.  **Mode Selection**: It runs in one of two modes:
    *   `feature` (default): To generate instructions for a new feature, using `CUSTOM-PROMPT.txt` as its primary goal.
    *   `bug`: To generate a fix for a bug, using `CUSTOM-PROMPT-LOG.txt` and the contents of `sdd.log` as its context.
3.  **Code Analysis (RAG)**: It uses a `sentence-transformers` model to read the files in your project and find the `top_n` most semantically relevant files related to the task description.
4.  **Context Assembly**: It creates a comprehensive payload containing:
    *   The project's global `CONVENTIONS.md`.
    *   The code from the relevant files found in the previous step.
    *   The specific objective for the feature or bug.
5.  **Instruction Generation**: It passes the entire payload to the `fabric-ai` CLI, which processes the context and generates the final markdown output.
6.  **Output**: The generated steps are saved to `INSTRUCTIONS.md`.

## Requirements

*   Python 3.8+
*   An installation of `pipx` for managing CLI tools.

## Installation

1.  **Clone the repository:**
    ```bash
    git clone <repository_url>
    cd agent-instructor
    ```

2.  **Create a virtual environment:**
    Setting up a virtual environment ensures all libraries are kept isolated and doesn't pollute your global Python system.
    ```bash
    # Create the virtual environment
    python3 -m venv .venv
    ```

3.  **Install the required Python libraries:**
    Activate the environment and install dependencies, or install directly using the environment's `pip`:
    ```bash
    .venv/bin/pip install --upgrade pip
    .venv/bin/pip install -r requirements.txt
    ```

4.  **Install the external `fabric-ai` CLI:**
    This tool is a critical dependency for compiling the final instructions.
    ```bash
    pipx install fabric-ai
    ```

5.  **Enable global execution (Optional):**
    To run the orchestrator from any directory in your system, add its installation path to your shell's configuration file (e.g., `~/.bashrc` or `~/.zshrc`):
    ```bash
    export PATH="/Volumes/ExtDrive/Home/rzavala/Documents/Projects/Koiosoft/agent-instructor:$PATH"
    ```
    After adding the line, reload your profile (`source ~/.bashrc` or `source ~/.zshrc`).
    
    *Note: The launcher script `agent-instructor` automatically detects and uses the internal `.venv` virtual environment in its directory, so you do not need to activate the environment manually when executing it globally.*

## Configuration

Before running the tool, ensure the following files are present in the project root:

1.  `config.json`: Configures the code analysis (RAG) process. You can define which file extensions to include/exclude, which directories to ignore, and which AI model to use.
2.  `CONVENTIONS.md`: A markdown file containing the coding standards and architectural rules for your project. This is a mandatory file.
3.  `CUSTOM-PROMPT.txt`: A text file containing the high-level system prompt for generating instructions in `feature` mode.
4.  `CUSTOM-PROMPT-LOG.txt`: A text file containing the high-level system prompt for generating instructions in `bug` mode.

## Usage

The tool is run from the root of the project you want to analyze. If you configured global execution, you can run the commands from any repository.

### To Generate Instructions for a New Feature:
This is the default mode.

```bash
# If globally configured:
agent-instructor --mode feature

# If running locally from the tool's directory:
./agent-instructor --mode feature
```

### To Generate Instructions for a Bug Fix:
Make sure the relevant error logs or bug description are present in `sdd.log`.

```bash
# If globally configured:
agent-instructor --mode bug

# If running locally from the tool's directory:
./agent-instructor --mode bug
```

After running, the output will be saved in the `INSTRUCTIONS.md` file in your project root.
