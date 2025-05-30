# Browser Use: A Deep Dive for Contributors

Welcome to the `browser_use` codebase! This document provides a comprehensive guide for developers looking to understand its architecture, core concepts, and how to contribute effectively. `browser_use` is a powerful Python library designed to enable AI agents to programmatically control web browsers, primarily using Playwright for robust and modern browser automation.

## 1. Core Architecture Overview

The `browser_use` repository is a critical component for programmatic browser interaction, enabling automated tasks, data extraction, and web application testing. It's built with modularity, extensibility, and reliability in mind. (Reference: `Codebase Overview: browser_use`)

The architecture follows a layered approach:

*   **Agent (`browser_use/agent`)**: The high-level orchestrator. It takes a task, interacts with an LLM to decide on actions, manages state, and drives the browser interaction loop.
*   **Controller (`browser_use/controller`)**: Defines and executes the specific actions an agent can perform (e.g., click, type, navigate, extract content). It acts as an interface between the agent's decisions and the browser's capabilities.
*   **Browser (`browser_use/browser`)**: Manages the underlying Playwright browser instance, contexts, pages, profiles, and session state. It abstracts away the direct complexities of Playwright.
*   **DOM (`browser_use/dom`)**: Responsible for processing the Document Object Model (DOM) of web pages. It extracts a simplified, LLM-friendly representation of the page, identifies interactive elements, and provides mechanisms for the agent to understand and interact with page content.
*   **LLM Integration**: While not a separate module, LLM interaction is central. The agent uses an LLM (e.g., from `langchain_openai`, `langchain_anthropic`) to interpret tasks, understand page content, and decide on next actions.

The entire codebase is heavily asynchronous, leveraging Python's `async` and `await` keywords for non-blocking browser operations, which is essential for efficient web automation.

## 2. Key Modules and Their Roles

Let's break down the most important directories and files:

### `browser_use/agent/`

*   **`service.py` (Agent class)**: This is the central nervous system of an agent.
    *   Manages the main execution loop (`run`, `step`).
    *   Interfaces with the LLM to get the next action (`get_next_action`).
    *   Maintains the agent's state (`AgentState`), including history, failures, and current status.
    *   Handles callbacks for external systems (e.g., `register_new_step_callback`).
    *   Orchestrates the use of vision capabilities and memory.
    *   Initializes and uses the `MessageManager` and `Controller`.
*   **`prompts.py`**: Contains Pydantic models (`SystemPrompt`, `AgentMessagePrompt`, `PlannerPrompt`) for constructing the system messages and user messages sent to the LLM. These prompts guide the LLM's behavior and provide it with context about the task, available actions, and current browser state.
*   **`message_manager/service.py` (MessageManager class)**:
    *   Manages the conversation history with the LLM.
    *   Handles token limits by truncating messages if necessary.
    *   Formats messages, including browser state summaries (DOM, screenshots), for LLM consumption.
    *   Redacts sensitive data from messages.
*   **`memory/service.py` (Memory class)**:
    *   (Activated with `browser-use[memory]` install and `enable_memory=True`)
    *   Provides the agent with long-term memory capabilities, allowing it to recall information from previous steps or similar tasks.
    *   Uses vector stores (like FAISS) and sentence transformers for semantic search over past experiences.
*   **`views.py`**: Pydantic models defining the structure of agent outputs (`AgentOutput`), history items (`AgentHistory`), and various state representations.

### `browser_use/browser/`

*   **`session.py` (BrowserSession class)**: This is a crucial class that encapsulates a Playwright browser session.
    *   **Lifecycle Management**: Handles launching new browser instances (persistent or incognito) or connecting to existing ones (via CDP, WSS, or PID). See `start()` and `stop()`.
    *   **Profile Management**: Uses `BrowserProfile` to configure browser settings (headless mode, user data directory, viewport size, proxy, cookies, `allowed_domains` for security).
    *   **Tab and Page Management**: Manages multiple tabs (`Page` objects), including creating, switching, and closing tabs. The `agent_current_page` and `human_current_page` attributes track focus.
    *   **State Abstraction**: Provides methods like `get_state_summary()` which captures the current URL, title, open tabs, a screenshot, and the processed DOM tree.
    *   **Low-level Interactions**: Wraps Playwright actions like clicks (`_click_element_node`), text input (`_input_text_element_node`), navigation, and scrolling, often adding custom logic or error handling.
    *   **DOM Interaction**: Works closely with `DomService` to get interactive elements.
*   **`profile.py` (BrowserProfile class)**: A Pydantic model defining all configurable aspects of a browser session, such as `headless`, `user_data_dir`, `storage_state` (for cookies/localStorage), `viewport`, `proxy_server`, `allowed_domains`, `cookies_file` (deprecated), etc.
*   **`dom/service.py` (DomService class)**:
    *   Uses `buildDomTree.js` (a JavaScript file executed in the browser) to extract a structured representation of the DOM.
    *   Identifies interactive elements (buttons, links, inputs) and assigns them unique indices.
    *   Creates a `SelectorMap` that maps these indices to DOM element nodes.
    *   Provides methods to get clickable elements, highlight elements, and extract cross-origin iframe information.
*   **`dom/clickable_element_processor/service.py`**: Focuses on processing and hashing clickable elements, which helps in identifying new elements on a page between agent steps.
*   **`views.py`**: Pydantic models for browser-related data structures like `BrowserStateSummary`, `TabInfo`, `URLNotAllowedError`.

### `browser_use/controller/`

*   **`service.py` (Controller class)**:
    *   Acts as a registry and executor for all actions an agent can perform.
    *   Uses a `Registry` instance to store action definitions.
    *   The `act()` method takes an `ActionModel` (decided by the LLM) and executes the corresponding Python function.
    *   Default actions include `go_to_url`, `click_element_by_index`, `input_text`, `extract_content`, `done`, `scroll_down`, etc.
    *   Supports custom actions registered via the `@controller.registry.action(...)` decorator.
*   **`registry/service.py` (Registry class)**:
    *   Manages the available actions. Each action is defined with a description (for the LLM), an optional Pydantic parameter model, and the async function to execute.
    *   Can filter actions based on the current page's domain (`domains` parameter in `@action`).
    *   Generates a Pydantic model (`ActionModel`) dynamically based on the registered actions, which is used by the LLM for structured output (tool calling).
*   **`views.py`**: Pydantic models for the parameters of each built-in action (e.g., `GoToUrlAction`, `ClickElementAction`). These ensure that the LLM provides valid arguments for actions.

### Other Important Files/Directories

*   **`browser_use/__init__.py`**: Exposes the main classes (`Agent`, `BrowserSession`, `Controller`) for easy import.
*   **`browser_use/cli.py`**: Implements the command-line interface for `browser-use`.
*   **`browser_use/utils.py`**: Contains various helper functions and utility classes used across the codebase.
*   **`browser_use/exceptions.py`**: Custom exception classes.
*   **`browser_use/logging_config.py`**: Sets up logging for the library.
*   **`pyproject.toml`**: Defines project metadata, dependencies (managed with `uv` or `pip`), build system (Hatch), and tool configurations (Ruff for linting/formatting, Pytest for testing). (Reference: `pyproject.toml`)
*   **`examples/`**: Contains a wealth of usage examples for various features, custom functions, LLM integrations, and use cases. This is an excellent resource for understanding how to use the library and for finding contribution ideas.
*   **`docs/`**: The source for the official documentation website.
*   **`tests/`**: Contains automated tests, crucial for ensuring code quality and stability.

## 3. Core Concepts

Understanding these concepts is key to working with `browser_use`:

### Agent Lifecycle

1.  **Initialization**: An `Agent` is created with a `task` (e.g., "Find the price of product X on website Y and add it to the cart") and an LLM instance.
2.  **Run Loop**: The `agent.run()` method starts the execution. It iterates in steps (`agent.step()`) up to `max_steps`.
3.  **State Update**: In each step:
    *   The `BrowserSession` captures the current page's state (`get_state_summary()`), including URL, title, a simplified DOM tree with interactive elements indexed, and a screenshot.
4.  **LLM Interaction**:
    *   The `MessageManager` constructs a prompt for the LLM, including the original task, conversation history, the current browser state summary, and available actions.
    *   The `Agent` calls the LLM (`get_next_action()`) to get the next action(s) to perform. The LLM's response is expected to be a JSON object matching the `AgentOutput` Pydantic model, which includes the chosen action and its parameters.
5.  **Action Execution**:
    *   The `Controller` takes the `ActionModel` from the LLM's response.
    *   It validates the parameters against the action's Pydantic model.
    *   It executes the corresponding async function, which interacts with the `BrowserSession`.
6.  **Result Processing**: The result of the action (e.g., "navigated to URL", "clicked button X", extracted text) is captured.
7.  **History**: The browser state, LLM output, and action result are stored in `AgentHistoryList`.
8.  **Termination**: The loop continues until:
    *   The LLM chooses the `done` action.
    *   `max_steps` is reached.
    *   `max_failures` is reached.
    *   The agent is manually stopped.

### DOM Representation and Interaction

*   **`buildDomTree.js`**: This JavaScript code runs in the browser to traverse the DOM. It identifies visible, interactive elements, extracts their attributes (tag, text, role, aria-labels, etc.), and builds a tree structure.
*   **Element Indexing**: Interactive elements are assigned a unique `highlight_index`. The LLM refers to elements using these indices (e.g., "click element with index 5").
*   **`SelectorMap`**: A mapping from `highlight_index` to the `DOMElementNode` object, allowing the `Controller` to locate the actual Playwright `ElementHandle` for interaction.
*   **Vision (`use_vision=True`)**: If enabled, a screenshot of the page is sent to a vision-capable LLM along with the DOM text, allowing the LLM to "see" the page layout and make more informed decisions.

### Asynchronous Nature

*   Nearly all browser operations and LLM calls are I/O-bound. `asyncio` with `async/await` is used extensively to perform these operations concurrently without blocking the main thread, leading to better performance.
*   If you're new to `asyncio`, understanding concepts like event loops, coroutines, and tasks will be beneficial.

### Tool Calling / Function Calling

*   `browser_use` leverages the "tool calling" or "function calling" capabilities of modern LLMs.
*   The `Controller`'s `Registry` dynamically creates a Pydantic model (`ActionModel`) representing all available actions and their expected parameters.
*   This model is provided to the LLM, which then formats its output as a JSON object specifying the action to call and the arguments to use. This structured approach is more reliable than prompting the LLM to generate raw commands.
*   The `tool_calling_method` in `AgentSettings` can be 'auto', 'function_calling', 'tools', 'json_mode', or 'raw', depending on the LLM's capabilities. The agent attempts to auto-detect the best method.

## 4. Development Setup

(Reference: `docs/development/contribution-guide.mdx`, `pyproject.toml`)

1.  **Clone the Repository**:
    ```bash
    git clone https://github.com/porihacker/browser_use 
    # Or your fork: git clone https://github.com/YOUR_USERNAME/browser_use
    cd browser-use
    ```
2.  **Set up Python Environment**:
    *   `browser_use` requires Python >=3.11.
    *   It uses `uv` for fast dependency management (or `pip`).
    ```bash
    uv sync --all-extras --dev
    ```
    This installs all main dependencies, optional extras (like `memory`, `cli`), and development dependencies (like `pytest`, `ruff`).
3.  **Install Playwright Browsers**:
    ```bash
    playwright install chromium --with-deps
    # Or other browsers like firefox, webkit
    ```
4.  **Environment Variables**:
    *   Create a `.env` file in the root of the project.
    *   Add your LLM API keys (e.g., `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`).
    *   For detailed debugging, add:
        ```
        BROWSER_USE_LOGGING_LEVEL=debug
        ```
5.  **Pre-commit Hooks**:
    *   The project uses `pre-commit` for code quality checks (linting with Ruff, codespell).
    *   Install hooks:
        ```bash
        pre-commit install
        ```
    This will run checks automatically before each commit.

## 5. Testing

(Reference: `tests/`, `pyproject.toml` under `[tool.pytest.ini_options]`)

*   Tests are located in the `tests/` directory.
*   `pytest` is the test runner.
*   Key test files:
    *   `tests/test_core_functionality.py`: Tests fundamental agent actions.
    *   `tests/ci/`: Contains more specific integration tests for browser session features, controller actions, etc.
*   To run tests:
    ```bash
    # Run all tests
    pytest
    
    # Run specific file
    pytest tests/test_core_functionality.py
    
    # Run tests with a specific marker (e.g., unit)
    pytest -m unit
    ```
*   Tests often use `pytest-asyncio` for testing asynchronous code.
*   `pytest-httpserver` is used to mock HTTP responses for controlled testing of navigation and data extraction.
*   **Writing tests for new contributions is crucial.**

## 6. How to Contribute

(Reference: `CONTRIBUTING.md`, `docs/development/contribution-guide.mdx`)

### Finding Something to Work On

*   **GitHub Issues**: Look for issues tagged `good-first-issue` or `help-wanted` on the [official browser-use repository issues page](https://github.com/browser-use/browser-use/issues). (Note: The provided context URL `https://github.com/porihacker/browser_use` is likely a fork; contributions should generally go to the main upstream repository).
*   **Roadmap**: The `README.md` outlines a roadmap with areas like agent memory, planning, DOM extraction, and UX improvements.
*   **Discord**: Join the community Discord (link in `README.md`) to discuss ideas.
*   **Examples**: Extend existing examples or create new ones for novel use cases.

### Contribution Process

1.  **Fork the Repository**: Fork the main `browser-use/browser-use` repository to your GitHub account.
2.  **Clone Your Fork**:
    ```bash
    git clone https://github.com/YOUR_USERNAME/browser-use.git
    cd browser-use
    ```
3.  **Create a Branch**:
    ```bash
    git checkout -b my-feature-or-bugfix
    ```
4.  **Make Changes**: Implement your feature or fix the bug.
5.  **Add Tests**: Write `pytest` tests to cover your changes.
6.  **Run Linters/Formatters**:
    ```bash
    ruff check . --fix
    ruff format .
    # Or rely on pre-commit hooks
    ```
7.  **Commit Changes**:
    ```bash
    git add .
    git commit -m "feat: Implement X feature" 
    # Or "fix: Resolve Y bug", "docs: Update Z"
    ```
8.  **Push to Your Fork**:
    ```bash
    git push origin my-feature-or-bugfix
    ```
9.  **Submit a Pull Request (PR)**:
    *   Go to the original `browser-use/browser-use` repository on GitHub.
    *   You should see a prompt to create a PR from your new branch.
    *   Provide a clear description of your changes, why they are needed, and how they were tested. Include screenshots/GIFs if applicable.
    *   Ensure all CI checks pass.
10. **Code Review**: Respond to feedback from maintainers.

### Coding Standards

*   **Ruff**: Used for linting and formatting. Configuration is in `pyproject.toml`.
*   **Type Hinting**: The codebase uses Python type hints extensively. Ensure your contributions are also well-typed.
*   **Docstrings**: Add clear docstrings to new functions and classes.
*   **Clarity and Readability**: Write code that is easy to understand and maintain.

## 7. Areas for Contribution (Examples)

*   **Agent Enhancements**:
    *   Improve the `PlannerPrompt` or the planning logic in `Agent.service.py`.
    *   Optimize token usage in `MessageManager` or by refining DOM summarization.
    *   Enhance the `Memory` module for more sophisticated recall or summarization.
*   **DOM & Element Interaction**:
    *   Improve the accuracy of `buildDomTree.js` or `DomService` in identifying complex UI elements (e.g., custom date pickers, shadow DOM elements).
    *   Add more robust error handling for element interactions in `BrowserSession` or `Controller` actions.
*   **New Controller Actions**:
    *   Identify common browser tasks not yet covered and add them to `controller/service.py` with corresponding Pydantic models in `controller/views.py`.
    *   Example: Handling specific types of CAPTCHAs (if ethical and feasible), interacting with browser dialogs (alerts, prompts, confirms), advanced file downloads/uploads.
*   **LLM Support & Prompting**:
    *   Add support for new LLM providers in `examples/models/` and ensure compatibility with `Agent`.
    *   Refine system prompts in `agent/prompts.py` for better reliability or new capabilities.
*   **BrowserSession Features**:
    *   Enhance proxy management or security features in `BrowserProfile` and `BrowserSession`.
    *   Improve handling of browser extensions.
*   **Testing**:
    *   Increase test coverage, especially for edge cases or complex interactions.
    *   Add end-to-end tests for common use cases found in `examples/use-cases/`.
*   **Documentation & Examples**:
    *   Improve existing documentation in the `/docs` directory.
    *   Add new examples to the `/examples` directory showcasing features or solving new problems.
*   **Bug Fixes**: Tackle existing issues from the GitHub issue tracker.

## 8. Advanced Topics

*   **Custom Functions & Hooks**: The `Agent` allows for `on_step_start` and `on_step_end` hooks. The `Controller` allows registering completely custom actions. See `examples/custom-functions/` for inspiration.
*   **Playwright Script Generation**: The agent can save its history as a Playwright script (`save_playwright_script_path` in `AgentSettings`). Understanding `agent/playwright_script_generator.py` can be useful.
*   **Sensitive Data Handling**: The agent and browser session have mechanisms for handling sensitive data, redacting it from logs and LLM prompts.
*   **Telemetry**: `browser_use/telemetry/service.py` handles optional product telemetry.

## 9. Conclusion

`browser_use` is a sophisticated project with many opportunities for impactful contributions. By understanding its architecture and core components, you can effectively extend its capabilities, improve its robustness, and help the community build powerful browser-based AI agents.

Don't hesitate to ask questions on the project's GitHub issues or Discord channel. Happy coding!
