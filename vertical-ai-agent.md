---
layout: default
permalink: /vertical-ai-agent/
---
# Vertical AI Agent: A Reflective LLM Workflow
> Note: Currently in private repo and under continuous development. Feel free to ask for access at [enam.biswas.ewubd@gmail.com](mailto:enam.biswas.ewubd@gmail.com)!

The Vertical AI Agent is a sophisticated Python library for building robust LLM-powered agents. It implements a "Reflective Workflow," where a primary LLM's decisions to use tools are validated by a secondary "reflection" LLM before execution. This process enhances reliability, reduces errors, and ensures that tool usage is more accurate and contextually appropriate.

The system is designed to load tool definitions dynamically from JSON schemas, making the agent's capabilities easily extensible.

## Key Features

-   **Reflective Workflow**: A dual-LLM system (Primary + Reflection) for intelligent tool call validation and response assessment.
-   **Dynamic Tool Loading**: Easily define tools in JSON and load them dynamically from files, directories, or dictionaries.
-   **Built-in Tool Validator**: A powerful utility to validate your tool schemas for correctness and consistency before you use them.
-   **Extensible Authentication**: Supports a wide range of authentication mechanisms (API Key, Bearer, OAuth2, mTLS, HMAC, etc.) that are automatically handled by the system.
-   **Integrated Mock Tool Server**: Comes with a built-in test server that simulates all supported authentication types, allowing for seamless local testing.
-   **Multi-Provider LLM Support**: Works with major LLM providers like OpenAI, Anthropic, and Google.

## Installation

To install the package from your local repository, navigate to the project's root directory and run:

```bash
pip install .
```

Ensure you have also set up your LLM API keys as environment variables (e.g., `OPENAI_API_KEY`).

## Running the Test Tool Server

This library includes a mock tool server that simulates a real-world API with various authentication schemes. The example tools in the `tools/` directory are pre-configured to work with this server.

To start the server, run the following command in your terminal:

```bash
vai-tool-server
```

You should see output indicating the server has started, typically on port `51234`:

```
INFO - vai-tool-server - Starting mock tool server on http://localhost:51234/execute_tool
INFO - vai-tool-server - Press CTRL+C to stop the server.
```

Leave this server running in a separate terminal while you run your agent.

## Quick Start

This example demonstrates how to load all available tools, initialize the agent, and run a query.

```python
import asyncio
import os
from vai import VAIAgent, ToolLoader, VAIConfig

async def main():
    # Load all valid tools from the 'tools/' directory
    tool_loader = ToolLoader(verbose=True)
    all_tools = tool_loader.from_directory(directory_path="tools")

    # Configure the agent with your chosen LLMs
    config = VAIConfig(
        primary_llm_config={
            "provider": "openai",
            "model": "gpt-4.1-mini",
        },
        reflection_llm_config={
            "provider": "openai",
            "model": "gpt-4.1-mini",
        },
        verbose=True
    )

    # Initialize the graph builder with the configuration
    agent = VAIAgent(config)

    # Run the agent with a user query
    # The agent will use the tools to find the answer
    query = "What is 123 plus 456?"
    result_state = await agent.run(
        user_message=query, 
        available_tools=all_tools
    )

    # Print the final response from the assistant
    final_response = result_state['messages'][-1].content
    print(f"Final Answer: {final_response}")

if __name__ == "__main__":
    # Ensure API keys are set, e.g., os.environ["OPENAI_API_KEY"] = "sk-..."
    asyncio.run(main())

# Expected Output:
# Final Answer: The sum of 123 and 456 is 579.
```

## Usage Guide (A Deeper Dive)

### 1. Defining a Tool

Tools are defined in a JSON schema. This decouples the tool's interface from its implementation. Here is an example of `tools/add_numbers.json`:

```json
{
    "version": "1.0.0",
    "type": "function",
    "function": {
        "name": "add_numbers",
        "description": "Add two numbers together.",
        "parameters": {
            "type": "object",
            "properties": {
                "a": { "type": "number", "description": "First number" },
                "b": { "type": "number", "description": "Second number" }
            },
            "required": ["a", "b"]
        }
    },
    "authentication": {
        "type": "mtls_simulated",
        /* ... auth details ... */
    },
    "api": {
        "endpoint": "http://localhost:51234/execute_tool",
        "method": "POST"
    }
}
```

### 2. Validating Tools

Before loading tools, you can validate them using the `ToolValidator`. This helps catch errors in your schemas early. This example is taken from `test.ipynb`.

```python
from vai import ToolValidator

# Initialize the validator
tool_validator = ToolValidator(verbose=True)

# Validate all tools in a directory
validation_results = tool_validator.validate_directory(directory_path="tools")

print(validation_results)
# Output shows a dictionary with errors and warnings for each file.
# {'tools\\add_numbers.json': {'errors': [], 'warnings': []}, ...}
```

### 3. Loading Tools

The `ToolLoader` can load tools from various sources.

```python
from vai import ToolLoader

tool_loader = ToolLoader()

# From a directory (most common)
tools_from_dir = tool_loader.from_directory(directory_path="tools")

# From a single file
add_tool = tool_loader.from_file(file_path="tools/add_numbers.json")

# From a list of dictionary definitions
# all_tool_definitions = [ ... list of dicts ... ]
# tools_from_list = tool_loader.from_list(all_tool_definitions)
```

### 4. Configuring and Running the Agent

The `VAIAgent` is the main entry point for running the agent. It is highly configurable, allowing you to specify different LLMs for the primary and reflection roles, set recursion limits, and control verbosity.

## Core Concepts

-   **Primary LLM**: The main reasoning engine. It analyzes the user query, maintains the conversation, and proposes tool calls to solve the user's request.
-   **Reflection LLM**: A "supervisor" LLM. It validates the Primary LLM's proposed tool calls against the tool's schema and a set of operational guidelines. It can also be used to assess the quality of the final textual response before it's sent to the user. This crucial step reduces hallucinations and improves the agent's reliability.
-   **State Management**: The entire workflow is stateful, managed within a `AgentState` object. This object tracks messages, tool calls, and reflection history, and can be serialized to persist conversations.
-   **Tool Execution**: When a tool call is validated, the system sends a standard HTTP request to the endpoint defined in the tool's JSON file, automatically handling the specified authentication.
