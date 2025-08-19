# Semantic Kernel Multi-Agent Orchestration C# Hands-On Lab - Azure Government

This comprehensive hands-on lab will guide you through building advanced multi-agent systems with Semantic Kernel in C#. You'll learn to create single agents, manage conversation threads, implement sequential orchestration, and handle agent response callbacks using Azure OpenAI in Azure Government Cloud.

## Prerequisites

### Development Environment
- Visual Studio Code
- .NET 8.0 SDK or later
- C# extension for VS Code

### Required Azure OpenAI Resources
- Azure OpenAI resource deployed in Azure Government Cloud
- Azure OpenAI service endpoint (e.g., `https://your-resource.openai.usgovcloudapi.net/`)
- Azure OpenAI API key
- Deployed models: `gpt-4` or `gpt-35-turbo` (for optimal agent performance)

## Required NuGet Packages

Before starting this multi-agent lab, you'll need to install the following NuGet packages. These packages provide the agent orchestration capabilities beyond the basic Semantic Kernel framework:

| Package Name | Version | Purpose |
|--------------|---------|---------|
| `Microsoft.SemanticKernel` | Latest | Core Semantic Kernel framework and orchestration engine |
| `Microsoft.SemanticKernel.Connectors.AzureOpenAI` | Latest | Azure OpenAI integration for chat completion |
| `Microsoft.SemanticKernel.Agents.Abstractions` | Latest | Agent abstractions and interfaces |
| `Microsoft.SemanticKernel.Agents.Core` | Latest | Core agent implementation and thread management |
| `Microsoft.SemanticKernel.Agents.OpenAI` | Latest | OpenAI-compatible agent implementations |
| `Microsoft.Extensions.Logging.Console` | Latest | Console logging support for debugging |
| `Microsoft.Extensions.Configuration` | Latest | Configuration management |
| `Microsoft.Extensions.Configuration.Json` | Latest | JSON configuration provider |

**Installation Command:**
```bash
# Run this single command to install all required packages
dotnet add package Microsoft.SemanticKernel && \
dotnet add package Microsoft.SemanticKernel.Connectors.AzureOpenAI && \
dotnet add package Microsoft.SemanticKernel.Agents.Abstractions && \
dotnet add package Microsoft.SemanticKernel.Agents.Core && \
dotnet add package Microsoft.SemanticKernel.Agents.OpenAI && \
dotnet add package Microsoft.Extensions.Logging.Console && \
dotnet add package Microsoft.Extensions.Configuration && \
dotnet add package Microsoft.Extensions.Configuration.Json
```

**Alternative Individual Installation:**
```bash
dotnet add package Microsoft.SemanticKernel
dotnet add package Microsoft.SemanticKernel.Connectors.AzureOpenAI
dotnet add package Microsoft.SemanticKernel.Agents.Abstractions
dotnet add package Microsoft.SemanticKernel.Agents.Core
dotnet add package Microsoft.SemanticKernel.Agents.OpenAI
dotnet add package Microsoft.Extensions.Logging.Console
dotnet add package Microsoft.Extensions.Configuration
dotnet add package Microsoft.Extensions.Configuration.Json
```

## Step 1: Project Setup

1. **Open VS Code** and create a new folder for your multi-agent project:
   ```bash
   mkdir SemanticKernelMultiAgent
   cd SemanticKernelMultiAgent
   code .
   ```

2. **Open the integrated terminal** in VS Code (`Ctrl+`` ` or `View > Terminal`) and create a new console application:
   ```bash
   dotnet new console
   ```

3. **Install the required NuGet packages** using the installation command from the Required NuGet Packages section above.

   > **What you're doing:** These packages provide the advanced agent orchestration capabilities needed for multi-agent scenarios, including conversation thread management and sequential orchestration.

4. **Create an `appsettings.json` file** for configuration:

```json
{
  "AzureOpenAI": {
    "Endpoint": "https://your-resource.openai.usgovcloudapi.net/",
    "ApiKey": "your-azure-openai-api-key",
    "ChatDeploymentName": "gpt-4",
    "ModelId": "gpt-4"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.SemanticKernel": "Debug"
    }
  }
}
```

> **What you're doing:** This configuration file stores your Azure Government OpenAI settings. Always use environment variables or secure key vaults in production environments.

## Step 2: Create Base Agent Infrastructure

5. **Create a new file called `AgentBase.cs`** to define the base agent structure:

```csharp
using Microsoft.SemanticKernel;
using Microsoft.SemanticKernel.Agents;
using Microsoft.SemanticKernel.ChatCompletion;
using Microsoft.SemanticKernel.Connectors.AzureOpenAI;

public abstract class AgentBase
{
    protected readonly Kernel _kernel;
    protected readonly string _agentId;
    protected readonly string _name;
    protected readonly string _description;

    public string AgentId => _agentId;
    public string Name => _name;
    public string Description => _description;

    protected AgentBase(string agentId, string name, string description, Kernel kernel)
    {
        _agentId = agentId;
        _name = name;
        _description = description;
        _kernel = kernel;
    }

    public abstract Task<string> ProcessMessageAsync(string message, CancellationToken cancellationToken = default);
    
    protected virtual void LogAgentAction(string action, string details = "")
    {
        Console.WriteLine($"🤖 [{_name}] {action}");
        if (!string.IsNullOrEmpty(details))
        {
            Console.WriteLine($"   └─ {details}");
        }
    }
}
```

> **What you're doing:** This creates a base class that all your agents will inherit from, providing common functionality and a consistent interface for agent operations.

## Step 3: Create Chat Completion Agent

6. **Create a new file called `ChatCompletionAgent.cs`**:

```csharp
using Microsoft.SemanticKernel;
using Microsoft.SemanticKernel.ChatCompletion;
using Microsoft.SemanticKernel.Contents;

public class ChatCompletionAgent : AgentBase
{
    private readonly IChatCompletionService _chatService;
    private readonly ChatHistory _chatHistory;
    private readonly string _systemPrompt;

    public ChatCompletionAgent(
        string agentId,
        string name,
        string description,
        string systemPrompt,
        Kernel kernel)
        : base(agentId, name, description, kernel)
    {
        _chatService = kernel.GetRequiredService<IChatCompletionService>();
        _chatHistory = new ChatHistory();
        _systemPrompt = systemPrompt;
        
        // Initialize with system prompt
        if (!string.IsNullOrEmpty(_systemPrompt))
        {
            _chatHistory.AddSystemMessage(_systemPrompt);
        }
        
        LogAgentAction("Initialized", $"System prompt: {_systemPrompt}");
    }

    public override async Task<string> ProcessMessageAsync(string message, CancellationToken cancellationToken = default)
    {
        LogAgentAction("Processing message", message);
        
        try
        {
            // Add user message to history
            _chatHistory.AddUserMessage(message);
            
            // Get response from Azure OpenAI
            var response = await _chatService.GetChatMessageContentAsync(
                _chatHistory,
                kernel: _kernel,
                cancellationToken: cancellationToken);
            
            // Add assistant response to history
            _chatHistory.AddAssistantMessage(response.Content ?? "");
            
            LogAgentAction("Generated response", response.Content ?? "No response");
            
            return response.Content ?? "";
        }
        catch (Exception ex)
        {
            LogAgentAction("Error processing message", ex.Message);
            throw;
        }
    }
    
    public void ClearHistory()
    {
        _chatHistory.Clear();
        if (!string.IsNullOrEmpty(_systemPrompt))
        {
            _chatHistory.AddSystemMessage(_systemPrompt);
        }
        LogAgentAction("Chat history cleared");
    }
    
    public int GetMessageCount() => _chatHistory.Count;
    
    public IReadOnlyList<ChatMessageContent> GetChatHistory() => _chatHistory;
}
```

> **What you're doing:** This creates a Chat Completion Agent that maintains conversation history and can process messages using Azure OpenAI. The agent remembers the conversation context and applies a system prompt to guide its behavior.

## Step 4: Create Azure AI Agent (Enhanced Agent)

7. **Create a new file called `AzureAIAgent.cs`** for more advanced agent capabilities:

```csharp
using Microsoft.SemanticKernel;
using Microsoft.SemanticKernel.ChatCompletion;
using Microsoft.SemanticKernel.Contents;
using Microsoft.SemanticKernel.Connectors.AzureOpenAI;
using System.Text.Json;

public class AzureAIAgent : AgentBase
{
    private readonly IChatCompletionService _chatService;
    private readonly ChatHistory _chatHistory;
    private readonly string _systemPrompt;
    private readonly Dictionary<string, object> _context;
    private readonly List<AgentAction> _actionHistory;

    public AzureAIAgent(
        string agentId,
        string name,
        string description,
        string systemPrompt,
        Kernel kernel)
        : base(agentId, name, description, kernel)
    {
        _chatService = kernel.GetRequiredService<IChatCompletionService>();
        _chatHistory = new ChatHistory();
        _systemPrompt = systemPrompt;
        _context = new Dictionary<string, object>();
        _actionHistory = new List<AgentAction>();
        
        // Initialize with enhanced system prompt
        var enhancedPrompt = $"{_systemPrompt}\n\nYou are an AI agent named '{_name}' with ID '{_agentId}'. {_description}";
        _chatHistory.AddSystemMessage(enhancedPrompt);
        
        LogAgentAction("Initialized with enhanced capabilities", enhancedPrompt);
    }

    public override async Task<string> ProcessMessageAsync(string message, CancellationToken cancellationToken = default)
    {
        LogAgentAction("Processing message with context", message);
        
        var action = new AgentAction
        {
            Timestamp = DateTime.UtcNow,
            InputMessage = message,
            AgentId = _agentId
        };

        try
        {
            // Add context information to the prompt if available
            var contextualMessage = BuildContextualMessage(message);
            
            _chatHistory.AddUserMessage(contextualMessage);
            
            // Configure Azure OpenAI execution settings for Azure Government
            var executionSettings = new AzureOpenAIPromptExecutionSettings
            {
                Temperature = 0.7,
                MaxTokens = 1000,
                TopP = 0.9,
                FrequencyPenalty = 0.0,
                PresencePenalty = 0.0
            };
            
            var response = await _chatService.GetChatMessageContentAsync(
                _chatHistory,
                executionSettings,
                _kernel,
                cancellationToken);
            
            _chatHistory.AddAssistantMessage(response.Content ?? "");
            
            action.OutputMessage = response.Content ?? "";
            action.Success = true;
            
            LogAgentAction("Generated enhanced response", response.Content ?? "No response");
            
            return response.Content ?? "";
        }
        catch (Exception ex)
        {
            action.Success = false;
            action.ErrorMessage = ex.Message;
            LogAgentAction("Error in enhanced processing", ex.Message);
            throw;
        }
        finally
        {
            _actionHistory.Add(action);
        }
    }
    
    private string BuildContextualMessage(string message)
    {
        if (_context.Count == 0)
            return message;
        
        var contextJson = JsonSerializer.Serialize(_context, new JsonSerializerOptions { WriteIndented = true });
        return $"Context: {contextJson}\n\nUser Message: {message}";
    }
    
    public void AddContext(string key, object value)
    {
        _context[key] = value;
        LogAgentAction("Context updated", $"{key} = {value}");
    }
    
    public void RemoveContext(string key)
    {
        if (_context.Remove(key))
        {
            LogAgentAction("Context removed", key);
        }
    }
    
    public IReadOnlyDictionary<string, object> GetContext() => _context;
    
    public IReadOnlyList<AgentAction> GetActionHistory() => _actionHistory;
}

public class AgentAction
{
    public DateTime Timestamp { get; set; }
    public string AgentId { get; set; } = "";
    public string InputMessage { get; set; } = "";
    public string OutputMessage { get; set; } = "";
    public bool Success { get; set; }
    public string? ErrorMessage { get; set; }
}
```

> **What you're doing:** This creates an enhanced Azure AI Agent that includes context management, action history tracking, and more sophisticated Azure OpenAI integration specifically configured for Azure Government Cloud.

## Step 5: Create Conversation Thread Manager

8. **Create a new file called `ConversationThread.cs`**:

```csharp
using Microsoft.SemanticKernel.Contents;
using System.Text.Json;

public class ConversationThread
{
    private readonly string _threadId;
    private readonly List<ThreadMessage> _messages;
    private readonly Dictionary<string, object> _metadata;
    private readonly DateTime _createdAt;

    public string ThreadId => _threadId;
    public IReadOnlyList<ThreadMessage> Messages => _messages;
    public IReadOnlyDictionary<string, object> Metadata => _metadata;
    public DateTime CreatedAt => _createdAt;
    public DateTime LastUpdated { get; private set; }

    public ConversationThread(string? threadId = null)
    {
        _threadId = threadId ?? Guid.NewGuid().ToString();
        _messages = new List<ThreadMessage>();
        _metadata = new Dictionary<string, object>();
        _createdAt = DateTime.UtcNow;
        LastUpdated = _createdAt;
        
        Console.WriteLine($"🧵 Thread created: {_threadId}");
    }

    public void AddMessage(string agentId, string agentName, string content, ThreadMessageRole role)
    {
        var message = new ThreadMessage
        {
            Id = Guid.NewGuid().ToString(),
            AgentId = agentId,
            AgentName = agentName,
            Content = content,
            Role = role,
            Timestamp = DateTime.UtcNow
        };
        
        _messages.Add(message);
        LastUpdated = DateTime.UtcNow;
        
        Console.WriteLine($"📝 [{agentName}] {role}: {content}");
    }
    
    public void AddUserMessage(string content)
    {
        AddMessage("user", "User", content, ThreadMessageRole.User);
    }
    
    public void AddAgentMessage(string agentId, string agentName, string content)
    {
        AddMessage(agentId, agentName, content, ThreadMessageRole.Agent);
    }
    
    public void SetMetadata(string key, object value)
    {
        _metadata[key] = value;
        LastUpdated = DateTime.UtcNow;
    }
    
    public T? GetMetadata<T>(string key)
    {
        if (_metadata.TryGetValue(key, out var value) && value is T typedValue)
        {
            return typedValue;
        }
        return default(T);
    }
    
    public IEnumerable<ThreadMessage> GetMessagesByAgent(string agentId)
    {
        return _messages.Where(m => m.AgentId == agentId);
    }
    
    public string GetThreadSummary()
    {
        var summary = $"Thread {_threadId}: {_messages.Count} messages, " +
                     $"Created: {_createdAt:yyyy-MM-dd HH:mm:ss}, " +
                     $"Last Updated: {LastUpdated:yyyy-MM-dd HH:mm:ss}";
        return summary;
    }
    
    public string ExportToJson()
    {
        var export = new
        {
            ThreadId = _threadId,
            CreatedAt = _createdAt,
            LastUpdated = LastUpdated,
            Messages = _messages,
            Metadata = _metadata
        };
        
        return JsonSerializer.Serialize(export, new JsonSerializerOptions { WriteIndented = true });
    }
}

public class ThreadMessage
{
    public string Id { get; set; } = "";
    public string AgentId { get; set; } = "";
    public string AgentName { get; set; } = "";
    public string Content { get; set; } = "";
    public ThreadMessageRole Role { get; set; }
    public DateTime Timestamp { get; set; }
}

public enum ThreadMessageRole
{
    User,
    Agent,
    System
}
```

> **What you're doing:** This creates a conversation thread manager that tracks all messages, maintains metadata, and provides a structured way to manage multi-agent conversations with full history and context preservation.

## Step 6: Create Sequential Orchestration

9. **Create a new file called `SequentialOrchestrator.cs`**:

```csharp
using System.Text.Json;

public class SequentialOrchestrator
{
    private readonly List<AgentBase> _agents;
    private readonly ConversationThread _thread;
    private readonly List<OrchestrationStep> _orchestrationHistory;
    
    public delegate Task AgentResponseCallback(string agentId, string agentName, string response, bool isSuccess);
    public delegate Task<bool> HumanResponseFunction(string prompt, out string humanResponse);
    
    public event AgentResponseCallback? OnAgentResponse;
    public event HumanResponseFunction? OnHumanResponseRequired;

    public ConversationThread Thread => _thread;
    public IReadOnlyList<AgentBase> Agents => _agents;
    public IReadOnlyList<OrchestrationStep> OrchestrationHistory => _orchestrationHistory;

    public SequentialOrchestrator(ConversationThread? thread = null)
    {
        _agents = new List<AgentBase>();
        _thread = thread ?? new ConversationThread();
        _orchestrationHistory = new List<OrchestrationStep>();
        
        Console.WriteLine("🎭 Sequential Orchestrator initialized");
    }

    public void AddAgent(AgentBase agent)
    {
        _agents.Add(agent);
        Console.WriteLine($"➕ Agent added to orchestrator: {agent.Name} (ID: {agent.AgentId})");
    }

    public async Task<OrchestrationResult> ExecuteSequentialAsync(
        string initialMessage,
        CancellationToken cancellationToken = default)
    {
        Console.WriteLine($"\n🚀 Starting sequential orchestration with {_agents.Count} agents");
        Console.WriteLine($"Initial message: {initialMessage}");
        
        var result = new OrchestrationResult
        {
            StartTime = DateTime.UtcNow,
            InitialMessage = initialMessage,
            Steps = new List<OrchestrationStep>()
        };

        // Add initial user message to thread
        _thread.AddUserMessage(initialMessage);
        
        string currentMessage = initialMessage;
        
        try
        {
            for (int i = 0; i < _agents.Count; i++)
            {
                var agent = _agents[i];
                var step = new OrchestrationStep
                {
                    StepNumber = i + 1,
                    AgentId = agent.AgentId,
                    AgentName = agent.Name,
                    InputMessage = currentMessage,
                    StartTime = DateTime.UtcNow
                };

                Console.WriteLine($"\n🔄 Step {step.StepNumber}: Processing with {agent.Name}");
                
                try
                {
                    // Process message with current agent
                    var agentResponse = await agent.ProcessMessageAsync(currentMessage, cancellationToken);
                    
                    step.OutputMessage = agentResponse;
                    step.Success = true;
                    step.EndTime = DateTime.UtcNow;
                    
                    // Add agent response to thread
                    _thread.AddAgentMessage(agent.AgentId, agent.Name, agentResponse);
                    
                    // Trigger agent response callback
                    if (OnAgentResponse != null)
                    {
                        await OnAgentResponse(agent.AgentId, agent.Name, agentResponse, true);
                    }
                    
                    // Check if human response is required before proceeding to next agent
                    if (i < _agents.Count - 1 && OnHumanResponseRequired != null)
                    {
                        var prompt = $"Agent {agent.Name} responded: {agentResponse}\n\nWould you like to modify the input for the next agent ({_agents[i + 1].Name})? Current input will be: '{agentResponse}'";
                        
                        if (await OnHumanResponseRequired(prompt, out string humanResponse))
                        {
                            if (!string.IsNullOrEmpty(humanResponse))
                            {
                                currentMessage = humanResponse;
                                _thread.AddUserMessage($"Human intervention: {humanResponse}");
                                Console.WriteLine($"👤 Human provided modified input: {humanResponse}");
                            }
                            else
                            {
                                currentMessage = agentResponse;
                            }
                        }
                        else
                        {
                            currentMessage = agentResponse;
                        }
                    }
                    else
                    {
                        currentMessage = agentResponse;
                    }
                }
                catch (Exception ex)
                {
                    step.Success = false;
                    step.ErrorMessage = ex.Message;
                    step.EndTime = DateTime.UtcNow;
                    
                    Console.WriteLine($"❌ Error in step {step.StepNumber}: {ex.Message}");
                    
                    // Trigger agent response callback with error
                    if (OnAgentResponse != null)
                    {
                        await OnAgentResponse(agent.AgentId, agent.Name, $"Error: {ex.Message}", false);
                    }
                    
                    result.Success = false;
                    result.ErrorMessage = ex.Message;
                }
                
                result.Steps.Add(step);
                _orchestrationHistory.Add(step);
            }
            
            result.EndTime = DateTime.UtcNow;
            result.FinalResponse = currentMessage;
            result.Success = result.Success != false; // Only false if explicitly set to false due to error
            
            Console.WriteLine($"\n✅ Sequential orchestration completed successfully");
            Console.WriteLine($"Final response: {result.FinalResponse}");
        }
        catch (Exception ex)
        {
            result.Success = false;
            result.ErrorMessage = ex.Message;
            result.EndTime = DateTime.UtcNow;
            
            Console.WriteLine($"❌ Orchestration failed: {ex.Message}");
        }
        
        return result;
    }
    
    public string GetOrchestrationSummary()
    {
        var summary = new
        {
            ThreadId = _thread.ThreadId,
            AgentCount = _agents.Count,
            TotalSteps = _orchestrationHistory.Count,
            Agents = _agents.Select(a => new { a.AgentId, a.Name, a.Description }).ToList(),
            ThreadSummary = _thread.GetThreadSummary()
        };
        
        return JsonSerializer.Serialize(summary, new JsonSerializerOptions { WriteIndented = true });
    }
}

public class OrchestrationResult
{
    public DateTime StartTime { get; set; }
    public DateTime? EndTime { get; set; }
    public string InitialMessage { get; set; } = "";
    public string FinalResponse { get; set; } = "";
    public bool Success { get; set; } = true;
    public string? ErrorMessage { get; set; }
    public List<OrchestrationStep> Steps { get; set; } = new();
    public TimeSpan Duration => EndTime?.Subtract(StartTime) ?? TimeSpan.Zero;
}

public class OrchestrationStep
{
    public int StepNumber { get; set; }
    public string AgentId { get; set; } = "";
    public string AgentName { get; set; } = "";
    public string InputMessage { get; set; } = "";
    public string OutputMessage { get; set; } = "";
    public DateTime StartTime { get; set; }
    public DateTime? EndTime { get; set; }
    public bool Success { get; set; } = true;
    public string? ErrorMessage { get; set; }
    public TimeSpan Duration => EndTime?.Subtract(StartTime) ?? TimeSpan.Zero;
}
```

> **What you're doing:** This creates a sophisticated sequential orchestrator that manages multiple agents in sequence, handles callbacks, supports human-in-the-loop scenarios, and provides detailed tracking of the orchestration process.

## Step 7: Complete Application Implementation

10. **Update `Program.cs`** with the complete multi-agent implementation:

```csharp
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.Logging;
using Microsoft.SemanticKernel;
using Microsoft.SemanticKernel.Connectors.AzureOpenAI;
using System.Text;

class Program
{
    private static IConfiguration? _configuration;
    private static ILogger<Program>? _logger;

    static async Task Main(string[] args)
    {
        // Setup configuration
        _configuration = new ConfigurationBuilder()
            .AddJsonFile("appsettings.json", optional: false, reloadOnChange: true)
            .Build();

        // Setup logging
        using var loggerFactory = LoggerFactory.Create(builder =>
            builder.AddConsole().SetMinimumLevel(LogLevel.Information));
        _logger = loggerFactory.CreateLogger<Program>();

        Console.WriteLine("🤖 Semantic Kernel Multi-Agent Orchestration Lab");
        Console.WriteLine("=================================================\n");

        try
        {
            // Create kernel with Azure OpenAI for Azure Government
            var kernel = CreateKernel();
            
            // Demonstrate single agent creation and usage
            await DemonstrateSingleAgentAsync(kernel);
            
            // Demonstrate thread management
            await DemonstrateThreadManagementAsync(kernel);
            
            // Demonstrate multi-agent orchestration
            await DemonstrateMultiAgentOrchestrationAsync(kernel);
        }
        catch (Exception ex)
        {
            Console.WriteLine($"❌ Application error: {ex.Message}");
            _logger?.LogError(ex, "Application failed");
        }

        Console.WriteLine("\n🎉 Multi-Agent Lab completed!");
        Console.WriteLine("Press any key to exit...");
        Console.ReadKey();
    }

    private static Kernel CreateKernel()
    {
        var builder = Kernel.CreateBuilder();

        // Get Azure OpenAI configuration
        var endpoint = _configuration!["AzureOpenAI:Endpoint"]!;
        var apiKey = _configuration!["AzureOpenAI:ApiKey"]!;
        var deploymentName = _configuration!["AzureOpenAI:ChatDeploymentName"]!;

        Console.WriteLine($"🔧 Configuring kernel with Azure Government endpoint: {endpoint}");

        // Add Azure OpenAI chat completion for Azure Government
        builder.AddAzureOpenAIChatCompletion(
            deploymentName: deploymentName,
            endpoint: endpoint,
            apiKey: apiKey);

        // Add logging
        builder.Services.AddLogging(c => c.AddConsole().SetMinimumLevel(LogLevel.Information));

        var kernel = builder.Build();
        Console.WriteLine("✅ Kernel created successfully with Azure Government configuration!\n");

        return kernel;
    }

    private static async Task DemonstrateSingleAgentAsync(Kernel kernel)
    {
        Console.WriteLine("🔹 DEMONSTRATION 1: Single Agent Creation and Usage");
        Console.WriteLine("====================================================");

        // Create a Chat Completion Agent
        var chatAgent = new ChatCompletionAgent(
            agentId: "chat-agent-001",
            name: "Chat Assistant",
            description: "A general-purpose chat assistant",
            systemPrompt: "You are a helpful AI assistant that provides clear, concise, and accurate responses.",
            kernel: kernel);

        // Create an Azure AI Agent
        var azureAgent = new AzureAIAgent(
            agentId: "azure-agent-001", 
            name: "Azure Specialist",
            description: "An AI agent specialized in Azure services and solutions",
            systemPrompt: "You are an Azure expert who helps users with Azure services, best practices, and solutions. Always provide practical, actionable advice.",
            kernel: kernel);

        // Test Chat Completion Agent
        Console.WriteLine("\n🧪 Testing Chat Completion Agent:");
        var chatResponse = await chatAgent.ProcessMessageAsync("What is artificial intelligence?");
        Console.WriteLine($"Response: {chatResponse}\n");

        // Test Azure AI Agent with context
        Console.WriteLine("🧪 Testing Azure AI Agent with context:");
        azureAgent.AddContext("user_role", "Cloud Architect");
        azureAgent.AddContext("project_type", "Government Compliance");
        
        var azureResponse = await azureAgent.ProcessMessageAsync("How should I design a secure Azure architecture for a government application?");
        Console.WriteLine($"Response: {azureResponse}\n");

        // Show agent capabilities
        Console.WriteLine("📊 Agent Information:");
        Console.WriteLine($"Chat Agent Messages: {chatAgent.GetMessageCount()}");
        Console.WriteLine($"Azure Agent Context: {azureAgent.GetContext().Count} items");
        Console.WriteLine($"Azure Agent Actions: {azureAgent.GetActionHistory().Count} actions\n");
    }

    private static async Task DemonstrateThreadManagementAsync(Kernel kernel)
    {
        Console.WriteLine("🔹 DEMONSTRATION 2: Conversation Thread Management");
        Console.WriteLine("===================================================");

        // Create a conversation thread
        var thread = new ConversationThread();
        Console.WriteLine($"Thread Details: {thread.GetThreadSummary()}\n");

        // Create agents for the thread
        var analyzerAgent = new ChatCompletionAgent(
            agentId: "analyzer-001",
            name: "Data Analyzer",
            description: "Analyzes data and provides insights",
            systemPrompt: "You are a data analyst who examines information and provides clear insights and recommendations.",
            kernel: kernel);

        var reviewerAgent = new AzureAIAgent(
            agentId: "reviewer-001",
            name: "Content Reviewer", 
            description: "Reviews and validates content",
            systemPrompt: "You are a content reviewer who ensures accuracy, clarity, and completeness of information.",
            kernel: kernel);

        // Simulate a conversation in the thread
        Console.WriteLine("🗨️ Simulating conversation in thread:");
        
        var userMessage = "Please analyze the benefits of using Azure Government for sensitive workloads";
        thread.AddUserMessage(userMessage);

        var analysisResponse = await analyzerAgent.ProcessMessageAsync(userMessage);
        thread.AddAgentMessage(analyzerAgent.AgentId, analyzerAgent.Name, analysisResponse);

        var reviewResponse = await reviewerAgent.ProcessMessageAsync($"Please review this analysis: {analysisResponse}");
        thread.AddAgentMessage(reviewerAgent.AgentId, reviewerAgent.Name, reviewResponse);

        // Show thread information
        Console.WriteLine($"\n📊 Thread Summary: {thread.GetThreadSummary()}");
        Console.WriteLine($"Total Messages: {thread.Messages.Count}");
        
        thread.SetMetadata("topic", "Azure Government Analysis");
        thread.SetMetadata("participants", new[] { "User", analyzerAgent.Name, reviewerAgent.Name });
        
        Console.WriteLine("💾 Thread exported to JSON:");
        Console.WriteLine(thread.ExportToJson().Substring(0, Math.Min(500, thread.ExportToJson().Length)) + "...\n");
    }

    private static async Task DemonstrateMultiAgentOrchestrationAsync(Kernel kernel)
    {
        Console.WriteLine("🔹 DEMONSTRATION 3: Multi-Agent Sequential Orchestration");
        Console.WriteLine("=========================================================");

        // Create specialized agents for the orchestration
        var requirements_agent = new ChatCompletionAgent(
            agentId: "requirements-agent",
            name: "Requirements Analyst",
            description: "Analyzes and structures requirements",
            systemPrompt: "You are a requirements analyst. Take user input and structure it into clear, detailed requirements. Format your response as a numbered list.",
            kernel: kernel);

        var architect_agent = new AzureAIAgent(
            agentId: "architect-agent",
            name: "Solutions Architect", 
            description: "Designs technical solutions",
            systemPrompt: "You are a solutions architect. Based on requirements, design a high-level technical solution. Focus on Azure services and architecture patterns.",
            kernel: kernel);

        var security_agent = new ChatCompletionAgent(
            agentId: "security-agent",
            name: "Security Specialist",
            description: "Reviews solutions for security compliance",
            systemPrompt: "You are a security specialist focused on Azure Government compliance. Review the proposed solution and add security recommendations, especially for government/compliance requirements.",
            kernel: kernel);

        // Create orchestrator and add agents
        var orchestrator = new SequentialOrchestrator();
        orchestrator.AddAgent(requirements_agent);
        orchestrator.AddAgent(architect_agent);
        orchestrator.AddAgent(security_agent);

        // Implement agent response callback
        orchestrator.OnAgentResponse += async (agentId, agentName, response, isSuccess) =>
        {
            Console.WriteLine($"🔔 Agent Response Callback triggered:");
            Console.WriteLine($"   Agent: {agentName} (ID: {agentId})");
            Console.WriteLine($"   Success: {isSuccess}");
            Console.WriteLine($"   Response Length: {response.Length} characters");
            
            if (!isSuccess)
            {
                Console.WriteLine($"   ⚠️ Agent encountered an error");
            }
            
            // You could log to external systems, trigger notifications, etc.
            await Task.Delay(100); // Simulate some callback processing
        };

        // Implement human response function
        orchestrator.OnHumanResponseRequired += async (prompt, out string humanResponse) =>
        {
            Console.WriteLine($"\n🤔 Human input requested:");
            Console.WriteLine($"Prompt: {prompt}");
            Console.WriteLine("\nOptions:");
            Console.WriteLine("1. Press ENTER to continue with agent output");
            Console.WriteLine("2. Type custom input to modify the flow");
            Console.Write("Your choice: ");
            
            var input = Console.ReadLine();
            
            if (string.IsNullOrWhiteSpace(input))
            {
                humanResponse = "";
                Console.WriteLine("✅ Continuing with agent output...\n");
                return await Task.FromResult(false); // No modification
            }
            else
            {
                humanResponse = input;
                Console.WriteLine($"✅ Using human input: {input}\n");
                return await Task.FromResult(true); // Human provided modification
            }
        };

        // Execute the orchestration
        var initialRequest = "I need to build a secure document management system for a government agency that handles classified documents. The system needs to be compliant with FedRAMP and support multiple users with role-based access.";
        
        Console.WriteLine($"🚀 Starting orchestration with request:");
        Console.WriteLine($"'{initialRequest}'\n");

        var result = await orchestrator.ExecuteSequentialAsync(initialRequest);

        // Display results
        Console.WriteLine("\n📋 ORCHESTRATION RESULTS:");
        Console.WriteLine("=========================");
        Console.WriteLine($"Success: {result.Success}");
        Console.WriteLine($"Duration: {result.Duration.TotalSeconds:F2} seconds");
        Console.WriteLine($"Steps Completed: {result.Steps.Count}");

        if (!result.Success && !string.IsNullOrEmpty(result.ErrorMessage))
        {
            Console.WriteLine($"Error: {result.ErrorMessage}");
        }

        Console.WriteLine($"\n📄 Final Response:");
        Console.WriteLine($"{result.FinalResponse}");

        // Show detailed step information
        Console.WriteLine($"\n🔍 Detailed Step Information:");
        foreach (var step in result.Steps)
        {
            Console.WriteLine($"Step {step.StepNumber}: {step.AgentName}");
            Console.WriteLine($"  Duration: {step.Duration.TotalMilliseconds:F0}ms");
            Console.WriteLine($"  Success: {step.Success}");
            if (!step.Success && !string.IsNullOrEmpty(step.ErrorMessage))
            {
                Console.WriteLine($"  Error: {step.ErrorMessage}");
            }
        }

        // Export orchestration summary
        Console.WriteLine($"\n💾 Orchestration Summary:");
        Console.WriteLine(orchestrator.GetOrchestrationSummary());
    }
}
```

> **What you're doing:** This complete application demonstrates all aspects of multi-agent orchestration including single agent creation, thread management, sequential orchestration with callbacks, and human-in-the-loop functionality.

## Running the Multi-Agent Lab

1. **Navigate to your project folder** in VS Code
2. **Update `appsettings.json`** with your actual Azure Government credentials:
   ```json
   {
     "AzureOpenAI": {
       "Endpoint": "https://your-actual-resource.openai.usgovcloudapi.net/",
       "ApiKey": "your-actual-api-key",
       "ChatDeploymentName": "gpt-4",
       "ModelId": "gpt-4"
     }
   }
   ```
3. **Run the application:**
   ```bash
   dotnet run
   ```
4. **Follow the interactive prompts** during the multi-agent orchestration demonstration

## Key Learning Points

### What You've Built

**Single Agent Creation:**
- **Chat Completion Agent:** Basic conversational AI with history management
- **Azure AI Agent:** Enhanced agent with context management and action tracking
- Both agents configured specifically for Azure Government Cloud compliance

**Thread Management:**
- **Conversation Threads:** Structured conversation management with metadata
- **Message Tracking:** Complete history of all agent and user interactions
- **Export Capabilities:** JSON export for persistence and analysis

**Multi-Agent Orchestration:**
- **Sequential Processing:** Agents process messages in a defined order
- **Agent Response Callbacks:** Real-time notifications of agent actions
- **Human Response Functions:** Human-in-the-loop capability for guided orchestration
- **Error Handling:** Comprehensive error management and recovery

**Advanced Features:**
- **Context Management:** Agents can maintain and share context information
- **Action History:** Detailed tracking of all agent actions and decisions
- **Metadata Management:** Rich metadata support for threads and conversations
- **Azure Government Integration:** All components designed for government compliance requirements

### Development Patterns

**Agent Architecture:**
```
AgentBase (Abstract)
├── ChatCompletionAgent (Basic conversational agent)
└── AzureAIAgent (Enhanced agent with context and tracking)
```

**Orchestration Flow:**
```
User Input → Agent 1 → [Human Intervention?] → Agent 2 → ... → Final Response
     ↓            ↓                                ↓
  Thread      Callback                        Callback
```

**File Organization:**
```
Multi-Agent Project:
├── Program.cs (main application)
├── appsettings.json (Azure Government configuration)
├── AgentBase.cs (base agent class)
├── ChatCompletionAgent.cs (basic agent implementation)
├── AzureAIAgent.cs (enhanced agent implementation)
├── ConversationThread.cs (thread management)
└── SequentialOrchestrator.cs (orchestration engine)
```

### Best Practices Demonstrated

**Security:**
- Environment-based configuration management
- Azure Government specific endpoints and compliance
- Secure credential handling through configuration files
- Context isolation between agents

**Error Handling:**
- Comprehensive try-catch blocks in all async operations
- Graceful degradation when agents fail
- Detailed error logging and reporting
- Recovery mechanisms in orchestration

**Performance:**
- Asynchronous processing throughout
- Efficient memory management with readonly collections
- Cancellation token support for long-running operations
- Minimal object allocation in hot paths

**Maintainability:**
- Clear separation of concerns between components
- Extensible agent architecture through inheritance
- Configurable orchestration patterns
- Comprehensive logging and diagnostics

### Troubleshooting Tips

**Common Issues:**
1. **Agent Creation Failures:** Verify Azure OpenAI deployment names and endpoints
2. **Orchestration Errors:** Check agent system prompts for clarity and completeness
3. **Callback Issues:** Ensure async/await patterns are properly implemented
4. **Thread Management:** Verify proper message ordering and metadata handling
5. **Azure Government Access:** Confirm subscription has access to required services

**Performance Considerations:**
- Use appropriate Azure OpenAI model sizes for your use case
- Implement proper cancellation for long-running orchestrations
- Monitor token usage across multiple agents
- Consider implementing agent response caching for repeated queries

**Government Compliance:**
- All data processing occurs within Azure Government boundaries
- Maintain audit trails of all agent interactions
- Implement proper access controls for sensitive conversations
- Follow government data handling and retention policies

This comprehensive multi-agent lab demonstrates advanced Semantic Kernel capabilities for building sophisticated AI agent systems that can handle complex, multi-step processes while maintaining Azure Government compliance and security requirements.
