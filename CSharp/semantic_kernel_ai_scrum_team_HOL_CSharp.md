# HOL - AI Scrum Team with Semantic Kernel (C#)

## ⚠️ CRITICAL: QATester Improvements

**Latest update: Enhanced QA Tester for better content generation**

1. **Removed "TEST CASES COMPLETE" instruction**: Termination now uses reliable agent response counting
2. **Enhanced QA instructions**: More detailed guidance for comprehensive test scenarios  
3. **GPT-3.5-Turbo optimization**: Simplified instructions work better with this model
4. **Better content structure**: Clear markdown formatting for professional output

**Key Changes Made:**
- QATester focuses on content quality instead of completion signals
- Workflow uses agent response counting (more reliable than text-based termination)
- Test scenarios now include edge cases, security, performance, and integration tests
- Instructions optimized for consistent output across different AI models

## Overview

In this hands-on lab, you'll learn how to build an AI-first Scrum team using Microsoft Semantic Kernel's Group Chat Orchestration pattern in C#. This system simulates a collaborative Scrum team that transforms raw requirements into production-ready Agile artifacts through coordinated AI agents.

> **⚠️ Important Note**: This HOL has been updated to use the latest Semantic Kernel Agent Framework. The old `AgentGroupChat` pattern has been deprecated and replaced with the new `GroupChatOrchestration` pattern. This version uses the current stable architecture as of August 2025.

## What You'll Build

- **Product Owner Agent**: Generates System Requirements Specification (SRS) from raw requirements
- **Business Analyst Agent**: Breaks SRS into detailed User Stories & Acceptance Criteria  
- **Solution Architect Agent**: Designs solution architecture and identifies technical dependencies
- **QA Tester Agent**: Writes comprehensive test scenarios and test cases in Gherkin syntax
- **Custom Group Chat Manager**: Controls conversation flow, turn-taking, and termination logic

## Learning Objectives

By the end of this lab, you will:

- Understand Semantic Kernel's Group Chat Orchestration pattern in C#
- Create specialized AI agents with role-specific instructions
- Implement custom conversation management logic
- Build a complete AI Scrum team workflow: Raw Requirements → SRS → User Stories → Architecture → Test Cases
- Handle agent coordination and output management in .NET

## Prerequisites

- .NET 8.0 or higher
- Basic understanding of async/await patterns in C#
- Familiarity with Agile/Scrum methodologies  
- Azure OpenAI account with deployment
- Visual Studio 2022 or VS Code with C# extension

## Technology Stack

- **Microsoft.SemanticKernel**: Core Semantic Kernel framework
- **Microsoft.SemanticKernel.Agents.Core**: Agent creation and ChatCompletionAgent functionality
- **Microsoft.SemanticKernel.Agents.Orchestration**: Group chat orchestration patterns (prerelease)
- **Microsoft.SemanticKernel.Agents.Runtime.InProcess**: In-process runtime for agent orchestration (prerelease)
- **Microsoft.SemanticKernel.Connectors.AzureOpenAI**: Azure OpenAI Connector
- **Microsoft.Extensions.Configuration**: Configuration management
- **Microsoft.Extensions.Configuration.UserSecrets**:
- **Microsoft.Extensions.Logging**: Logging support
- **Microsoft.Extensions.Logging.Console**:

## Step 1: Project Setup

### Create New Console Application

```bash
dotnet new console -n AIScrumTeam
cd AIScrumTeam
```

### Install NuGet Packages

```bash
# Core Semantic Kernel
dotnet add package Microsoft.SemanticKernel

# Azure OpenAI Connector (required for AddAzureOpenAIChatCompletion)
dotnet add package Microsoft.SemanticKernel.Connectors.AzureOpenAI

# Agent packages (current structure)
dotnet add package Microsoft.SemanticKernel.Agents.Core
dotnet add package Microsoft.SemanticKernel.Agents.Orchestration --prerelease
dotnet add package Microsoft.SemanticKernel.Agents.Runtime.InProcess --prerelease

# Configuration and logging packages
dotnet add package Microsoft.Extensions.Configuration
dotnet add package Microsoft.Extensions.Configuration.UserSecrets
dotnet add package Microsoft.Extensions.Logging
dotnet add package Microsoft.Extensions.Logging.Console
```

### Suppress Experimental Warnings

Since the Agent Framework and Orchestration features are experimental, add the following to your `.csproj` file to suppress compiler warnings:

```xml
<PropertyGroup>
  <NoWarn>$(NoWarn);SKEXP0110</NoWarn>
</PropertyGroup>
```

**Note**: `SKEXP0110` covers Semantic Kernel Agents experimental features. You can also use pragma directives in individual files if preferred.

### Important Notes About Experimental Features

- **Breaking Changes**: Experimental features may introduce breaking changes without notice
- **Limited Support**: Microsoft provides limited support for experimental features  
- **Production Use**: Consider carefully before using experimental features in production
- **Documentation**: Keep up to date with the [Semantic Kernel release notes](https://github.com/microsoft/semantic-kernel/releases) for changes

### Project Structure

Create the following folder structure:

```
AIScrumTeam/
├── Agents/
│   ├── ProductOwnerAgent.cs
│   ├── BusinessAnalystAgent.cs
│   ├── SolutionArchitectAgent.cs
│   └── QATestAgent.cs
├── Managers/
│   └── ScrumGroupChatManager.cs
├── Models/
│   └── ScrumDeliverables.cs
├── Services/
│   └── ScrumTeamOrchestrator.cs
├── Program.cs
└── appsettings.json
```

### Configuration Setup

Create `appsettings.json`:

```json
{
  "AzureOpenAI": {
    "Endpoint": "https://your-resource.openai.azure.com/",
    "ApiKey": "your-api-key",
    "DeploymentName": "gpt-4"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.SemanticKernel": "Warning"
    }
  }
}
```

## Step 2: Create Base Agent Infrastructure

### Agent Configuration Model

Create `Models/ScrumDeliverables.cs`:

```csharp
using System.ComponentModel;
using System.Text;

namespace AIScrumTeam.Models
{
    public class ScrumDeliverables
    {
        public string? ProductOwnerOutput { get; set; }
        public string? BusinessAnalystOutput { get; set; }
        public string? SolutionArchitectOutput { get; set; }
        public string? QATestOutput { get; set; }
        public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
        
        public string GenerateMarkdownReport()
        {
            var report = new StringBuilder();
            report.AppendLine("# 📋 Scrum AI Team Deliverables");
            report.AppendLine();
            
            if (!string.IsNullOrEmpty(ProductOwnerOutput))
            {
                report.AppendLine("## ProductOwner");
                report.AppendLine(ProductOwnerOutput);
                report.AppendLine();
            }
            
            if (!string.IsNullOrEmpty(BusinessAnalystOutput))
            {
                report.AppendLine("## BusinessAnalyst");
                report.AppendLine(BusinessAnalystOutput);
                report.AppendLine();
            }
            
            if (!string.IsNullOrEmpty(SolutionArchitectOutput))
            {
                report.AppendLine("## SolutionArchitect");
                report.AppendLine(SolutionArchitectOutput);
                report.AppendLine();
            }
            
            if (!string.IsNullOrEmpty(QATestOutput))
            {
                report.AppendLine("## QATester");
                report.AppendLine(QATestOutput);
                report.AppendLine();
            }
            
            return report.ToString();
        }
    }
}
```

## Step 3: Create Individual Agents

### Product Owner Agent

Create `Agents/ProductOwnerAgent.cs`:

```csharp
using Microsoft.SemanticKernel;
using Microsoft.SemanticKernel.Agents;

namespace AIScrumTeam.Agents
{
    public static class ProductOwnerAgent
    {
        public static ChatCompletionAgent Create(Kernel kernel)
        {
            return new ChatCompletionAgent()
            {
                Name = "ProductOwner",
                Instructions = """
                    You are an experienced Product Owner with expertise in Agile methodology.
                    Your job is to transform raw, high-level business requirements into a clear, 
                    structured System Requirements Specification (SRS).
                    
                    The SRS should include:
                    1. Project Overview with system name and purpose
                    2. Business Requirements (BR1, BR2, etc.)
                    3. System Requirements:
                       - Functional Requirements (detailed feature descriptions)
                       - Non-Functional Requirements (performance, security, scalability, UI/UX)
                    4. Constraints (technical and business limitations)
                    5. Assumptions (what we assume to be true)
                    6. Business Rules (operational rules and policies)
                    7. Summary
                    
                    Structure your response clearly with these sections and ensure all requirements
                    are specific, measurable, and testable.
                    """,
                Kernel = kernel,
                Description = "Generates detailed System Requirements Specification from raw requirements"
            };
        }
    }
}
```

### Business Analyst Agent

Create `Agents/BusinessAnalystAgent.cs`:

```csharp
using Microsoft.SemanticKernel;
using Microsoft.SemanticKernel.Agents;

namespace AIScrumTeam.Agents
{
    public static class BusinessAnalystAgent
    {
        public static ChatCompletionAgent Create(Kernel kernel)
        {
            return new ChatCompletionAgent()
            {
                Name = "BusinessAnalyst",
                Instructions = """
                    You are a professional Business Analyst specializing in Agile user story creation.
                    
                    Your task is to take the System Requirements Specification (SRS) and break it down 
                    into detailed, implementable user stories with comprehensive acceptance criteria.
                    
                    For each story:
                    1. Use the standard format: "As a [user type], I want [goal] so that [benefit]"
                    2. Organize stories into logical Epics (e.g., "Epic 1: User Management")
                    3. Write detailed acceptance criteria in Given/When/Then format
                    4. Include multiple scenarios per story (happy path, error cases, edge cases)
                    5. Ensure stories are small, testable, and independent
                    
                    Structure your response with:
                    - Brief introduction explaining your approach
                    - Epics with numbered user stories
                    - Comprehensive acceptance criteria for each story
                    - Clear separation between different scenarios
                    
                    Make sure all stories follow INVEST principles (Independent, Negotiable, 
                    Valuable, Estimable, Small, Testable).
                    """,
                Kernel = kernel,
                Description = "Breaks SRS into detailed user stories with Given/When/Then acceptance criteria"
            };
        }
    }
}
```

### Solution Architect Agent

Create `Agents/SolutionArchitectAgent.cs`:

```csharp
using Microsoft.SemanticKernel;
using Microsoft.SemanticKernel.Agents;

namespace AIScrumTeam.Agents
{
    public static class SolutionArchitectAgent
    {
        public static ChatCompletionAgent Create(Kernel kernel)
        {
            return new ChatCompletionAgent()
            {
                Name = "SolutionArchitect",
                Instructions = """
                    You are a senior Solution Architect with expertise in designing scalable, 
                    secure, and maintainable systems.
                    
                    Review the user stories and create a comprehensive high-level solution architecture.
                    
                    Your architecture should include:
                    
                    1. **Major Components**
                       - Frontend (technology options, functionality, dependencies)
                       - Backend (technology options, business logic, APIs)
                       - Database (technology choices, key tables/collections)
                       - Third-party integrations (payment gateways, APIs, etc.)
                       - Authentication and Security systems
                    
                    2. **System Interactions**
                       - User journey workflows
                       - Data flow diagrams (conceptual)
                       - Integration patterns
                    
                    3. **Infrastructure Needs**
                       - Hosting and compute services
                       - Security measures
                       - Scalability considerations
                       - Backup and disaster recovery
                    
                    4. **Technical Dependencies and Risks**
                       - External service dependencies
                       - Potential risks and mitigation strategies
                       - Performance considerations
                    
                    5. **Additional Technical User Stories**
                       - System-level stories for infrastructure, monitoring, etc.
                    
                    Focus on modern cloud-native architecture patterns, security best practices,
                    and scalability. Provide specific technology recommendations with rationale.
                    """,
                Kernel = kernel,
                Description = "Designs solution architecture and identifies technical dependencies"
            };
        }
    }
}
```

### QA Test Agent

Create `Agents/QATestAgent.cs`:

```csharp
using Microsoft.SemanticKernel;
using Microsoft.SemanticKernel.Agents;

namespace AIScrumTeam.Agents
{
    public static class QATestAgent
    {
        public static ChatCompletionAgent Create(Kernel kernel)
        {
            return new ChatCompletionAgent()
            {
                Name = "QATester",
                Instructions = """
                    You are a QA Engineer. Based on the user stories and solution architecture, create comprehensive test scenarios.
                    
                    Write 5-7 test scenarios in this format:
                    
                    ## Test Scenarios
                    
                    **Feature: [Feature Name]**
                    
                    ### Scenario 1: [Happy Path Test]
                    - **Given:** [Initial conditions]
                    - **When:** [User action]
                    - **Then:** [Expected result]
                    
                    ### Scenario 2: [Error Handling Test]
                    - **Given:** [Initial conditions]
                    - **When:** [Invalid action]
                    - **Then:** [Error handling result]
                    
                    ### Scenario 3: [Security Test]
                    - **Given:** [Security context]
                    - **When:** [Security-related action]
                    - **Then:** [Security validation result]
                    
                    Continue with additional scenarios covering:
                    - Edge cases and boundary conditions
                    - Integration points
                    - Performance expectations
                    - Data validation
                    
                    Provide detailed, testable scenarios that a developer can implement.
                    """,
                Kernel = kernel,
                Description = "Generates comprehensive test scenarios in structured format"
            };
        }
    }
}
```

## Step 4: Create Custom Group Chat Manager

Create `Managers/ScrumGroupChatManager.cs`:

```csharp
#pragma warning disable SKEXP0110 // Semantic Kernel Agents
#pragma warning disable SKEXP0001 // Semantic Kernel Chat History

using Microsoft.SemanticKernel.Agents;
using Microsoft.SemanticKernel.Agents.Orchestration.GroupChat;
using Microsoft.SemanticKernel.ChatCompletion;
using Microsoft.Extensions.Logging;

namespace AIScrumTeam.Managers
{
    public class ScrumGroupChatManager : GroupChatManager
    {
        private readonly ILogger<ScrumGroupChatManager> _logger;
        private readonly string[] _agentOrder = { "ProductOwner", "BusinessAnalyst", "SolutionArchitect", "QATester" };
        private bool _isCompleted = false;
        
        public ScrumGroupChatManager(ILogger<ScrumGroupChatManager> logger) 
        {
            _logger = logger;
            MaximumInvocationCount = 20; // Increased from 15 to give more room for completion
        }

        public override ValueTask<GroupChatManagerResult<string>> SelectNextAgent(ChatHistory history, GroupChatTeam team, CancellationToken cancellationToken = default)
        {
            // Check for completion first to avoid further agent selection
            if (_isCompleted)
            {
                _logger.LogInformation("Workflow already completed, no next agent");
                return ValueTask.FromResult(new GroupChatManagerResult<string>("") { Reason = "Workflow complete" });
            }

            var lastMessage = history.LastOrDefault();
            var lastAgentName = lastMessage?.AuthorName;
            
            _logger.LogInformation($"Selecting next agent. Last message from: {lastAgentName ?? "None"}");
            
            // SIMPLIFIED: If QATester just responded, terminate immediately
            if (lastAgentName == "QATester")
            {
                _logger.LogInformation("QATester has responded, marking workflow as complete");
                _isCompleted = true;
                return ValueTask.FromResult(new GroupChatManagerResult<string>("") { Reason = "QATester completed" });
            }
            
            // Count agent responses for monitoring
            var agentCounts = history
                .Where(m => !string.IsNullOrEmpty(m.AuthorName) && m.AuthorName != "user")
                .GroupBy(m => m.AuthorName!)
                .ToDictionary(g => g.Key, g => g.Count());
            
            // Simple sequential workflow: PO → BA → SA → QA → DONE
            var nextAgentName = lastAgentName switch
            {
                null or "user" => "ProductOwner",
                "ProductOwner" => "BusinessAnalyst", 
                "BusinessAnalyst" => "SolutionArchitect",
                "SolutionArchitect" => "QATester",
                _ => "" // Force termination for any other case
            };
            
            _logger.LogInformation($"Selected next agent: {nextAgentName}");
            return ValueTask.FromResult(new GroupChatManagerResult<string>(nextAgentName));
        }
        
        private string HandleQANext(ChatHistory history, Dictionary<string, int> agentCounts)
        {
            var qaCount = agentCounts.GetValueOrDefault("QATester", 0);
            
            // If QA has responded multiple times, check for completion more strictly
            if (qaCount >= 2)
            {
                _logger.LogWarning($"QA has responded {qaCount} times. Checking for completion signals.");
                
                // Look for any completion indicators in QA messages
                var qaMessages = history
                    .Where(m => m.AuthorName == "QATester")
                    .Select(m => m.Content ?? "")
                    .ToList();
                    
                var hasAnyCompletionSignal = qaMessages.Any(content => 
                    content.Contains("TEST CASES COMPLETE", StringComparison.OrdinalIgnoreCase) ||
                    content.Contains("testing complete", StringComparison.OrdinalIgnoreCase) ||
                    content.Contains("test cases complete", StringComparison.OrdinalIgnoreCase) ||
                    content.Contains("qa complete", StringComparison.OrdinalIgnoreCase));
                    
                if (hasAnyCompletionSignal || qaCount >= 3)
                {
                    _logger.LogInformation("Forcing completion due to QA signals or limit reached");
                    _isCompleted = true;
                    return ""; // Trigger termination
                }
            }
            
            // Allow QA another chance
            return "QATester";
        }
        
        private bool HasQACompleted(ChatHistory history)
        {
            var qaMessages = history
                .Where(m => m.AuthorName == "QATester")
                .Select(m => m.Content ?? "")
                .ToList();
                
            return qaMessages.Any(content => 
                content.Contains("TEST CASES COMPLETE", StringComparison.OrdinalIgnoreCase) ||
                content.Contains("test cases complete", StringComparison.OrdinalIgnoreCase));
        }

        public override ValueTask<GroupChatManagerResult<bool>> ShouldTerminate(ChatHistory history, CancellationToken cancellationToken = default)
        {
            // Primary termination condition: workflow marked as completed
            if (_isCompleted)
            {
                _logger.LogInformation("🛑 Terminating: Workflow marked as completed");
                return ValueTask.FromResult(new GroupChatManagerResult<bool>(true) { Reason = "Workflow completed successfully" });
            }
            
            // SIMPLIFIED: Terminate immediately if QATester has responded
            // This is MUCH better than waiting for specific text like "TEST CASES COMPLETE"
            var qaResponseCount = history.Count(m => m.AuthorName == "QATester");
            if (qaResponseCount >= 1)
            {
                _logger.LogInformation("🛑 Terminating: QATester has provided response - this is more reliable than text-based termination");
                _isCompleted = true;
                return ValueTask.FromResult(new GroupChatManagerResult<bool>(true) { Reason = "QATester completed" });
            }
            
            // Safety termination conditions
            
            // Check for maximum conversation length (safety condition)
            var maxMessages = 50;
            var isMaxReached = history.Count >= maxMessages;
            
            if (isMaxReached)
            {
                _logger.LogWarning($"🛑 Terminating: Maximum message limit reached ({maxMessages})");
                return ValueTask.FromResult(new GroupChatManagerResult<bool>(true) { Reason = $"Maximum message limit reached ({maxMessages})" });
            }
            
            // Check for stuck agent (same agent responding many times in a row)
            var lastFiveMessages = history.TakeLast(5).ToList();
            var isStuck = lastFiveMessages.Count >= 3 && 
                         lastFiveMessages.All(m => m.AuthorName == lastFiveMessages.First().AuthorName);
            
            if (isStuck)
            {
                _logger.LogWarning("🛑 Terminating: Agent appears to be stuck in a loop");
                return ValueTask.FromResult(new GroupChatManagerResult<bool>(true) { Reason = "Agent stuck in loop" });
            }
            
            var reason = $"Workflow continuing (Messages: {history.Count}, QA responses: {qaResponseCount})";
            _logger.LogInformation($"Termination check: {reason}");
            
            return ValueTask.FromResult(new GroupChatManagerResult<bool>(false) { Reason = reason });
        }

        public override ValueTask<GroupChatManagerResult<bool>> ShouldRequestUserInput(ChatHistory history, CancellationToken cancellationToken = default)
        {
            return ValueTask.FromResult(new GroupChatManagerResult<bool>(false) 
            { 
                Reason = "The AI scrum team does not request user input during internal deliberations." 
            });
        }

        public override ValueTask<GroupChatManagerResult<string>> FilterResults(ChatHistory history, CancellationToken cancellationToken = default)
        {
            // Create a summary of the scrum team's work
            var summary = "AI Scrum Team has completed their comprehensive analysis and deliverables generation.";
            
            return ValueTask.FromResult(new GroupChatManagerResult<string>(summary));
        }
    }
}
```

## Step 5: Create Orchestration Service

Create `Services/ScrumTeamOrchestrator.cs`:

```csharp
#pragma warning disable SKEXP0110 // Semantic Kernel Agents
#pragma warning disable SKEXP0001 // Semantic Kernel Chat History 

using Microsoft.SemanticKernel;
using Microsoft.SemanticKernel.Agents;
using Microsoft.SemanticKernel.Agents.Orchestration;
using Microsoft.SemanticKernel.Agents.Orchestration.GroupChat;
using Microsoft.SemanticKernel.Agents.Runtime.InProcess;
using Microsoft.SemanticKernel.ChatCompletion;
using Microsoft.SemanticKernel.Connectors.AzureOpenAI; // Required for AddAzureOpenAIChatCompletion
using AIScrumTeam.Agents;
using AIScrumTeam.Managers;
using AIScrumTeam.Models;

namespace AIScrumTeam.Services
{
    public class ScrumTeamOrchestrator
    {
        private readonly IConfiguration _configuration;
        private readonly ILogger<ScrumTeamOrchestrator> _logger;
        private readonly ILoggerFactory _loggerFactory;
        private readonly Kernel _kernel;

        public ScrumTeamOrchestrator(IConfiguration configuration, ILogger<ScrumTeamOrchestrator> logger, ILoggerFactory loggerFactory)
        {
            _configuration = configuration;
            _logger = logger;
            _loggerFactory = loggerFactory;
            _kernel = CreateKernel();
        }

        private Kernel CreateKernel()
        {
            var builder = Kernel.CreateBuilder();
            
            builder.AddAzureOpenAIChatCompletion(
                deploymentName: _configuration["AzureOpenAI:DeploymentName"]!,
                endpoint: _configuration["AzureOpenAI:Endpoint"]!,
                apiKey: _configuration["AzureOpenAI:ApiKey"]!
            );
            
            return builder.Build();
        }

        public async Task<ScrumDeliverables> ProcessRequirementsAsync(
            string requirements, 
            CancellationToken cancellationToken = default)
        {
            _logger.LogInformation("🚀 Starting AI Scrum Team workflow...");
            
            // Create agents
            var agents = new Agent[]
            {
                ProductOwnerAgent.Create(_kernel),
                BusinessAnalystAgent.Create(_kernel),
                SolutionArchitectAgent.Create(_kernel),
                QATestAgent.Create(_kernel)
            };
            
            _logger.LogInformation($"✅ Created {agents.Length} agents");
            
            // Create conversation history to capture agent responses
            var conversationHistory = new ChatHistory();
            var agentResponseCount = new Dictionary<string, int>();
            var lastActivityTime = DateTime.UtcNow;
            
            // Enhanced response callback with detailed monitoring
            ValueTask ResponseCallback(ChatMessageContent response)
            {
                conversationHistory.Add(response);
                lastActivityTime = DateTime.UtcNow;
                
                // Track agent response counts
                if (!string.IsNullOrEmpty(response.AuthorName))
                {
                    agentResponseCount[response.AuthorName] = agentResponseCount.GetValueOrDefault(response.AuthorName) + 1;
                }
                
                // Log detailed response info
                var preview = response.Content?[..Math.Min(200, response.Content?.Length ?? 0)] ?? "";
                _logger.LogInformation($"🗣️ {response.AuthorName} (#{agentResponseCount.GetValueOrDefault(response.AuthorName)}): {preview}...");
                
                // Check for completion signals
                if (response.AuthorName == "QATester" && 
                    (response.Content?.Contains("TEST CASES COMPLETE", StringComparison.OrdinalIgnoreCase) == true ||
                     response.Content?.Contains("test cases complete", StringComparison.OrdinalIgnoreCase) == true))
                {
                    _logger.LogInformation("🎯 QA completion signal detected!");
                }
                
                return ValueTask.CompletedTask;
            }
            
            // Create group chat manager
            var manager = new ScrumGroupChatManager(
                _loggerFactory.CreateLogger<ScrumGroupChatManager>()
            );
            
            // Create group chat orchestration with response callback
            var orchestration = new GroupChatOrchestration(manager, agents)
            {
                ResponseCallback = ResponseCallback
            };
            
            // Create runtime
            var runtime = new InProcessRuntime();
            await runtime.StartAsync();
            
            _logger.LogInformation("🔄 Processing requirements through AI Scrum Team...");
            
            var deliverables = new ScrumDeliverables();
            
            try
            {
                // Execute the workflow with extended timeout
                var result = await orchestration.InvokeAsync(requirements, runtime);
                
                // Reduced timeout to 10 minutes for faster feedback
                var timeout = TimeSpan.FromMinutes(10);
                using var cts = CancellationTokenSource.CreateLinkedTokenSource(cancellationToken);
                cts.CancelAfter(timeout);
                
                _logger.LogInformation($"⏱️ Orchestration started with {timeout.TotalMinutes} minute timeout");
                
                // Start progress monitoring
                var progressTask = MonitorProgressAsync(conversationHistory, agentResponseCount, cts.Token);
                
                var finalOutput = await result.GetValueAsync(timeout, cts.Token);
                
                // Stop progress monitoring
                cts.Cancel();
                
                // Extract deliverables from conversation history
                ExtractDeliverables(conversationHistory, deliverables);
                
                _logger.LogInformation("✅ QA has completed all test cases. Workflow finished.");
            }
            catch (TimeoutException ex)
            {
                _logger.LogError("⏰ Orchestration timed out!");
                LogCurrentState(conversationHistory, agentResponseCount, lastActivityTime);
                
                // Still try to extract partial deliverables
                ExtractDeliverables(conversationHistory, deliverables);
                
                throw new InvalidOperationException(
                    $"AI Scrum Team workflow timed out. Last activity: {DateTime.UtcNow.Subtract(lastActivityTime).TotalSeconds:F1} seconds ago. " +
                    $"Total messages: {conversationHistory.Count}. Partial deliverables may be available.", ex);
            }
            catch (OperationCanceledException ex)
            {
                _logger.LogWarning("🛑 Orchestration was cancelled");
                LogCurrentState(conversationHistory, agentResponseCount, lastActivityTime);
                
                // Still try to extract partial deliverables
                ExtractDeliverables(conversationHistory, deliverables);
                
                throw new InvalidOperationException(
                    $"AI Scrum Team workflow was cancelled. Partial deliverables may be available.", ex);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "❌ Error during AI Scrum Team execution");
                LogCurrentState(conversationHistory, agentResponseCount, lastActivityTime);
                throw;
            }
            finally
            {
                await runtime.RunUntilIdleAsync();
            }
            
            _logger.LogInformation("🎉 AI Scrum Team workflow completed successfully!");
            return deliverables;
        }

        private void ExtractDeliverables(ChatHistory conversationHistory, ScrumDeliverables deliverables)
        {
            foreach (var message in conversationHistory)
            {
                if (string.IsNullOrEmpty(message.AuthorName) || message.AuthorName == "user")
                    continue;
                    
                var content = message.Content;
                
                switch (message.AuthorName)
                {
                    case "ProductOwner":
                        deliverables.ProductOwnerOutput = content;
                        break;
                    case "BusinessAnalyst":
                        deliverables.BusinessAnalystOutput = content;
                        break;
                    case "SolutionArchitect":
                        deliverables.SolutionArchitectOutput = content;
                        break;
                    case "QATester":
                        deliverables.QATestOutput = content;
                        break;
                }
            }
        }

        private void LogCurrentState(ChatHistory conversationHistory, Dictionary<string, int> agentResponseCount, DateTime lastActivityTime)
        {
            _logger.LogError($"📊 Final state: {conversationHistory.Count} messages");
            
            foreach (var kvp in agentResponseCount)
            {
                _logger.LogError($"  {kvp.Key}: {kvp.Value} responses");
            }
            
            _logger.LogError($"Last activity: {DateTime.UtcNow.Subtract(lastActivityTime).TotalSeconds:F1} seconds ago");
            
            // Log last few messages for debugging
            _logger.LogError("🔍 Last 5 messages:");
            foreach (var message in conversationHistory.TakeLast(5))
            {
                var preview = message.Content?[..Math.Min(200, message.Content?.Length ?? 0)] ?? "";
                _logger.LogError($"  {message.AuthorName}: {preview}...");
            }
        }

        private async Task MonitorProgressAsync(
            ChatHistory conversationHistory, 
            Dictionary<string, int> agentResponseCount, 
            CancellationToken cancellationToken)
        {
            var lastCount = 0;
            var stuckCheckCount = 0;
            
            try
            {
                while (!cancellationToken.IsCancellationRequested)
                {
                    await Task.Delay(45000, cancellationToken); // Check every 45 seconds
                    
                    var currentCount = conversationHistory.Count;
                    
                    if (currentCount == lastCount)
                    {
                        stuckCheckCount++;
                        _logger.LogWarning($"⚠️ No progress in 45 seconds (check #{stuckCheckCount})");
                        
                        if (stuckCheckCount >= 4) // 3 minutes without progress
                        {
                            _logger.LogError("🚨 No progress for 3+ minutes - possible deadlock!");
                            
                            // Log current state
                            var lastMessage = conversationHistory.LastOrDefault();
                            if (lastMessage != null)
                            {
                                _logger.LogError($"Last message from: {lastMessage.AuthorName}");
                                _logger.LogError($"Message preview: {lastMessage.Content?[..Math.Min(150, lastMessage.Content?.Length ?? 0)]}...");
                            }
                        }
                    }
                    else
                    {
                        stuckCheckCount = 0;
                        var qaCount = agentResponseCount.GetValueOrDefault("QATester", 0);
                        _logger.LogInformation($"📈 Progress: {currentCount} total messages (QA: {qaCount} responses)");
                    }
                    
                    lastCount = currentCount;
                }
            }
            catch (OperationCanceledException)
            {
                // Expected when cancellation is requested
            }
        }

        public async Task SaveDeliverablesAsync(ScrumDeliverables deliverables, string filePath)
        {
            var markdownContent = deliverables.GenerateMarkdownReport();
            await File.WriteAllTextAsync(filePath, markdownContent);
            _logger.LogInformation($"💾 Deliverables saved to: {filePath}");
        }
    }
}
```

## Step 6: Update Program.cs

Replace the contents of `Program.cs`:

```csharp
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;
using AIScrumTeam.Services;

namespace AIScrumTeam
{
    class Program
    {
        static async Task Main(string[] args)
        {
            // Build configuration
            var configuration = new ConfigurationBuilder()
                .SetBasePath(Directory.GetCurrentDirectory())
                .AddJsonFile("appsettings.json", optional: false, reloadOnChange: true)
                .AddUserSecrets<Program>(optional: true)
                .AddEnvironmentVariables()
                .Build();

            // Configure services
            var services = new ServiceCollection();
            services.AddSingleton<IConfiguration>(configuration);
            services.AddLogging(builder =>
            {
                builder.AddConfiguration(configuration.GetSection("Logging"));
                builder.AddConsole();
            });
            services.AddTransient<ScrumTeamOrchestrator>();

            // Build service provider
            using var serviceProvider = services.BuildServiceProvider();
            var logger = serviceProvider.GetRequiredService<ILogger<Program>>();
            
            try
            {
                logger.LogInformation("🚀 AI Scrum Team Application Starting...");
                
                // Get the orchestrator
                var orchestrator = serviceProvider.GetRequiredService<ScrumTeamOrchestrator>();
                
                // Sample requirements (you can replace this with file input or command line args)
                var sampleRequirements = """
                    We need a comprehensive e-learning platform that supports:
                    - Student enrollment and progress tracking
                    - Instructor course creation and management
                    - Interactive video lessons with embedded quizzes
                    - Discussion forums for each course
                    - Mobile application for offline learning
                    - Integration with popular payment gateways
                    - Multi-language support for global reach
                    - Advanced analytics dashboard for learning outcomes
                    - Certificate generation upon course completion
                    - Real-time notifications for assignments and announcements
                    """;
                
                // You can also load from file:
                // var requirements = await File.ReadAllTextAsync("requirements.txt");
                
                logger.LogInformation("📋 Processing requirements...");
                Console.WriteLine($"Requirements: {sampleRequirements[..100]}...");
                
                // Process requirements
                var deliverables = await orchestrator.ProcessRequirementsAsync(sampleRequirements);
                
                // Save deliverables
                var outputPath = Path.Combine(Directory.GetCurrentDirectory(), "scrum_deliverables.md");
                await orchestrator.SaveDeliverablesAsync(deliverables, outputPath);
                
                logger.LogInformation("✅ Application completed successfully!");
                Console.WriteLine($"\n🎉 Deliverables saved to: {outputPath}");
                Console.WriteLine("\nPress any key to exit...");
                Console.ReadKey();
            }
            catch (Exception ex)
            {
                logger.LogError(ex, "❌ Application failed");
                Console.WriteLine($"Error: {ex.Message}");
                Environment.Exit(1);
            }
        }
    }
}
```

### Alternative: Interactive Mode with Human Input

For a more interactive experience, you can modify the Program.cs to accept requirements from the user at runtime. Here are several approaches:

#### Option 1: Console Input

Replace the sample requirements section with interactive prompts:

```csharp
// Interactive requirements gathering
Console.WriteLine("🤖 AI Scrum Team - Interactive Requirements Gathering");
Console.WriteLine("=" * 60);
Console.WriteLine();

// Get project name
Console.Write("📋 Enter the project name: ");
var projectName = Console.ReadLine();

Console.WriteLine($"\n🎯 Let's gather requirements for: {projectName}");
Console.WriteLine("Please describe what you want to build. Include features, functionality, and any specific requirements.");
Console.WriteLine("When finished, press Enter twice to continue.\n");

// Multi-line input for requirements
var requirements = new StringBuilder();
string line;
int emptyLineCount = 0;

Console.WriteLine("💡 Tip: Press Enter twice when you're done entering requirements");
Console.Write("Requirements: ");

while (emptyLineCount < 2)
{
    line = Console.ReadLine();
    
    if (string.IsNullOrWhiteSpace(line))
    {
        emptyLineCount++;
        if (emptyLineCount == 1)
        {
            Console.WriteLine("Press Enter again to finish, or continue typing...");
        }
    }
    else
    {
        emptyLineCount = 0;
        requirements.AppendLine(line);
    }
}

var userRequirements = requirements.ToString().Trim();

if (string.IsNullOrWhiteSpace(userRequirements))
{
    Console.WriteLine("⚠️ No requirements provided. Using sample requirements...");
    userRequirements = """
        We need a comprehensive e-learning platform that supports:
        - Student enrollment and progress tracking
        - Instructor course creation and management
        - Interactive video lessons with embedded quizzes
        - Discussion forums for each course
        - Mobile application for offline learning
        - Integration with popular payment gateways
        - Multi-language support for global reach
        - Advanced analytics dashboard for learning outcomes
        - Certificate generation upon course completion
        - Real-time notifications for assignments and announcements
        """;
}

Console.WriteLine($"\n📋 Processing requirements for: {projectName}");
Console.WriteLine("🔄 AI Scrum Team is analyzing your requirements...\n");
```

#### Option 2: File Input with User Selection

Add file input capabilities:

```csharp
// File or interactive input selection
Console.WriteLine("🤖 AI Scrum Team - Requirements Input Options");
Console.WriteLine("=" * 50);
Console.WriteLine("1. Enter requirements interactively");
Console.WriteLine("2. Load requirements from file");
Console.WriteLine("3. Use sample requirements");
Console.Write("\nSelect option (1-3): ");

var option = Console.ReadKey().KeyChar;
Console.WriteLine("\n");

string userRequirements = string.Empty;
string projectName = "AI Scrum Project";

switch (option)
{
    case '1':
        // Interactive input (use code from Option 1 above)
        Console.Write("📋 Enter the project name: ");
        projectName = Console.ReadLine() ?? "Interactive Project";
        
        Console.WriteLine("\nEnter your requirements (press Enter twice when done):");
        var requirements = new StringBuilder();
        string line;
        int emptyLineCount = 0;

        while (emptyLineCount < 2)
        {
            line = Console.ReadLine();
            if (string.IsNullOrWhiteSpace(line))
                emptyLineCount++;
            else
            {
                emptyLineCount = 0;
                requirements.AppendLine(line);
            }
        }
        userRequirements = requirements.ToString().Trim();
        break;

    case '2':
        // File input
        Console.Write("📁 Enter the path to your requirements file: ");
        var filePath = Console.ReadLine();
        
        if (File.Exists(filePath))
        {
            userRequirements = await File.ReadAllTextAsync(filePath);
            projectName = Path.GetFileNameWithoutExtension(filePath);
            Console.WriteLine($"✅ Loaded requirements from: {filePath}");
        }
        else
        {
            Console.WriteLine("❌ File not found. Using sample requirements...");
            goto case '3';
        }
        break;

    case '3':
    default:
        // Sample requirements
        projectName = "E-Learning Platform";
        userRequirements = """
            We need a comprehensive e-learning platform that supports:
            - Student enrollment and progress tracking
            - Instructor course creation and management
            - Interactive video lessons with embedded quizzes
            - Discussion forums for each course
            - Mobile application for offline learning
            - Integration with popular payment gateways
            - Multi-language support for global reach
            - Advanced analytics dashboard for learning outcomes
            - Certificate generation upon course completion
            - Real-time notifications for assignments and announcements
            """;
        Console.WriteLine("📋 Using sample e-learning platform requirements");
        break;
}

Console.WriteLine($"\n🎯 Project: {projectName}");
Console.WriteLine("🔄 AI Scrum Team is processing your requirements...\n");
```

#### Option 3: Command Line Arguments

Support command line arguments for automation:

```csharp
static async Task Main(string[] args)
{
    // ... configuration setup ...

    try
    {
        logger.LogInformation("🚀 AI Scrum Team Application Starting...");
        
        var orchestrator = serviceProvider.GetRequiredService<ScrumTeamOrchestrator>();
        
        string requirements;
        string projectName = "AI Scrum Project";
        
        // Check command line arguments
        if (args.Length > 0)
        {
            if (args[0] == "--file" && args.Length > 1)
            {
                // Load from file: dotnet run --file requirements.txt
                var filePath = args[1];
                if (File.Exists(filePath))
                {
                    requirements = await File.ReadAllTextAsync(filePath);
                    projectName = Path.GetFileNameWithoutExtension(filePath);
                    logger.LogInformation($"📁 Loaded requirements from: {filePath}");
                }
                else
                {
                    Console.WriteLine($"❌ File not found: {filePath}");
                    return;
                }
            }
            else if (args[0] == "--interactive")
            {
                // Interactive mode: dotnet run --interactive
                requirements = GetInteractiveRequirements(out projectName);
            }
            else
            {
                // Direct input: dotnet run "Build a chat application with real-time messaging"
                requirements = string.Join(" ", args);
                projectName = "Command Line Project";
                logger.LogInformation("📝 Using command line requirements");
            }
        }
        else
        {
            // No arguments - default to interactive mode
            requirements = GetInteractiveRequirements(out projectName);
        }

        // Proceed with processing...
        logger.LogInformation($"📋 Processing requirements for: {projectName}");
        Console.WriteLine($"Requirements Preview: {requirements[..Math.Min(100, requirements.Length)]}...");
        
        var deliverables = await orchestrator.ProcessRequirementsAsync(requirements);
        
        // Save with project-specific filename
        var safeProjectName = string.Join("_", projectName.Split(Path.GetInvalidFileNameChars()));
        var outputPath = Path.Combine(Directory.GetCurrentDirectory(), $"{safeProjectName}_deliverables.md");
        await orchestrator.SaveDeliverablesAsync(deliverables, outputPath);
        
        logger.LogInformation("✅ Application completed successfully!");
        Console.WriteLine($"\n🎉 Deliverables saved to: {outputPath}");
        Console.WriteLine("\nPress any key to exit...");
        Console.ReadKey();
    }
    catch (Exception ex)
    {
        logger.LogError(ex, "❌ Application failed");
        Console.WriteLine($"Error: {ex.Message}");
        Environment.Exit(1);
    }
}

private static string GetInteractiveRequirements(out string projectName)
{
    Console.WriteLine("🤖 AI Scrum Team - Interactive Mode");
    Console.WriteLine("=" * 40);
    
    Console.Write("📋 Project Name: ");
    projectName = Console.ReadLine() ?? "Interactive Project";
    
    Console.WriteLine("\n💡 Describe your project requirements:");
    Console.WriteLine("Include features, functionality, integrations, and constraints.");
    Console.WriteLine("Press Enter twice when finished.\n");
    
    var requirements = new StringBuilder();
    string line;
    int emptyLineCount = 0;

    while (emptyLineCount < 2)
    {
        line = Console.ReadLine();
        
        if (string.IsNullOrWhiteSpace(line))
        {
            emptyLineCount++;
        }
        else
        {
            emptyLineCount = 0;
            requirements.AppendLine(line);
        }
    }

    var result = requirements.ToString().Trim();
    
    if (string.IsNullOrWhiteSpace(result))
    {
        Console.WriteLine("⚠️ No requirements entered. Please try again.");
        return GetInteractiveRequirements(out projectName);
    }
    
    return result;
}
```

#### Usage Examples

With these modifications, users can run the application in various ways:

```bash
# Interactive mode (default)
dotnet run

# Interactive mode (explicit)
dotnet run --interactive

# Load from file
dotnet run --file my-requirements.txt

# Direct command line input
dotnet run "Build a social media platform with user authentication, posts, comments, and real-time notifications"
```

#### Creating a Requirements Template

You can also provide a template file for users:

Create `requirements-template.txt`:

```text
Project Name: [Enter your project name]

Project Overview:
[Describe what you want to build in 2-3 sentences]

Core Features:
- [Feature 1]
- [Feature 2]
- [Feature 3]

User Types:
- [User type 1: description]
- [User type 2: description]

Technical Requirements:
- [Technology preference, if any]
- [Performance requirements]
- [Security requirements]

Integration Requirements:
- [External services to integrate]
- [APIs to consume/provide]

Constraints:
- [Budget constraints]
- [Timeline constraints]
- [Technical constraints]

Success Criteria:
- [How will you measure success?]
```



## Step 7: Build and Test

### Build the Application

```bash
dotnet build
```

### Run the Application

```bash
dotnet run
```

### Custom Requirements Example

Create a `CustomRequirementsRunner.cs` for testing different scenarios:

```csharp
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Logging;
using AIScrumTeam.Services;

namespace AIScrumTeam
{
    public class CustomRequirementsRunner
    {
        public static async Task RunCustomScenarioAsync()
        {
            var configuration = new ConfigurationBuilder()
                .SetBasePath(Directory.GetCurrentDirectory())
                .AddJsonFile("appsettings.json")
                .Build();

            var services = new ServiceCollection();
            services.AddSingleton<IConfiguration>(configuration);
            services.AddLogging(builder => builder.AddConsole());
            services.AddTransient<ScrumTeamOrchestrator>();

            using var serviceProvider = services.BuildServiceProvider();
            var orchestrator = serviceProvider.GetRequiredService<ScrumTeamOrchestrator>();

            var customRequirements = """
                Build a modern healthcare patient portal with the following capabilities:
                - Patient registration and profile management
                - Appointment scheduling with healthcare providers
                - Secure messaging between patients and doctors
                - Medical record access and download
                - Prescription refill requests
                - Insurance verification and billing
                - Telehealth video consultation support
                - Mobile app with biometric authentication
                - HIPAA compliance and audit logging
                - Integration with electronic health record (EHR) systems
                """;

            Console.WriteLine("🏥 Processing Healthcare Portal Requirements...");
            
            var deliverables = await orchestrator.ProcessRequirementsAsync(customRequirements);
            await orchestrator.SaveDeliverablesAsync(deliverables, "healthcare_portal_deliverables.md");
            
            Console.WriteLine("✅ Healthcare portal analysis complete!");
        }
    }
}
```

## Step 8: Advanced Features

### Adding Validation and Quality Checks

Create `Services/DeliverableValidator.cs`:

```csharp
using AIScrumTeam.Models;

namespace AIScrumTeam.Services
{
    public class DeliverableValidator
    {
        public ValidationResult ValidateDeliverables(ScrumDeliverables deliverables)
        {
            var result = new ValidationResult();
            
            // Check for required sections
            if (string.IsNullOrEmpty(deliverables.ProductOwnerOutput))
                result.Errors.Add("Missing Product Owner SRS output");
                
            if (string.IsNullOrEmpty(deliverables.BusinessAnalystOutput))
                result.Errors.Add("Missing Business Analyst user stories");
                
            if (string.IsNullOrEmpty(deliverables.SolutionArchitectOutput))
                result.Errors.Add("Missing Solution Architect design");
                
            if (string.IsNullOrEmpty(deliverables.QATestOutput))
                result.Errors.Add("Missing QA test cases");
            
            // Validate content quality
            ValidateProductOwnerOutput(deliverables.ProductOwnerOutput, result);
            ValidateBusinessAnalystOutput(deliverables.BusinessAnalystOutput, result);
            ValidateSolutionArchitectOutput(deliverables.SolutionArchitectOutput, result);
            ValidateQAOutput(deliverables.QATestOutput, result);
            
            result.IsValid = !result.Errors.Any();
            return result;
        }
        
        private void ValidateProductOwnerOutput(string? output, ValidationResult result)
        {
            if (string.IsNullOrEmpty(output)) return;
            
            var requiredSections = new[] { "Project Overview", "Business Requirements", "System Requirements" };
            foreach (var section in requiredSections)
            {
                if (!output.Contains(section, StringComparison.OrdinalIgnoreCase))
                    result.Warnings.Add($"SRS missing recommended section: {section}");
            }
        }
        
        private void ValidateBusinessAnalystOutput(string? output, ValidationResult result)
        {
            if (string.IsNullOrEmpty(output)) return;
            
            if (!output.Contains("As a", StringComparison.OrdinalIgnoreCase))
                result.Warnings.Add("Business Analyst output may be missing proper user story format");
                
            if (!output.Contains("Given", StringComparison.OrdinalIgnoreCase) ||
                !output.Contains("When", StringComparison.OrdinalIgnoreCase) ||
                !output.Contains("Then", StringComparison.OrdinalIgnoreCase))
                result.Warnings.Add("Business Analyst output may be missing Given/When/Then acceptance criteria");
        }
        
        private void ValidateSolutionArchitectOutput(string? output, ValidationResult result)
        {
            if (string.IsNullOrEmpty(output)) return;
            
            var requiredSections = new[] { "Components", "Architecture", "Infrastructure" };
            foreach (var section in requiredSections)
            {
                if (!output.Contains(section, StringComparison.OrdinalIgnoreCase))
                    result.Warnings.Add($"Architecture output missing recommended section: {section}");
            }
        }
        
        private void ValidateQAOutput(string? output, ValidationResult result)
        {
            if (string.IsNullOrEmpty(output)) return;
            
            if (!output.Contains("Given", StringComparison.OrdinalIgnoreCase) ||
                !output.Contains("When", StringComparison.OrdinalIgnoreCase) ||
                !output.Contains("Then", StringComparison.OrdinalIgnoreCase))
                result.Warnings.Add("QA output may be missing proper Gherkin format");
                
            if (!output.Contains("Test cases complete", StringComparison.OrdinalIgnoreCase))
                result.Warnings.Add("QA output missing completion confirmation");
        }
    }
    
    public class ValidationResult
    {
        public bool IsValid { get; set; }
        public List<string> Errors { get; set; } = new();
        public List<string> Warnings { get; set; } = new();
    }
}
```

### Enhanced Configuration with Options Pattern

Create `Configuration/AIScrumTeamOptions.cs`:

```csharp
namespace AIScrumTeam.Configuration
{
    public class AIScrumTeamOptions
    {
        public const string SectionName = "AIScrumTeam";
        
        public AzureOpenAIOptions AzureOpenAI { get; set; } = new();
        public WorkflowOptions Workflow { get; set; } = new();
        public OutputOptions Output { get; set; } = new();
    }
    
    public class AzureOpenAIOptions
    {
        public string Endpoint { get; set; } = string.Empty;
        public string ApiKey { get; set; } = string.Empty;
        public string DeploymentName { get; set; } = string.Empty;
        public int MaxTokens { get; set; } = 4000;
        public double Temperature { get; set; } = 0.7;
    }
    
    public class WorkflowOptions
    {
        public int MaxRounds { get; set; } = 15;
        public int TimeoutMinutes { get; set; } = 30;
        public bool EnableValidation { get; set; } = true;
        public bool EnableLogging { get; set; } = true;
    }
    
    public class OutputOptions
    {
        public string DefaultOutputPath { get; set; } = "deliverables";
        public string[] SupportedFormats { get; set; } = { "markdown", "json", "html" };
        public bool IncludeTimestamps { get; set; } = true;
        public bool IncludeMetadata { get; set; } = true;
    }
}
```

### Adding Multiple Output Formats

Create `Services/OutputFormatter.cs`:

```csharp
using AIScrumTeam.Models;
using System.Text.Json;

namespace AIScrumTeam.Services
{
    public class OutputFormatter
    {
        public async Task SaveAsMarkdownAsync(ScrumDeliverables deliverables, string filePath)
        {
            var content = deliverables.GenerateMarkdownReport();
            await File.WriteAllTextAsync(filePath, content);
        }
        
        public async Task SaveAsJsonAsync(ScrumDeliverables deliverables, string filePath)
        {
            var options = new JsonSerializerOptions { WriteIndented = true };
            var json = JsonSerializer.Serialize(deliverables, options);
            await File.WriteAllTextAsync(filePath, json);
        }
        
        public async Task SaveAsHtmlAsync(ScrumDeliverables deliverables, string filePath)
        {
            var html = GenerateHtmlReport(deliverables);
            await File.WriteAllTextAsync(filePath, html);
        }
        
        private string GenerateHtmlReport(ScrumDeliverables deliverables)
        {
            return $"""
                <!DOCTYPE html>
                <html>
                <head>
                    <title>Scrum AI Team Deliverables</title>
                    <style>
                        body {{ font-family: Arial, sans-serif; margin: 20px; }}
                        .section {{ margin-bottom: 30px; }}
                        .agent-name {{ color: #2E86AB; font-size: 24px; font-weight: bold; }}
                        .content {{ background-color: #f5f5f5; padding: 15px; border-radius: 5px; }}
                        pre {{ white-space: pre-wrap; }}
                    </style>
                </head>
                <body>
                    <h1>📋 Scrum AI Team Deliverables</h1>
                    <p><strong>Generated:</strong> {deliverables.CreatedAt:yyyy-MM-dd HH:mm:ss} UTC</p>
                    
                    <div class="section">
                        <div class="agent-name">ProductOwner</div>
                        <div class="content"><pre>{deliverables.ProductOwnerOutput}</pre></div>
                    </div>
                    
                    <div class="section">
                        <div class="agent-name">BusinessAnalyst</div>
                        <div class="content"><pre>{deliverables.BusinessAnalystOutput}</pre></div>
                    </div>
                    
                    <div class="section">
                        <div class="agent-name">SolutionArchitect</div>
                        <div class="content"><pre>{deliverables.SolutionArchitectOutput}</pre></div>
                    </div>
                    
                    <div class="section">
                        <div class="agent-name">QATester</div>
                        <div class="content"><pre>{deliverables.QATestOutput}</pre></div>
                    </div>
                </body>
                </html>
                """;
        }
    }
}
```

## Expected Output Structure

When you run the AI Scrum Team, you'll get comprehensive deliverables including:

### 1. System Requirements Specification (SRS)
- Project overview with system name and purpose
- Business requirements (BR1, BR2, etc.)
- Functional and non-functional requirements
- Constraints, assumptions, and business rules

### 2. User Stories and Acceptance Criteria
- Stories organized into logical epics
- "As a...I want...So that..." format
- Comprehensive Given/When/Then acceptance criteria
- Multiple scenarios per story

### 3. Solution Architecture
- Major system components
- Technology recommendations
- System interactions and workflows
- Infrastructure requirements
- Technical dependencies and risks

### 4. Test Cases in Gherkin Format
- Organized by feature
- Happy path, error handling, and edge cases
- Security and performance considerations
- Complete Gherkin syntax formatting

## 🔧 Better Termination Strategies (Microsoft Best Practices)

**Instead of relying on LLM text output like "TEST CASES COMPLETE", use these more reliable approaches:**

### ✅ **Approach 1: MaximumInvocationCount (Simplest)**
The most reliable approach is using the built-in `MaximumInvocationCount` property:

```csharp
public ScrumGroupChatManager()
{
    // Auto-terminate after reasonable number of interactions
    MaximumInvocationCount = 8; // 4 agents × 2 rounds = sensible limit
}
```

### ✅ **Approach 2: Agent Response Counting**
Track responses and terminate when all agents have contributed:

```csharp
public override ValueTask<GroupChatManagerResult<bool>> ShouldTerminate(ChatHistory history, CancellationToken cancellationToken = default)
{
    // Count responses by each agent
    var poCount = history.Count(m => m.AuthorName == "ProductOwner");
    var baCount = history.Count(m => m.AuthorName == "BusinessAnalyst");
    var saCount = history.Count(m => m.AuthorName == "SolutionArchitect");
    var qaCount = history.Count(m => m.AuthorName == "QATester");
    
    // Terminate when all agents have contributed
    if (poCount >= 1 && baCount >= 1 && saCount >= 1 && qaCount >= 1)
    {
        return ValueTask.FromResult(new GroupChatManagerResult<bool>(true) 
        { 
            Reason = "All agents have provided their contributions" 
        });
    }
    
    return ValueTask.FromResult(new GroupChatManagerResult<bool>(false));
}
```

### ✅ **Approach 3: Content Analysis (Most Sophisticated)**
Analyze message content for semantic completion:

```csharp
public override ValueTask<GroupChatManagerResult<bool>> ShouldTerminate(ChatHistory history, CancellationToken cancellationToken = default)
{
    // Check for substantial QA content
    var lastFewMessages = history.TakeLast(3).ToList();
    bool hasQAContent = lastFewMessages.Any(m => 
        m.Content?.Length > 100 && // Substantial content
        (m.Content.Contains("test", StringComparison.OrdinalIgnoreCase) ||
         m.Content.Contains("scenario", StringComparison.OrdinalIgnoreCase) ||
         m.Content.Contains("validation", StringComparison.OrdinalIgnoreCase)));
         
    if (hasQAContent && history.Count >= 6) // Reasonable conversation + QA content
    {
        return ValueTask.FromResult(new GroupChatManagerResult<bool>(true) 
        { 
            Reason = "QA content detected, workflow appears complete" 
        });
    }
    
    return ValueTask.FromResult(new GroupChatManagerResult<bool>(false));
}
```

### ✅ **Approach 4: RegexTerminationStrategy (Built-in)**
Use Semantic Kernel's built-in regex termination:

```csharp
// In older AgentGroupChat (deprecated but shows concept)
var terminationStrategy = new RegexTerminationStrategy(
    new Regex(@"\b(complete|finished|done)\b", RegexOptions.IgnoreCase),
    agentName: "QATester"); // Only terminate on QATester messages
```

### ✅ **Approach 5: KernelFunctionTerminationStrategy (AI-Powered)**
Let AI decide when to terminate:

```csharp
var terminationFunction = KernelFunctionFromPrompt.Create(
    @"Analyze the conversation and determine if the Scrum workflow is complete.
      Look for: Product Owner requirements, Business Analysis, Architecture design, and QA test scenarios.
      Respond with 'TERMINATE' if all components are present, otherwise 'CONTINUE'.");

var terminationStrategy = new KernelFunctionTerminationStrategy(
    terminationFunction, 
    kernel)
{
    ResultParser = (result) => result.GetValue<string>()?.Contains("TERMINATE") == true
};
```

### 🎯 **Recommended Approach for This HOL**

The current implementation uses **Approach 2** (agent response counting) which is much more reliable than text-based termination. Here's why this is better:

**❌ Problems with text-based termination:**
- LLMs don't always follow instructions exactly
- Formatting can vary ("Test cases complete" vs "TEST CASES COMPLETE")
- Model temperature affects consistency
- Can cause infinite loops if string never appears

**✅ Benefits of response counting:**
- Deterministic and reliable
- Not dependent on LLM text generation
- Guarantees each agent contributes
- Prevents infinite loops
- Easy to debug and monitor

The current code automatically terminates when QATester responds, ensuring the workflow completes successfully without relying on specific text output.


### Common Issues

1. **"'IKernelBuilder' does not contain a definition for 'AddAzureOpenAIChatCompletion'" Error**

   This error occurs when the required NuGet package or using statement is missing. To fix:

   ```bash
   # Install the Azure OpenAI connector package
   dotnet add package Microsoft.SemanticKernel.Connectors.AzureOpenAI
   ```

   Add the using statement to your file:
   ```csharp
   using Microsoft.SemanticKernel.Connectors.AzureOpenAI;
   ```

   The `AddAzureOpenAIChatCompletion` method is an extension method provided by the AzureOpenAI connector package.

2. **"'ILogger&lt;T&gt;' does not contain a definition for 'CreateLogger'" Error**

   This error occurs because `ILogger<T>` doesn't have a `CreateLogger` method. You need `ILoggerFactory` instead. To fix:

   ```csharp
   // Add ILoggerFactory to the constructor
   public ScrumTeamOrchestrator(IConfiguration configuration, ILogger<ScrumTeamOrchestrator> logger, ILoggerFactory loggerFactory)
   {
       _loggerFactory = loggerFactory;
       // ... other code
   }

   // Use ILoggerFactory to create loggers
   var manager = new ScrumGroupChatManager(
       _loggerFactory.CreateLogger<ScrumGroupChatManager>()
   );
   ```

   The `ILoggerFactory` is automatically registered by the `AddLogging()` call in Program.cs.

3. **"'OrchestrationResult&lt;string&gt;' does not contain a definition for 'ConversationHistory'" Error**

   This error occurs because the new GroupChatOrchestration pattern uses a `ResponseCallback` to capture conversation history instead of a property on the result. To fix:

   ```csharp
   // Create conversation history to capture agent responses
   var conversationHistory = new ChatHistory();
   
   // Create response callback to capture agent messages
   ValueTask ResponseCallback(ChatMessageContent response)
   {
       conversationHistory.Add(response);
       return ValueTask.CompletedTask;
   }
   
   // Create orchestration with response callback
   var orchestration = new GroupChatOrchestration(manager, agents)
   {
       ResponseCallback = ResponseCallback
   };
   
   // After execution, use conversationHistory instead of result.ConversationHistory
   foreach (var message in conversationHistory)
   {
       // Process messages...
   }
   ```

4. **🚨 QATester Hanging / Never Returning Response (CRITICAL ISSUE)**

   This is the most common issue reported. The QATester agent gets stuck and never returns a response, causing the workflow to timeout. Here are the causes and solutions:

   **Root Causes:**
   - Inconsistent termination signal detection
   - Agent selection logic conflicts
   - Timeout values too low for complex interactions
   - Missing completion signal in QA agent instructions

   **Solution 1: Update QA Agent Instructions (REQUIRED)**
   
   The QA agent must use a consistent, uppercase completion signal:

   ```csharp
   public static ChatCompletionAgent Create(Kernel kernel)
   {
       return new ChatCompletionAgent()
       {
           Name = "QATester",
           Instructions = """
               // ... existing instructions ...
               
               IMPORTANT: After completing all test cases, you MUST end your response with exactly this phrase:
               "TEST CASES COMPLETE"
               
               This signals that your testing analysis is finished and the workflow should terminate.
               """,
           Kernel = kernel,
           Description = "Generates comprehensive test scenarios and test cases in Gherkin format"
       };
   }
   ```

   **Solution 2: Improve GroupChatManager Logic (REQUIRED)**
   
   The termination logic needs to be more robust and handle edge cases:

   ```csharp
   public class ScrumGroupChatManager : GroupChatManager
   {
       private bool _isCompleted = false;
       
       public override ValueTask<GroupChatManagerResult<string>> SelectNextAgent(ChatHistory history, GroupChatTeam team, CancellationToken cancellationToken = default)
       {
           // Check for completion first to avoid further agent selection
           if (_isCompleted)
           {
               return ValueTask.FromResult(new GroupChatManagerResult<string>("") { Reason = "Workflow complete" });
           }

           var lastMessage = history.LastOrDefault();
           var lastAgentName = lastMessage?.AuthorName;
           
           // Check if QA has completed their work
           if (lastAgentName == "QATester" && HasQACompleted(history))
           {
               _isCompleted = true;
               return ValueTask.FromResult(new GroupChatManagerResult<string>("") { Reason = "QA work complete" });
           }
           
           // Rest of agent selection logic...
       }
       
       private bool HasQACompleted(ChatHistory history)
       {
           var qaMessages = history
               .Where(m => m.AuthorName == "QATester")
               .Select(m => m.Content ?? "")
               .ToList();
               
           return qaMessages.Any(content => 
               content.Contains("TEST CASES COMPLETE", StringComparison.OrdinalIgnoreCase) ||
               content.Contains("test cases complete", StringComparison.OrdinalIgnoreCase));
       }
   }
   ```

   **Solution 3: Increase Timeouts and Add Monitoring (RECOMMENDED)**
   
   ```csharp
   public async Task<ScrumDeliverables> ProcessRequirementsAsync(string requirements, CancellationToken cancellationToken = default)
   {
       // Increased timeout to 30 minutes
       var timeout = TimeSpan.FromMinutes(30);
       using var cts = CancellationTokenSource.CreateLinkedTokenSource(cancellationToken);
       cts.CancelAfter(timeout);
       
       // Add progress monitoring
       var progressTask = MonitorProgressAsync(conversationHistory, agentResponseCount, cts.Token);
       
       var finalOutput = await result.GetValueAsync(timeout, cts.Token);
   }
   ```

   **Solution 4: Add Fallback Recovery Logic (RECOMMENDED)**
   
   ```csharp
   public override ValueTask<GroupChatManagerResult<bool>> ShouldTerminate(ChatHistory history, CancellationToken cancellationToken = default)
   {
       // Multiple termination conditions for robustness
       
       // 1. Check QA response count (safety valve)
       var qaResponseCount = history.Count(m => m.AuthorName == "QATester");
       if (qaResponseCount >= 3)
       {
           _logger.LogWarning($"🛑 Force terminating: QA has responded {qaResponseCount} times");
           return ValueTask.FromResult(new GroupChatManagerResult<bool>(true) { Reason = "QA response limit reached" });
       }
       
       // 2. Check for completion signals
       var isQAComplete = HasQACompleted(history);
       if (isQAComplete)
       {
           return ValueTask.FromResult(new GroupChatManagerResult<bool>(true) { Reason = "QA completed" });
       }
       
       // 3. Other safety conditions...
   }
   ```

   **Debugging Steps:**
   1. Enable debug logging to see agent responses
   2. Check if QA agent is receiving proper context from previous agents
   3. Verify Azure OpenAI service limits and quotas
   4. Monitor for token limit issues
   5. Check network connectivity and service availability

5. **"Orchestration did not complete within the allowed duration" Timeout Error**

   This error occurs when the AI agents don't complete their workflow within the timeout period. Common causes and solutions:

   **a) QA Agent not signaling completion properly:**
   ```csharp
   // Ensure QA Agent instructions include clear completion signal
   Instructions = """
       // ... other instructions ...
       
       After completing all test cases, end your response with:
       "TEST CASES COMPLETE"
       
       This signals that your testing analysis is finished.
       """,
   ```

   **b) Increase timeout duration and add better logging:**
   ```csharp
   public async Task<ScrumDeliverables> ProcessRequirementsAsync(
       string requirements, 
       CancellationToken cancellationToken = default)
   {
       // ... existing code ...
       
       try
       {
           // Execute with longer timeout and better monitoring
           var result = await orchestration.InvokeAsync(requirements, runtime);
           
           // Increase timeout to 30 minutes and add cancellation token support
           using var cts = CancellationTokenSource.CreateLinkedTokenSource(cancellationToken);
           cts.CancelAfter(TimeSpan.FromMinutes(30));
           
           var finalOutput = await result.GetValueAsync(TimeSpan.FromMinutes(30), cts.Token);
           
           _logger.LogInformation("✅ Orchestration completed successfully");
           
           // ... rest of the method ...
       }
       catch (TimeoutException ex)
       {
           _logger.LogError("⏰ Orchestration timed out. Last conversation state:");
           
           // Log current conversation state for debugging
           foreach (var message in conversationHistory.TakeLast(5))
           {
               _logger.LogError($"  {message.AuthorName}: {message.Content?[..Math.Min(200, message.Content?.Length ?? 0)]}...");
           }
           
           throw new InvalidOperationException(
               "AI Scrum Team workflow timed out. This may indicate agents are stuck in a loop or termination condition not met.", ex);
       }
   }
   ```

6. **NuGet Package Conflicts**

```bash
dotnet clean
dotnet restore
dotnet build
```

7. **Azure OpenAI Connection Issues**

```csharp
// Test connection in Program.cs
var testKernel = Kernel.CreateBuilder()
    .AddAzureOpenAIChatCompletion(deploymentName, endpoint, apiKey)
    .Build();
    
var chatService = testKernel.GetRequiredService<IChatCompletionService>();
Console.WriteLine("✅ Azure OpenAI connection successful");
```

8. **Configuration Problems**

- Verify appsettings.json format
- Check environment variables
- Validate Azure OpenAI credentials

### Debug Mode

Enable detailed logging in `appsettings.json`:

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Debug",
      "Microsoft.SemanticKernel": "Information",
      "AIScrumTeam": "Debug"
    }
  }
}
```

### Monitoring Orchestration Progress

To debug timeout issues, add real-time monitoring to your ScrumTeamOrchestrator:

```csharp
public async Task<ScrumDeliverables> ProcessRequirementsAsync(
    string requirements, 
    CancellationToken cancellationToken = default)
{
    _logger.LogInformation("🚀 Starting AI Scrum Team workflow...");
    
    // Create agents with enhanced logging
    var agents = new Agent[]
    {
        ProductOwnerAgent.Create(_kernel),
        BusinessAnalystAgent.Create(_kernel),
        SolutionArchitectAgent.Create(_kernel),
        QATestAgent.Create(_kernel)
    };
    
    _logger.LogInformation($"✅ Created {agents.Length} agents");
    
    // Create conversation history with monitoring
    var conversationHistory = new ChatHistory();
    var agentResponseCount = new Dictionary<string, int>();
    var lastActivityTime = DateTime.UtcNow;
    
    // Enhanced response callback with detailed monitoring
    ValueTask ResponseCallback(ChatMessageContent response)
    {
        conversationHistory.Add(response);
        lastActivityTime = DateTime.UtcNow;
        
        // Track agent response counts
        if (!string.IsNullOrEmpty(response.AuthorName))
        {
            agentResponseCount[response.AuthorName] = agentResponseCount.GetValueOrDefault(response.AuthorName) + 1;
        }
        
        // Log detailed response info
        var preview = response.Content?[..Math.Min(150, response.Content?.Length ?? 0)] ?? "";
        _logger.LogInformation($"🗣️ {response.AuthorName} (#{agentResponseCount.GetValueOrDefault(response.AuthorName)}): {preview}...");
        
        // Log current state
        var totalMessages = conversationHistory.Count;
        var uniqueAgents = agentResponseCount.Keys.Count;
        _logger.LogInformation($"📊 Progress: {totalMessages} messages, {uniqueAgents} agents active");
        
        // Check for completion signals
        if (response.AuthorName == "QATester" && 
            response.Content?.Contains("Test cases complete", StringComparison.OrdinalIgnoreCase) == true)
        {
            _logger.LogInformation("🎯 QA completion signal detected!");
        }
        
        // Warn if conversation is getting long
        if (totalMessages > 20)
        {
            _logger.LogWarning($"⚠️ Long conversation detected ({totalMessages} messages). Check termination logic.");
        }
        
        return ValueTask.CompletedTask;
    }
    
    // Create manager with enhanced logging
    var manager = new ScrumGroupChatManager(_loggerFactory.CreateLogger<ScrumGroupChatManager>());
    
    // Create orchestration
    var orchestration = new GroupChatOrchestration(manager, agents)
    {
        ResponseCallback = ResponseCallback
    };
    
    // Create runtime
    var runtime = new InProcessRuntime();
    await runtime.StartAsync();
    
    _logger.LogInformation("🔄 Processing requirements through AI Scrum Team...");
    
    var deliverables = new ScrumDeliverables();
    
    try
    {
        // Start orchestration
        var result = await orchestration.InvokeAsync(requirements, runtime);
        
        // Monitor progress with timeout
        var timeout = TimeSpan.FromMinutes(20);
        _logger.LogInformation($"⏱️ Orchestration started with {timeout.TotalMinutes} minute timeout");
        
        // Create cancellation token for timeout
        using var cts = CancellationTokenSource.CreateLinkedTokenSource(cancellationToken);
        cts.CancelAfter(timeout);
        
        // Monitor progress in background
        var progressTask = MonitorProgressAsync(conversationHistory, agentResponseCount, cts.Token);
        
        // Wait for completion
        var finalOutput = await result.GetValueAsync(timeout, cts.Token);
        
        _logger.LogInformation("✅ Orchestration completed successfully");
        
        // Stop progress monitoring
        cts.Cancel();
        
        // Process results...
        foreach (var message in conversationHistory)
        {
            // ... existing message processing logic ...
        }
        
        _logger.LogInformation("✅ QA has completed all test cases. Workflow finished.");
    }
    catch (TimeoutException ex)
    {
        _logger.LogError("⏰ Orchestration timed out!");
        _logger.LogError($"📊 Final state: {conversationHistory.Count} messages");
        
        foreach (var kvp in agentResponseCount)
        {
            _logger.LogError($"  {kvp.Key}: {kvp.Value} responses");
        }
        
        // Log last few messages for debugging
        _logger.LogError("🔍 Last 5 messages:");
        foreach (var message in conversationHistory.TakeLast(5))
        {
            var preview = message.Content?[..Math.Min(200, message.Content?.Length ?? 0)] ?? "";
            _logger.LogError($"  {message.AuthorName}: {preview}...");
        }
        
        throw new InvalidOperationException(
            $"AI Scrum Team workflow timed out after {timeout.TotalMinutes} minutes. " +
            $"Last activity: {DateTime.UtcNow.Subtract(lastActivityTime).TotalSeconds:F1} seconds ago. " +
            $"Total messages: {conversationHistory.Count}. " +
            "This may indicate agents are stuck in a loop or termination condition not met.", ex);
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "❌ Error during AI Scrum Team execution");
        throw;
    }
    finally
    {
        await runtime.RunUntilIdleAsync();
    }
    
    _logger.LogInformation("🎉 AI Scrum Team workflow completed successfully!");
    return deliverables;
}

private async Task MonitorProgressAsync(
    ChatHistory conversationHistory, 
    Dictionary<string, int> agentResponseCount, 
    CancellationToken cancellationToken)
{
    var lastCount = 0;
    var stuckCheckCount = 0;
    
    try
    {
        while (!cancellationToken.IsCancellationRequested)
        {
            await Task.Delay(30000, cancellationToken); // Check every 30 seconds
            
            var currentCount = conversationHistory.Count;
            
            if (currentCount == lastCount)
            {
                stuckCheckCount++;
                _logger.LogWarning($"⚠️ No progress in 30 seconds (check #{stuckCheckCount})");
                
                if (stuckCheckCount >= 3) // 90 seconds without progress
                {
                    _logger.LogError("🚨 No progress for 90+ seconds - possible deadlock!");
                    
                    // Log current state
                    var lastMessage = conversationHistory.LastOrDefault();
                    if (lastMessage != null)
                    {
                        _logger.LogError($"Last message from: {lastMessage.AuthorName}");
                        _logger.LogError($"Message preview: {lastMessage.Content?[..Math.Min(100, lastMessage.Content?.Length ?? 0)]}...");
                    }
                }
            }
            else
            {
                stuckCheckCount = 0;
                _logger.LogInformation($"📈 Progress: {currentCount} total messages");
            }
            
            lastCount = currentCount;
        }
    }
    catch (OperationCanceledException)
    {
        // Expected when cancellation is requested
    }
}
```

## Best Practices

### 1. Configuration Management
- Use user secrets for development
- Use Azure Key Vault for production
- Validate configuration on startup

### 2. Error Handling
```csharp
try
{
    var deliverables = await orchestrator.ProcessRequirementsAsync(requirements);
    return deliverables;
}
catch (HttpRequestException ex)
{
    _logger.LogError(ex, "Network error communicating with Azure OpenAI");
    throw new ScrumTeamException("Failed to communicate with AI service", ex);
}
catch (TaskCanceledException ex)
{
    _logger.LogError(ex, "Request timed out");
    throw new ScrumTeamException("AI processing timed out", ex);
}
```

### 3. Resource Management
```csharp
// Use dependency injection for proper disposal
public class ScrumTeamOrchestrator : IDisposable
{
    private readonly Kernel _kernel;
    
    public void Dispose()
    {
        _kernel?.Dispose();
    }
}
```

### 4. Performance Optimization
- Implement caching for repeated requests
- Use streaming for large outputs
- Configure appropriate timeout values
- Monitor token usage

## Key Migration Changes from Previous Versions

This HOL has been updated to reflect the latest Semantic Kernel Agent Framework. Here are the key changes:

### 1. Package Structure Changes

**Old packages (deprecated):**
- `Microsoft.SemanticKernel.Agents` (alpha version)

**New packages (current):**
- `Microsoft.SemanticKernel.Agents.Core` - Core agent functionality
- `Microsoft.SemanticKernel.Agents.Orchestration` - Orchestration patterns (prerelease)
- `Microsoft.SemanticKernel.Agents.Runtime.InProcess` - In-process runtime (prerelease)

### 2. Architecture Changes

**Old approach:**
- Used `AgentGroupChat` class (now deprecated)
- Direct agent management with `GroupChatManager`
- Manual conversation flow control

**New approach:**
- Uses `GroupChatOrchestration` pattern
- Cleaner separation with `InProcessRuntime`
- More robust orchestration framework

### 3. Code Migration Summary

| Old Pattern | New Pattern |
|-------------|-------------|
| `AgentGroupChat` | `GroupChatOrchestration` |
| `GroupChatManager` base class | `GroupChatManager` with updated methods |
| `Agent[]` | `ChatCompletionAgent[]` |
| `using Microsoft.SemanticKernel.Agents` | `using Microsoft.SemanticKernel.Agents.Core` |
| Direct chat invocation | Runtime-managed orchestration |

### 4. Benefits of the New Architecture

- **Better separation of concerns**: Clear distinction between agents, orchestration, and runtime
- **Improved extensibility**: Easier to add new orchestration patterns
- **Enhanced reliability**: More robust error handling and state management
- **Future-proof**: Aligned with Semantic Kernel's long-term architecture

## Conclusion

You've successfully built an AI Scrum Team using the latest Semantic Kernel Agent Framework in C#! This system demonstrates:

- **Modern agent orchestration** with the new GroupChatOrchestration pattern
- **Enterprise-ready architecture** with proper configuration and logging
- **Comprehensive deliverable generation** from requirements to test cases
- **Extensible design** using current Semantic Kernel best practices
- **Future-proof implementation** aligned with Semantic Kernel's roadmap

### Next Steps

- Explore other orchestration patterns (Sequential, Concurrent, Handoff, Magentic)
- Add integration with Azure DevOps or JIRA
- Implement iterative review cycles with human-in-the-loop
- Add support for different project types and domains
- Create web API for remote access
- Add real-time collaboration features

### Key Takeaways

- Semantic Kernel's new orchestration patterns provide powerful multi-agent capabilities
- The GroupChatOrchestration pattern enables sophisticated workflow control
- Proper configuration and logging are essential for production use
- Well-structured agents produce high-quality, consistent deliverables
- The migration to the new architecture provides better maintainability and extensibility
- This AI Scrum Team can significantly accelerate Agile planning and analysis phases

This updated AI Scrum Team leverages the latest Semantic Kernel capabilities to transform how you approach project requirements analysis, providing comprehensive, actionable deliverables that development teams can immediately use to build better software.
