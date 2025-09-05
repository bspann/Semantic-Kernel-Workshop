# HOL - AI Scrum Team with Semantic Kernel (Python)

> **Note**: This hands-on lab has been updated to use the latest Semantic Kernel Python requirements and API structures as of August 2025. The code examples reflect the current recommended practices and package versions.

## Overview
In this hands-on lab, you'll learn how to build an AI-first Scrum team using Microsoft Semantic Kernel's Group Chat Orchestration pattern. This system simulates a collaborative Scrum team that transforms raw requirements into production-ready Agile artifacts through coordinated AI agents.

## What You'll Build
- **Product Owner Agent**: Generates System Requirements Specification (SRS) from raw requirements
- **Business Analyst Agent**: Breaks SRS into detailed User Stories & Acceptance Criteria
- **Solution Architect Agent**: Designs solution architecture and identifies technical dependencies
- **QA Tester Agent**: Writes comprehensive test scenarios and test cases in Gherkin syntax
- **Custom Group Chat Manager**: Controls conversation flow, turn-taking, and termination logic

## Learning Objectives
By the end of this lab, you will:
- Understand Semantic Kernel's Group Chat Orchestration pattern
- Create specialized AI agents with role-specific instructions
- Implement custom conversation management logic
- Build a complete AI Scrum team workflow: Raw Requirements → SRS → User Stories → Architecture → Test Cases
- Handle agent coordination and output management

## Prerequisites
- Python 3.10 or higher
- Basic understanding of async/await patterns
- Familiarity with Agile/Scrum methodologies
- Azure OpenAI account with deployment

## Technology Stack
- **Semantic Kernel Python**: Multi-agent orchestration framework
- **Azure OpenAI**: AI service for ChatCompletionAgent
- **Python asyncio**: Asynchronous runtime support (built-in with Python 3.10+)
- **python-dotenv**: Environment variable management

## Step 1: Environment Setup

> **Important**: This lab requires Semantic Kernel Python 1.31.0 or higher. The API structure and imports have been updated to reflect the latest recommended patterns from Microsoft's official documentation.

### Install Dependencies
Create a new Python project and install the required packages:

```bash
pip install semantic-kernel[azure] python-dotenv
```

### Environment Configuration
Create a `.env` file with your Azure OpenAI configuration:

```env
AZURE_OPENAI_ENDPOINT=https://your-resource.openai.azure.us/
AZURE_OPENAI_KEY=your-api-key
MODEL_NAME=gpt-4o
```

### Project Structure
Create the following folder structure:

```text
ai_scrum_team/
├── agents/
│   ├── __init__.py
│   ├── product_owner.py
│   ├── business_analyst.py
│   ├── solution_architect.py
│   └── qa_agent.py
├── manager/
│   ├── __init__.py
│   └── scrum_group_chat_manager.py
├── runtime/
│   ├── __init__.py
│   └── run_scrum_team.py
├── .env
└── requirements.txt
```

## Step 2: Create the Product Owner Agent

Create `agents/product_owner.py`:

```python
from semantic_kernel.agents import ChatCompletionAgent

def create_product_owner_agent(kernel):
    return ChatCompletionAgent(
        kernel=kernel,
        name="ProductOwner",
        description="Generates a detailed System Requirements Specification (SRS) from raw requirements.",
        instructions=(
            "You are an experienced Product Owner. "
            "Your job is to transform raw, high-level business requirements into a clear, structured System Requirements Specification (SRS). "
            "The SRS should include a project overview, detailed features, constraints, assumptions, and any business rules. "
            "Structure your response with clear sections for Project Overview, Business Requirements, "
            "System Requirements (both functional and non-functional), Constraints, Assumptions, and Business Rules."
        )
    )
```

## Step 3: Create the Business Analyst Agent

Create `agents/business_analyst.py`:

```python
from semantic_kernel.agents import ChatCompletionAgent

def create_business_analyst_agent(kernel):
    return ChatCompletionAgent(
        kernel=kernel,
        name="BusinessAnalyst",
        description="Breaks the SRS into detailed user stories with acceptance criteria in Given/When/Then format.",
        instructions=(
            "You are a professional Business Analyst. "
            "You take a System Requirements Specification (SRS) and break it down into Agile user stories. "
            "For each story, write acceptance criteria in Given/When/Then format. "
            "Ensure the stories are clear, small, and testable. "
            "Organize stories into logical Epics and use the standard format: "
            "'As a [user], I want [goal] so that [benefit]'. "
            "Include comprehensive acceptance criteria with multiple scenarios for each story."
        )
    )
```

## Step 4: Create the Solution Architect Agent

Create `agents/solution_architect.py`:

```python
from semantic_kernel.agents import ChatCompletionAgent

def create_solution_architect_agent(kernel):
    return ChatCompletionAgent(
        kernel=kernel
        name="SolutionArchitect",
        description="Designs the overall solution architecture and identifies technical dependencies.",
        instructions=(
            "You are a senior Solution Architect. "
            "You review the user stories and design a high-level solution architecture. "
            "Provide an overview of major components, system interactions, and infrastructure needs. "
            "Identify any technical dependencies, risks, or new technical user stories required to deliver the solution. "
            "Include sections for: Major Components, System Interactions, Infrastructure Needs, "
            "Technical Dependencies and Risks, and Additional Technical User Stories. "
            "Focus on scalability, security, and maintainability."
        )
    )
```

## Step 5: Create the QA Tester Agent

Create `agents/qa_agent.py`:

```python
from semantic_kernel.agents import ChatCompletionAgent

def create_qa_agent(kernel):
    return ChatCompletionAgent(
        kernel=kernel
        name="QATester",
        description="Generates test scenarios and test cases in Gherkin format for each user story.",
        instructions=(
            "You are a detail-oriented QA Engineer. "
            "For each user story and its acceptance criteria, write clear test scenarios and test cases in Gherkin syntax. "
            "Use Given/When/Then statements to cover both positive and edge cases. "
            "Organize test cases by feature and include comprehensive scenarios for: "
            "- Happy path scenarios "
            "- Error handling scenarios "
            "- Edge cases and boundary conditions "
            "- Security and validation scenarios "
            "After all test cases are written and you have nothing else to add, "
            "conclude your response with the phrase: 'Test cases complete'. "
            "Do not keep asking how you can assist further. "
            "Always stop when done."
        )
    )
```

## Step 6: Create the Custom Group Chat Manager

Create `manager/scrum_group_chat_manager.py`:

```python
from semantic_kernel.agents import GroupChatManager, StringResult, BooleanResult, MessageResult
from semantic_kernel.contents import ChatMessageContent, ChatHistory

class ScrumGroupChatManager(GroupChatManager):
    """
    Custom Group Chat Manager that controls the sequential flow of the Scrum team:
    ProductOwner → BusinessAnalyst → SolutionArchitect → QATester
    """
    
    async def select_next_agent(self, chat_history: ChatHistory, participant_descriptions: dict[str, str]) -> StringResult:
        
        last_message = chat_history.messages[-1]
        
        if last_message.role == "user":
            return StringResult(result="ProductOwner", reason="Start with PO to generate SRS.")
        elif last_message.name == "ProductOwner":
            return StringResult(result="BusinessAnalyst", reason="BA should create user stories next.")
        elif last_message.name == "BusinessAnalyst":
            return StringResult(result="SolutionArchitect", reason="SA should design architecture next.")
        elif last_message.name == "SolutionArchitect":
            return StringResult(result="QATester", reason="QA should write test cases last.")
        elif last_message.name == "QATester":
            return StringResult(result="QATester", reason="QA finalizes output.")
        else:
            return StringResult(result="ProductOwner", reason="Fallback to PO.")

    async def should_terminate(self, chat_history: ChatHistory) -> BooleanResult:
        """
        Determines when the conversation should end.
        Terminates when QA confirms completion with "Test cases complete".
        """
        for message in reversed(chat_history.messages):
            if message.name == "QATester" and "Test cases complete" in message.content:
                return BooleanResult(result=True, reason="QA confirmed completion.")
        
        return BooleanResult(result=False, reason="Still in progress.")

    async def should_request_user_input(self, chat_history: ChatHistory) -> BooleanResult:
        """
        Determines if user input is needed. This automated workflow doesn't require user input.
        """
        return BooleanResult(result=False, reason="No human input needed for automated workflow.")

    async def filter_results(self, chat_history: ChatHistory) -> MessageResult:
        """
        Formats the final output with all deliverables from each agent.
        """
        output = "# 📋 Scrum AI Team Deliverables\n\n"
        
        for message in chat_history.messages:
            output += f"## {message.name}\n{message.content}\n\n"
        
        return MessageResult(
            result=ChatMessageContent(role="assistant", content=output),
            reason="All deliverables compiled successfully."
        )
```

## Step 7: Create the Runtime Orchestrator

Create `runtime/run_scrum_team.py`:

```python
import asyncio
import sys
import os

from dotenv import load_dotenv

# Add the parent directory to the Python path so we can import from agents and manager
sys.path.append(os.path.dirname(os.path.dirname(os.path.abspath(__file__))))

from semantic_kernel import Kernel
from semantic_kernel.agents import AgentGroupChat
from semantic_kernel.agents.strategies.selection.sequential_selection_strategy import SequentialSelectionStrategy
from semantic_kernel.agents.strategies.termination.default_termination_strategy import DefaultTerminationStrategy
from semantic_kernel.contents import ChatMessageContent, AuthorRole
from semantic_kernel.connectors.ai.open_ai import AzureChatCompletion

from agents.product_owner import create_product_owner_agent
from agents.business_analyst import create_business_analyst_agent
from agents.solution_architect import create_solution_architect_agent
from agents.qa_agent import create_qa_agent

load_dotenv()

# Get configuration
endpoint = os.getenv("AZURE_OPENAI_ENDPOINT")
api_key = os.getenv("AZURE_OPENAI_KEY")
deployment_name = os.getenv("MODEL_NAME")

async def main():
    kernel = Kernel()
    service = AzureChatCompletion(
        deployment_name=deployment_name,
        api_key=api_key,
        endpoint=endpoint,
        api_version="2024-12-01-preview"
    )
    kernel.add_service(service)

    # Create agents using the latest API, preserving prompts/instructions
    product_owner = create_product_owner_agent(kernel=kernel)
    business_analyst = create_business_analyst_agent(kernel=kernel)
    solution_architect = create_solution_architect_agent(kernel=kernel)
    qa_agent = create_qa_agent(kernel=kernel)

    # Use custom manager for agent selection/termination
    chat = AgentGroupChat(
        agents=[product_owner, business_analyst, solution_architect, qa_agent],
        selection_strategy=SequentialSelectionStrategy(),
        termination_strategy=DefaultTerminationStrategy(maximum_iterations=15)
    )

    task = (
        "We need a system that lets customers order groceries online, "
        "select same-day delivery, pay by credit card or wallet, "
        "and use promo codes. The system must support secure user registration and order tracking."
    )

    print("🚀 Starting Scrum Team Collaboration...\n")
    print(f"Task: {task}\n")
    print("=" * 80 + "\n")

    await chat.add_chat_message(
        ChatMessageContent(
            role=AuthorRole.USER,
            content=task
        )
    )
    
    # Collect history as we process the chat
    history = []

    async for response in chat.invoke():
        if response.content:
            print(f"\n**{response.name}**")
            print(f"{response.content}\n")
            print("-" * 40)
            # Collect messages as we go
            history.append(response)

    # Alternative way to get history if the chat stores it internally
    # If chat has a history property, use it:
    # history = chat.history if hasattr(chat, 'history') else history

    with open("scrum_output.md", "w", encoding="utf-8") as f:
        f.write("# Scrum Team Deliverables\n\n")
        f.write(f"## Task\n{task}\n\n")
        for message in history:
            if message.name:
                f.write(f"\n## {message.name}\n\n")
                f.write(f"{message.content}\n\n")

    print("\n✅ Deliverables saved to scrum_output.md\n")

if __name__ == "__main__":
    asyncio.run(main())
```

## Step 8: Add Agent Initialization Files

Create `agents/__init__.py`:

```python
from .product_owner import create_product_owner_agent
from .business_analyst import create_business_analyst_agent  
from .solution_architect import create_solution_architect_agent
from .qa_agent import create_qa_agent

__all__ = [
    "create_product_owner_agent",
    "create_business_analyst_agent", 
    "create_solution_architect_agent",
    "create_qa_agent"
]
```

Create `manager/__init__.py`:

```python
from .scrum_group_chat_manager import ScrumGroupChatManager

__all__ = ["ScrumGroupChatManager"]
```

Create `runtime/__init__.py`:

```python
from .run_scrum_team import run_ai_scrum_team, main

__all__ = ["run_ai_scrum_team", "main"]
```

## Step 9: Create Requirements File

Create `requirements.txt`:

```txt
semantic-kernel[azure]>=1.31.0
python-dotenv>=1.0.0
```

## Step 10: Testing Your AI Scrum Team

### Basic Test
Run your AI Scrum Team with the sample requirements:

```bash
cd ai_scrum_team
python runtime/run_scrum_team.py
```

### Custom Requirements Test
Create a `test_requirements.py` file:

```python
import asyncio
from runtime.run_scrum_team import run_ai_scrum_team

async def test_custom_requirements():
    custom_requirements = """
    Build a modern e-learning platform that allows:
    - Students to enroll in courses and track progress
    - Instructors to create and manage course content
    - Interactive video lessons with quizzes
    - Discussion forums for each course
    - Mobile app support for offline learning
    - Integration with popular payment gateways
    - Multi-language support
    - Advanced analytics for learning outcomes
    """
    
    await run_ai_scrum_team(custom_requirements, "elearning_deliverables.md")

if __name__ == "__main__":
    asyncio.run(test_custom_requirements())
```

## Expected Output Structure

When you run the AI Scrum Team, you'll get a comprehensive markdown file with:

### 1. Product Owner Section
- **System Requirements Specification (SRS)**
- Project Overview with clear system purpose
- Business Requirements (BR1, BR2, etc.)
- Functional and Non-Functional Requirements
- Constraints, Assumptions, and Business Rules

### 2. Business Analyst Section
- **User Stories and Acceptance Criteria**
- Stories organized into logical Epics
- "As a...I want...So that..." format
- Given/When/Then acceptance criteria
- Multiple scenarios per story

### 3. Solution Architect Section
- **High-Level Solution Architecture**
- Major Components (Frontend, Backend, Database, etc.)
- System Interactions and User Journeys
- Infrastructure Needs and Technology Recommendations
- Technical Dependencies and Risk Analysis
- Additional Technical User Stories

### 4. QA Tester Section
- **Test Scenarios in Gherkin Syntax**
- Organized by Feature
- Comprehensive scenarios covering:
  - Happy path testing
  - Error handling
  - Edge cases
  - Security validation
- Concludes with "Test cases complete"

## Advanced Customizations

### Custom Agent Instructions
You can customize agent behavior by modifying their instructions:

```python
def create_custom_product_owner_agent():
    return ChatCompletionAgent(
        name="ProductOwner",
        description="Specialized for fintech requirements",
        instructions=(
            "You are a Product Owner specializing in financial technology. "
            "Focus on regulatory compliance, security standards like PCI DSS, "
            "and financial industry best practices when creating SRS documents."
        ),
        service=AzureChatCompletion(
            deployment_name=model,
            api_key=api_key,
            base_url=endpoint
        )
    )
```

### Enhanced Group Chat Manager
Add more sophisticated conversation control:

```python
class EnhancedScrumGroupChatManager(ScrumGroupChatManager):
    def __init__(self, max_rounds=15, enable_reviews=True):
        super().__init__(max_rounds)
        self.enable_reviews = enable_reviews
        self.review_cycle = 0
    
    async def select_next_agent(self, chat_history: ChatHistory, participant_descriptions: dict[str, str]) -> StringResult:
        # Add review cycles where agents can comment on each other's work
        if self.enable_reviews and self.review_cycle < 2:
            # Implement review logic
            pass
        
        return await super().select_next_agent(chat_history, participant_descriptions)
```

### Multiple Output Formats
Add support for different output formats:

```python
async def run_ai_scrum_team_with_formats(requirements: str, formats=["markdown", "json", "html"]):
    """
    Generate deliverables in multiple formats
    """
    result = await run_ai_scrum_team(requirements)
    
    if "json" in formats:
        # Convert to structured JSON
        pass
    
    if "html" in formats:
        # Generate HTML report
        pass
    
    return result
```

## Troubleshooting

### Common Issues

1. **Azure OpenAI Connection Issues**
```python
# Test your connection
import os
from dotenv import load_dotenv
from semantic_kernel.connectors.ai.open_ai import AzureChatCompletion

load_dotenv()
try:
    service = AzureChatCompletion(
        deployment_name=os.getenv("MODEL_NAME"),
        api_key=os.getenv("AZURE_OPENAI_KEY"),
        base_url=os.getenv("AZURE_OPENAI_ENDPOINT")
    )
    print("✅ Azure OpenAI connection successful")
except Exception as e:
    print(f"❌ Connection failed: {e}")
```

2. **Agent Not Responding**
- Check agent instructions for clarity
- Verify the GroupChatManager logic
- Ensure proper async/await usage

3. **Conversation Not Terminating**
- Verify QA agent outputs "Test cases complete"
- Check termination logic in GroupChatManager
- Add max_rounds limit as safety

### Debug Mode
Enable detailed logging:

```python
import logging
logging.basicConfig(level=logging.DEBUG)

# Add debug prints in your manager
async def select_next_agent(self, chat_history, participant_descriptions):
    result = # ... your logic
    print(f"DEBUG: Selected next agent: {result.result}, Reason: {result.reason}")
    return result
```

## Best Practices

### 1. Agent Design
- Keep instructions clear and specific
- Define expected output formats
- Include examples in complex instructions
- Set clear termination conditions

### 2. Error Handling
```python
async def robust_run_scrum_team(requirements: str):
    try:
        return await run_ai_scrum_team(requirements)
    except Exception as e:
        print(f"Workflow failed: {e}")
        # Implement retry logic or fallback
        return None
```

### 3. Resource Management
```python
async def run_with_timeout(requirements: str, timeout_seconds=300):
    try:
        return await asyncio.wait_for(
            run_ai_scrum_team(requirements), 
            timeout=timeout_seconds
        )
    except asyncio.TimeoutError:
        print("Workflow timed out")
        return None
```

### 4. Output Validation
```python
def validate_deliverables(output: str) -> bool:
    """Validate that all expected sections are present"""
    required_sections = ["ProductOwner", "BusinessAnalyst", "SolutionArchitect", "QATester"]
    return all(section in output for section in required_sections)
```

## Conclusion

You've successfully built an AI Scrum Team using Semantic Kernel's Group Chat Orchestration pattern! This system demonstrates:

- **Multi-agent coordination** with sequential workflow
- **Role-based AI agents** with specialized instructions  
- **Custom conversation management** for complex workflows
- **Automated deliverable generation** from requirements to test cases

### Key Updates in This Version
- **Updated to Semantic Kernel Python 1.31.0+**: Leveraging the latest stable APIs and improvements
- **Python 3.10+ requirement**: Ensuring compatibility with modern Python features and performance optimizations
- **Simplified dependencies**: Removed deprecated packages like `asyncio-utils` that are no longer needed
- **Current import structure**: Updated imports to reflect the latest Semantic Kernel module organization

### Next Steps
- Experiment with different agent personalities and instructions
- Add more sophisticated review and iteration cycles
- Integrate with external tools (JIRA, Azure DevOps, etc.)
- Implement parallel workflows for different workstreams
- Add validation and quality checks for deliverables

### Key Takeaways
- Group Chat Orchestration enables complex multi-agent workflows
- Custom managers provide fine-grained control over conversation flow
- Well-designed agent instructions are crucial for quality output
- Async/await patterns enable efficient agent coordination
- Structured output makes deliverables actionable for development teams

This AI Scrum Team can significantly accelerate your project planning and requirements analysis phases, providing comprehensive Agile artifacts from high-level business requirements.
