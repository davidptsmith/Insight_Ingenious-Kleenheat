# Usage Guide

This guide shows how to use Insight Ingenious for various tasks.

## Getting Started

### Prerequisites

- Python 3.13 or higher
- [uv](https://docs.astral.sh/uv/) for Python package management

### Installation

```bash
# Clone the repository
git clone https://github.com/Insight-Services-APAC/Insight_Ingenious.git
cd Insight_Ingenious

# Install dependencies using uv
uv sync

# Initialize project (creates config templates and folder structure)
uv run ingen initialize-new-project
```

### Basic Configuration

1. Edit `config.yml` in your project directory
2. Create or edit `profiles.yml` in `~/.ingenious/`
3. Set environment variables:
   ```bash
   export INGENIOUS_PROJECT_PATH=/path/to/your/project/config.yml
   export INGENIOUS_PROFILE_PATH=~/.ingenious/profiles.yml
   ```

## Using the CLI

Insight Ingenious provides a command-line interface with various utilities:

### Initialize a New Project

```bash
uv run ingen initialize-new-project
```

This creates the folder structure and configuration files needed for your project.

### Run REST API Server

```bash
uv run ingen run-rest-api-server [PROJECT_DIR] [PROFILE_DIR] [HOST] [PORT] [RUN_DIR]
```

Starts the FastAPI server that presents your agent workflows via REST endpoints.

Arguments (all optional, positional):
- `PROJECT_DIR`: Path to the config file (defaults to environment variable `INGENIOUS_PROJECT_PATH`)
- `PROFILE_DIR`: Path to the profile file (defaults to `$HOME/.ingenious/profiles.yml`)
- `HOST`: Host to run on (default: `0.0.0.0`, use `127.0.0.1` for local development)
- `PORT`: Port number (default: `80`)
- `RUN_DIR`: Directory in which to launch the web server

### Run Tests

```bash
uv run ingen run-test-batch [--log-level LOG_LEVEL] [--run-args RUN_ARGS]
```

Runs tests against your agent prompts.

Options:
- `--log-level`: Controls verbosity (DEBUG, INFO, WARNING, ERROR). Default: WARNING
- `--run-args`: Key-value pairs for test runner (e.g., `--run-args='--test_name=TestName --test_type=TestType'`)

### Run Prompt Tuner

```bash
uv run ingen run-prompt-tuner
```

Starts the prompt tuner web application for fine-tuning your prompts. The application will run on the host and port specified in your configuration file.

### Data Preparation

```bash
uv run ingen dataprep [COMMAND]
```

Data-preparation utilities including the Scrapfly crawler façade. Use `uv run ingen dataprep --help` for more details.

## Using the Web UI

### Accessing the UI

Once the application is running, access the web UI at:
- http://localhost:80 - Main application (or the port specified in your config)
- http://localhost:80/chainlit - Chainlit chat interface
- http://localhost:80/prompt-tuner - Prompt tuning interface

Note: Default port is 80 as specified in config.yml. For local development, you may want to use a different port like 8000.

### Chatting with Agents

1. Navigate to http://localhost:80/chainlit (or your configured port)
2. Start a new conversation
3. Type your message
4. The appropriate agents will process your request and respond

### Tuning Prompts

1. Navigate to http://localhost:80/prompt-tuner (or your configured port)
2. Log in with credentials from your `profiles.yml`
3. Select the prompt template you want to edit
4. Make changes and test with sample data
5. Save revisions for version control

## Creating Custom Agents

### Custom Agent Structure

1. Create a new agent folder in `ingenious/services/chat_services/multi_agent/agents/your_agent_name/`
2. Create these files:
   - `agent.md`: Agent definition and persona
   - `tasks/task.md`: Task description for the agent

### Agent Definition Example

```markdown
# Your Agent Name

## Name and Persona

* Name: Your name is Ingenious and you are a [Specialty] Expert
* Description: You are a [specialty] expert assistant. Your role is to [description of responsibilities].

## System Message

### Backstory

[Background information about the agent's role and knowledge]

### Instructions

[Detailed instructions on how the agent should operate]

### Examples

[Example interactions or outputs]
```

## Creating Custom Conversation Patterns

### Pattern Structure

1. Create a new pattern module in `ingenious/services/chat_services/multi_agent/conversation_patterns/your_pattern_name/`
2. Implement the `ConversationPattern` class following the interface
3. Create a corresponding flow in `ingenious/services/chat_services/multi_agent/conversation_flows/your_pattern_name/`

### Pattern Implementation Example

```python
# conversation_patterns/your_pattern_name/your_pattern_name.py
import autogen
import logging

class ConversationPattern:
    def __init__(self, default_llm_config: dict, topics: list, memory_record_switch: bool, memory_path: str, thread_memory: str):
        self.default_llm_config = default_llm_config
        self.topics = topics
        self.memory_record_switch = memory_record_switch
        self.memory_path = memory_path
        self.thread_memory = thread_memory

        # Initialize agents
        self.user_proxy = autogen.UserProxyAgent(
            name="user_proxy",
            human_input_mode="NEVER",
            system_message="I represent the user's request"
        )

        self.your_agent = autogen.AssistantAgent(
            name="your_agent",
            system_message="Your agent's system message",
            llm_config=self.default_llm_config
        )

    async def get_conversation_response(self, input_message: str) -> [str, str]:
        # Set up agent interactions
        result = await self.user_proxy.a_initiate_chat(
            self.your_agent,
            message=input_message
        )

        return result.summary, ""
```

### Flow Implementation Example

```python
# conversation_flows/your_pattern_name/your_pattern_name.py
from ingenious.models.chat import ChatResponse
from ingenious.services.chat_services.multi_agent.conversation_patterns.your_pattern_name.your_pattern_name import ConversationPattern

class ConversationFlow:
    @staticmethod
    async def get_conversation_response(message: str, topics: list = [], thread_memory: str='', memory_record_switch = True, thread_chat_history: list = []) -> ChatResponse:
        # Get configuration
        import ingenious.config.config as config
        _config = config.get_config()
        llm_config = _config.models[0].__dict__
        memory_path = _config.chat_history.memory_path

        # Initialize the conversation pattern
        agent_pattern = ConversationPattern(
            default_llm_config=llm_config,
            topics=topics,
            memory_record_switch=memory_record_switch,
            memory_path=memory_path,
            thread_memory=thread_memory
        )

        # Get the conversation response
        res, memory_summary = await agent_pattern.get_conversation_response(message)

        return res, memory_summary
```

## Using Custom Templates

### Creating Custom Prompts

1. Create a new template in `templates/prompts/your_prompt_name.jinja`
2. Use Jinja2 syntax for dynamic content

Example:
```jinja
I am an agent tasked with providing insights about {{ topic }} based on the input payload.

User input: {{ user_input }}

Please provide a detailed analysis focusing on:
1. Key facts
2. Relevant context
3. Potential implications

Response format:
{
  "analysis": {
    "key_facts": [],
    "context": "",
    "implications": []
  },
  "recommendation": ""
}
```

### Using Templates in Agents

```python
async def get_conversation_response(self, chat_request: ChatRequest) -> ChatResponse:
    # Load the template
    template_content = await self.Get_Template(file_name="your_prompt_name.jinja")

    # Render the template with dynamic values
    env = Environment()
    template = env.from_string(template_content)
    rendered_prompt = template.render(
        topic="example topic",
        user_input=chat_request.user_prompt
    )

    # Use the rendered prompt
    your_agent.system_prompt = rendered_prompt
```

## API Integration

### Using the REST API

You can interact with Insight Ingenious through its REST API:

```bash
# Start a conversation
curl -X POST http://localhost:80/api/v1/chat \
  -H "Content-Type: application/json" \
  -H "Authorization: Basic $(echo -n username:password | base64)" \
  -d '{
    "user_prompt": "Your message here",
    "conversation_flow": "classification_agent"
  }'
```

### Creating Custom API Routes

1. Create a new route module in `ingenious_extensions_template/api/routes/custom.py`
2. Implement the `Api_Routes` class

Example:
```python
from fastapi import APIRouter, Depends, FastAPI
from ingenious.models.api_routes import IApiRoutes
from ingenious.models.config import Config

class Api_Routes(IApiRoutes):
    def __init__(self, config: Config, app: FastAPI):
        self.config = config
        self.app = app
        self.router = APIRouter()

    def add_custom_routes(self):
        @self.router.get("/api/v1/custom-endpoint")
        async def custom_endpoint():
            return {"message": "Custom endpoint response"}

        self.app.include_router(self.router)
```
