# HOL - AI Scrum Team with Semantic Kernel (C#)

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
- **Microsoft.Extensions.Configuration**: Configuration management
- **Microsoft.Extensions.Logging**: Logging support

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
                    You are a detail-oriented QA Engineer specializing in comprehensive test planning.
                    
                    Based on the user stories and acceptance criteria, create detailed test scenarios 
                    and test cases in Gherkin syntax (Given/When/Then format).
                    
                    Organize your test cases by Feature, covering:
                    
                    1. **Happy Path Scenarios** - Normal successful workflows
                    2. **Error Handling** - Invalid inputs, system errors, network issues
                    3. **Edge Cases** - Boundary conditions, unusual but valid scenarios
                    4. **Security Testing** - Authentication, authorization, data validation
                    5. **Performance Considerations** - Load scenarios where applicable
                    
                    For each feature, write multiple test scenarios that thoroughly validate:
                    - All acceptance criteria from user stories
                    - Negative test cases for error handling
                    - Data validation and security checks
                    - User experience flows
                    
                    Use proper Gherkin format:
                    ```gherkin
                    Feature: [Feature Name]
                    
                    Scenario: [Scenario Description]
                    Given [initial context]
                    When [action is performed]
                    Then [expected outcome]
                    And [additional verification]

                    After completing all test cases, end your response with:
                    "Test cases complete"
                    
                    This signals that your testing analysis is finished.
                    """,
                Kernel = kernel,
                Description = "Generates comprehensive test scenarios and test cases in Gherkin format"
            };
        }
    }
}
```

## Step 4: Create Custom Group Chat Manager

Create `Managers/ScrumGroupChatManager.cs`:

```csharp
#pragma warning disable SKEXP0110 // Semantic Kernel Agents

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
        
        public ScrumGroupChatManager(ILogger<ScrumGroupChatManager> logger) 
        {
            _logger = logger;
            MaximumInvocationCount = 15;
        }

        public override ValueTask<GroupChatManagerResult<string>> SelectNextAgent(ChatHistory history, GroupChatTeam team, CancellationToken cancellationToken = default)
        {
            var lastMessage = history.LastOrDefault();
            var lastAgentName = lastMessage?.AuthorName;
            
            _logger.LogInformation($"Selecting next agent. Last message from: {lastAgentName ?? "None"}");
            
            // Sequential workflow: PO → BA → SA → QA
            var nextAgentName = lastAgentName switch
            {
                null or "user" => "ProductOwner",
                "ProductOwner" => "BusinessAnalyst", 
                "BusinessAnalyst" => "SolutionArchitect",
                "SolutionArchitect" => "QATester",
                "QATester" => "QATester", // Allow QA to finalize
                _ => "ProductOwner" // Fallback
            };
            
            _logger.LogInformation($"Selected next agent: {nextAgentName}");
            return ValueTask.FromResult(new GroupChatManagerResult<string>(nextAgentName));
        }

        public override ValueTask<GroupChatManagerResult<bool>> ShouldTerminate(ChatHistory history, CancellationToken cancellationToken = default)
        {
            // Check if QA has completed their work
            var qaMessages = history
                .Where(m => m.AuthorName == "QATester")
                .Select(m => m.Content)
                .ToList();
                
            var isComplete = qaMessages.Any(content => 
                content != null && content.Contains("Test cases complete", StringComparison.OrdinalIgnoreCase));
                
            if (isComplete)
            {
                _logger.LogInformation("QA has completed test cases. Terminating conversation.");
            }
            
            return ValueTask.FromResult(new GroupChatManagerResult<bool>(isComplete));
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
            var messages = history.ToList();
            var summary = "Scrum Team has completed their analysis and recommendations.";
            
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
            
            // Create response callback to capture agent messages
            ValueTask ResponseCallback(ChatMessageContent response)
            {
                conversationHistory.Add(response);
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
                // Execute the workflow
                var result = await orchestration.InvokeAsync(requirements, runtime);
                var finalOutput = await result.GetValueAsync(TimeSpan.FromMinutes(10));
                
                // Extract deliverables from conversation history
                foreach (var message in conversationHistory)
                {
                    if (string.IsNullOrEmpty(message.AuthorName) || message.AuthorName == "user")
                        continue;
                        
                    var content = message.Content;
                    _logger.LogInformation($"📝 Response from {message.AuthorName}: {content?[..Math.Min(100, content?.Length ?? 0)]}...");
                    
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
                
                _logger.LogInformation("✅ QA has completed all test cases. Workflow finished.");
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

## Step 7: Add Missing Usings

Add these using statements to the top of relevant files:

For `Models/ScrumDeliverables.cs`:

```csharp
using System.Text;
```

For `Managers/ScrumGroupChatManager.cs`:

```csharp
#pragma warning disable SKEXP0110 // Semantic Kernel Agents

using Microsoft.Extensions.Logging;
using Microsoft.SemanticKernel.Agents;
using Microsoft.SemanticKernel.Agents.Orchestration.GroupChat;
using Microsoft.SemanticKernel.ChatCompletion;
```

For `Services/ScrumTeamOrchestrator.cs`:

```csharp
#pragma warning disable SKEXP0110 // Semantic Kernel Agents

using Microsoft.SemanticKernel;
using Microsoft.SemanticKernel.Agents;
using Microsoft.SemanticKernel.Agents.Orchestration;
using Microsoft.SemanticKernel.Agents.Orchestration.GroupChat;
using Microsoft.SemanticKernel.Agents.Runtime.InProcess;
using Microsoft.SemanticKernel.ChatCompletion;
```

## Step 8: Build and Test

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

## Step 9: Advanced Features

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

## Troubleshooting

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

4. **NuGet Package Conflicts**
```bash
dotnet clean
dotnet restore
dotnet build
```

3. **Azure OpenAI Connection Issues**
```csharp
// Test connection in Program.cs
var testKernel = Kernel.CreateBuilder()
    .AddAzureOpenAIChatCompletion(deploymentName, endpoint, apiKey)
    .Build();
    
var chatService = testKernel.GetRequiredService<IChatCompletionService>();
Console.WriteLine("✅ Azure OpenAI connection successful");
```

4. **Configuration Problems**
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
