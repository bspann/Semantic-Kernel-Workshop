# Semantic Kernel C# Hands-On Lab Guide - Azure Government

> **Note**: This hands-on lab has been updated to use the latest Semantic Kernel C# requirements and API structures as of August 2025. The code examples reflect the current recommended practices and package versions.

This comprehensive lab guide will walk you through building a complete Semantic Kernel application in C#. You'll learn to create kernels, add services, implement plugins, configure vector stores, and add filters using Azure OpenAI in Azure Government Cloud.

## Prerequisites

### Development Environment
- Visual Studio Code
- .NET 8.0 SDK or later
- C# extension for VS Code

### Required Azure OpenAI Resources
- Azure OpenAI resource deployed in Azure Government Cloud
- Azure OpenAI service endpoint (e.g., `https://your-resource.openai.usgovcloudapi.net/`)
- Azure OpenAI API key
- Deployed models: `gpt-4o` and `text-embedding-ada-002`

## Required NuGet Packages

> **Important**: This lab requires Semantic Kernel .NET 1.35.3+ packages. The package structure and API have been updated to reflect the latest recommended patterns from Microsoft's official documentation.

Before starting the C# lab, you'll need to install the following NuGet packages. Each package serves a specific purpose in the Semantic Kernel ecosystem:

| Package Name | Version | Purpose |
|--------------|---------|---------|
| `Microsoft.SemanticKernel` | Latest | Core Semantic Kernel framework and orchestration engine |
| `Microsoft.SemanticKernel.Connectors.AzureOpenAI` | Latest | Azure OpenAI integration for chat completion and embeddings |
| `Microsoft.SemanticKernel.Plugins.OpenApi` | Latest | Support for OpenAPI/REST API plugins |
| `Microsoft.SemanticKernel.Connectors.InMemory` | Latest (prerelease) | In-memory vector store provider for demonstrations and testing |
| `Microsoft.Extensions.AI` | Latest | Modern AI abstractions and interfaces |
| `Microsoft.Extensions.Logging.Console` | Latest | Console logging support for debugging |
| `Azure.Identity` | Latest | Azure authentication and identity management |

**Installation Command:**
```bash
# Run this single command to install all required packages
dotnet add package Microsoft.SemanticKernel && \
dotnet add package Microsoft.SemanticKernel.Connectors.AzureOpenAI && \
dotnet add package Microsoft.SemanticKernel.Plugins.OpenApi && \
dotnet add package Microsoft.SemanticKernel.Connectors.InMemory --prerelease && \
dotnet add package Microsoft.Extensions.AI && \
dotnet add package Microsoft.Extensions.Logging.Console && \
dotnet add package Azure.Identity
```

**Alternative Individual Installation:**
If you prefer to install packages one by one, use these commands in the VS Code terminal:

```bash
dotnet add package Microsoft.SemanticKernel
dotnet add package Microsoft.SemanticKernel.Connectors.AzureOpenAI  
dotnet add package Microsoft.SemanticKernel.Plugins.OpenApi
dotnet add package Microsoft.SemanticKernel.Connectors.InMemory --prerelease
dotnet add package Microsoft.Extensions.AI
dotnet add package Microsoft.Extensions.Logging.Console
dotnet add package Azure.Identity
```

## Step 1: Project Setup

1. **Open VS Code** and create a new folder for your project:
   ```bash
   mkdir SemanticKernelLab
   cd SemanticKernelLab
   code .
   ```

2. **Open the integrated terminal** in VS Code (`Ctrl+`` ` or `View > Terminal`) and create a new console application:
   ```bash
   dotnet new console
   ```

3. **Install the required NuGet packages** using the installation command from the Required NuGet Packages section above, or run them individually:
   ```bash
   dotnet add package Microsoft.SemanticKernel
   dotnet add package Microsoft.SemanticKernel.Connectors.AzureOpenAI
   dotnet add package Microsoft.SemanticKernel.Plugins.OpenApi
   dotnet add package Microsoft.SemanticKernel.Connectors.InMemory --prerelease
   dotnet add package Microsoft.Extensions.AI
   dotnet add package Microsoft.Extensions.Logging.Console
   dotnet add package Azure.Identity
   ```

   > **What you're doing:** These packages provide the core Semantic Kernel functionality, Azure OpenAI connectors, plugin support, memory capabilities, and logging infrastructure needed for the lab.

## Step 2: Create the Kernel and Add Chat Completion Service

4. **Replace the contents of `Program.cs`** with the following code. This sets up the kernel with Azure OpenAI in Azure Government:

```csharp
using Microsoft.SemanticKernel;
using Microsoft.SemanticKernel.ChatCompletion;
using Microsoft.SemanticKernel.Connectors.AzureOpenAI;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.DependencyInjection;

// Create a kernel builder
var builder = Kernel.CreateBuilder();

// Add Azure OpenAI chat completion service for Azure Government
builder.AddAzureOpenAIChatCompletion(
    deploymentName: "gpt-4o", // Your deployed model name
    endpoint: "https://your-resource.openai.azure.us/", // Azure Gov endpoint
    apiKey: "your-azure-openai-api-key"); // Replace with your actual API key

// Add console logging
builder.Services.AddLogging(c => c.AddConsole().SetMinimumLevel(LogLevel.Information));

// Build the kernel
Kernel kernel = builder.Build();

Console.WriteLine("✅ Kernel created successfully!");
Console.WriteLine("✅ Azure OpenAI chat completion service added for Azure Government!");

// Test basic chat completion
var chatCompletionService = kernel.GetRequiredService<IChatCompletionService>();
var response = await chatCompletionService.GetChatMessageContentAsync("Hello, what's the capital of France?");
Console.WriteLine($"AI Response: {response.Content}");
```

> **What you're doing:** This code creates a Semantic Kernel instance and adds an Azure OpenAI chat completion service specifically configured for Azure Government Cloud. Note the `.usgovcloudapi.net` endpoint which is specific to Azure Government.

**Important Configuration Notes:**
- **Endpoint:** Azure Government uses `.usgovcloudapi.net` instead of `.openai.azure.com`
- **Deployment Name:** Use the exact name you gave your model deployment in Azure OpenAI Studio
- **API Key:** Get this from your Azure OpenAI resource in the Azure Government portal
- **Model:** Azure Government typically has `gpt-4o`

5. **Run the application** to test the basic setup:
   ```bash
   dotnet run
   ```

## Step 3: Add Native Plugin

6. **Create a new file** called `MathPlugin.cs` in your project folder. Add the following native plugin code:

```csharp
using System.ComponentModel;
using Microsoft.SemanticKernel;

public class MathPlugin
{
    [KernelFunction]
    [Description("Adds two numbers together")]
    public double Add(
        [Description("The first number")] double number1,
        [Description("The second number")] double number2)
    {
        return number1 + number2;
    }

    [KernelFunction]
    [Description("Multiplies two numbers together")]
    public double Multiply(
        [Description("The first number")] double number1,
        [Description("The second number")] double number2)
    {
        return number1 * number2;
    }

    [KernelFunction]
    [Description("Calculates the square root of a number")]
    public double SquareRoot([Description("The number to calculate square root for")] double number)
    {
        if (number < 0)
            throw new ArgumentException("Cannot calculate square root of negative number");
        return Math.Sqrt(number);
    }
}
```

> **What you're doing:** This creates a native plugin with mathematical functions. Native plugins are C# classes with methods decorated with `[KernelFunction]` attributes that the AI can call.

## Step 4: Add OpenAPI Plugin

7. **Update your `Program.cs`** to include all plugins and demonstrate their usage:

```csharp
using Microsoft.SemanticKernel;
using Microsoft.SemanticKernel.ChatCompletion;
using Microsoft.SemanticKernel.Connectors.AzureOpenAI;
using Microsoft.SemanticKernel.Plugins.OpenApi;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.DependencyInjection;

// Create a kernel builder
var builder = Kernel.CreateBuilder();

// Add Azure OpenAI chat completion service for Azure Government
builder.AddAzureOpenAIChatCompletion(
    deploymentName: "gpt-35-turbo", // Your deployed model name
    endpoint: "https://your-resource.openai.usgovcloudapi.net/", // Azure Gov endpoint
    apiKey: "your-azure-openai-api-key"); // Replace with your actual API key

// Add console logging
builder.Services.AddLogging(c => c.AddConsole().SetMinimumLevel(LogLevel.Information));

// Build the kernel with all services
Kernel kernel = builder.Build();

Console.WriteLine("✅ Kernel created successfully!");

// Add native plugin after kernel is built
kernel.Plugins.AddFromType<MathPlugin>("MathPlugin");
Console.WriteLine("✅ Native MathPlugin added!");

// Add OpenAPI plugin (using a public API - note: ensure API is accessible from Azure Gov)
try
{
    await kernel.ImportPluginFromOpenApiAsync(
        "PetStorePlugin",
        new Uri("https://petstore.swagger.io/v2/swagger.json"),
        new OpenApiFunctionExecutionParameters()
        {
            HttpClient = new HttpClient()
        });
    Console.WriteLine("✅ OpenAPI PetStorePlugin added!");
    Console.WriteLine("Note: Some API operations may show warnings due to OpenAPI specification issues - this is normal.");
}
catch (Exception ex)
{
    Console.WriteLine($"⚠️ OpenAPI plugin failed to load: {ex.Message}");
    Console.WriteLine("This is expected in some environments due to network restrictions or API availability.");
    Console.WriteLine("Continuing without OpenAPI plugin...");
}

// Test native plugin
Console.WriteLine("\n🧮 Testing Native Plugin:");
var mathResult = await kernel.InvokeAsync("MathPlugin", "Add", new() { ["number1"] = "5", ["number2"] = "3" });
Console.WriteLine($"5 + 3 = {mathResult}");

// Test OpenAPI plugin (if loaded successfully)
Console.WriteLine("\n🌐 Testing OpenAPI Plugin:");
try
{
    // Note: The PetStore API may require authentication or have rate limiting
    // This demonstrates the concept even if the actual API call fails
    Console.WriteLine("Attempting to call PetStore API functions...");
    
    // Try to list available pets from the PetStore API
    var petsResult = await kernel.InvokeAsync("PetStorePlugin", "findPetsByStatus", new() { ["status"] = "available" });
    Console.WriteLine($"Available pets from PetStore: {petsResult}");
    
    // Try to get a specific pet by ID
    var petResult = await kernel.InvokeAsync("PetStorePlugin", "getPetById", new() { ["petId"] = "1" });
    Console.WriteLine($"Pet details: {petResult}");
}
catch (Exception ex)
{
    Console.WriteLine($"OpenAPI plugin test failed (this is expected in restricted environments):");
    Console.WriteLine($"Error: {ex.Message}");
    Console.WriteLine("This demonstrates that OpenAPI plugins can be loaded even when the actual API calls fail due to network restrictions, authentication requirements, or API changes.");
}

// Test function calling with chat completion
Console.WriteLine("\n💬 Testing Function Calling:");
var chatHistory = new ChatHistory();
chatHistory.AddUserMessage("Can you calculate the square root of 16 and then multiply it by 3? Also, can you find information about available pets from the pet store?");

var chatFunction = kernel.GetRequiredService<IChatCompletionService>();
var executionSettings = new AzureOpenAIPromptExecutionSettings()
{
    FunctionChoiceBehavior = FunctionChoiceBehavior.Auto()
};

var response = await chatFunction.GetChatMessageContentAsync(chatHistory, executionSettings, kernel);
Console.WriteLine($"AI Response: {response.Content}");
```

> **What you're doing:** This code demonstrates loading all types of plugins and using them with Azure OpenAI in Azure Government. The OpenAPI plugin automatically converts REST API endpoints into callable functions that the AI can use. In this example, the PetStore API provides functions like `findPetsByStatus` and `getPetById` that can be called programmatically or by the AI during function calling scenarios.

## Step 5: Add In-Memory Vector Store (Modern Approach)

8. **Create a new file** called `VectorStoreService.cs`:

```csharp
using Microsoft.SemanticKernel;
using Microsoft.SemanticKernel.Data;
using Microsoft.SemanticKernel.Embeddings;
using Microsoft.SemanticKernel.Connectors.AzureOpenAI;
using Microsoft.Extensions.AI;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.VectorData;
using Microsoft.SemanticKernel.Connectors.InMemory;
using System.Collections.Generic;
using System.Linq;

#pragma warning disable SKEXP0010 // Type is for evaluation purposes only and is subject to change or removal in future updates

public class VectorStoreService
{
    private readonly InMemoryVectorStore _vectorStore;
    private readonly IEmbeddingGenerator<string, Embedding<float>> _embeddingService;
    private const string CollectionName = "GeneralKnowledge";

    public VectorStoreService(string endpoint, string apiKey, string embeddingDeploymentName)
    {
        // Create an in-memory vector store (simpler and more reliable than SQLite)
        _vectorStore = new InMemoryVectorStore();
        
        // Create Azure OpenAI embedding service using dependency injection pattern
        var serviceCollection = new ServiceCollection();
        serviceCollection.AddAzureOpenAIEmbeddingGenerator(
            deploymentName: embeddingDeploymentName,
            endpoint: endpoint,
            apiKey: apiKey);
        var serviceProvider = serviceCollection.BuildServiceProvider();
        _embeddingService = serviceProvider.GetRequiredService<IEmbeddingGenerator<string, Embedding<float>>>();
    }

    public async Task SaveInformationAsync(string id, string text, string? description = null)
    {
        // Generate embedding for the text
        var embeddingResult = await _embeddingService.GenerateAsync(text);
        
        var record = new DataModel
        {
            Id = id,
            Text = text,
            Description = description ?? "",
            Embedding = embeddingResult.Vector
        };

        // Get collection and upsert the record
        var collection = _vectorStore.GetCollection<string, DataModel>(CollectionName);
        await collection.EnsureCollectionExistsAsync();
        await collection.UpsertAsync(record);
        Console.WriteLine($"💾 Saved to vector store: {id}");
    }

    public async Task<string> SearchInformationAsync(string query, double minScore = 0.8)
    {
        try
        {
            // Generate embedding for the search query
            var searchVector = (await _embeddingService.GenerateAsync(query)).Vector;
            
            // Get collection and search it using vector search
            var collection = _vectorStore.GetCollection<string, DataModel>(CollectionName);
            var searchResults = new List<VectorSearchResult<DataModel>>();
            await foreach (var result in collection.SearchAsync(searchVector, top: 3))
            {
                searchResults.Add(result);
            }
            
            if (searchResults.Count == 0)
            {
                return "No relevant information found in vector store.";
            }
            
            // Filter results based on minimum score
            var filteredResults = searchResults
                .Where(result => result.Score >= minScore)
                .ToList();
            
            if (filteredResults.Count == 0)
            {
                return $"No results found with minimum score of {minScore}. Highest score was {searchResults.First().Score:F3}";
            }
            
            // Format and return the results
            var formattedResults = filteredResults
                .Select(result => $"Score: {result.Score:F3} - {result.Record.Text}")
                .ToList();
            
            Console.WriteLine($"🔍 Found {filteredResults.Count} results for query: '{query}'");
            return string.Join("\n", formattedResults);
        }
        catch (Exception ex)
        {
            return $"Search error: {ex.Message}";
        }
    }
}

public class DataModel
{
    [VectorStoreKey]
    public string Id { get; set; } = string.Empty;

    [VectorStoreData]
    public string Text { get; set; } = string.Empty;

    [VectorStoreData]
    public string Description { get; set; } = string.Empty;

    [VectorStoreVector(1536)]
    public ReadOnlyMemory<float> Embedding { get; set; }
}
```

> **What you're doing:** This creates a vector store service using an in-memory vector store instead of the legacy memory system. In-memory vector stores are perfect for demonstrations, development, and testing scenarios. They provide excellent performance and don't require external dependencies like SQLite, making them ideal for HOL scenarios while still demonstrating the same concepts that scale to production with persistent vector stores.

## Step 6: Add Filters

9. **Create a new file** called `LoggingFilter.cs`:

```csharp
using Microsoft.SemanticKernel;

public class LoggingFilter : IFunctionInvocationFilter
{
    public async Task OnFunctionInvocationAsync(FunctionInvocationContext context, Func<FunctionInvocationContext, Task> next)
    {
        Console.WriteLine($"🔍 Filter: Executing function '{context.Function.Name}'");
        Console.WriteLine($"📥 Input: {string.Join(", ", context.Arguments.Select(arg => $"{arg.Key}={arg.Value}"))}");
        
        var startTime = DateTime.UtcNow;
        
        try
        {
            await next(context);
            var duration = DateTime.UtcNow - startTime;
            Console.WriteLine($"✅ Filter: Function '{context.Function.Name}' completed in {duration.TotalMilliseconds}ms");
            Console.WriteLine($"📤 Output: {context.Result}");
        }
        catch (Exception ex)
        {
            var duration = DateTime.UtcNow - startTime;
            Console.WriteLine($"❌ Filter: Function '{context.Function.Name}' failed after {duration.TotalMilliseconds}ms: {ex.Message}");
            throw;
        }
    }
}

public class SecurityFilter : IFunctionInvocationFilter
{
    private readonly HashSet<string> _blockedWords = new(StringComparer.OrdinalIgnoreCase)
    {
        "password", "secret", "apikey", "token"
    };

    public async Task OnFunctionInvocationAsync(FunctionInvocationContext context, Func<FunctionInvocationContext, Task> next)
    {
        // Check arguments for sensitive information
        foreach (var arg in context.Arguments)
        {
            if (_blockedWords.Any(blocked => arg.Value?.ToString()?.Contains(blocked, StringComparison.OrdinalIgnoreCase) == true))
            {
                Console.WriteLine($"🛡️ Security Filter: Blocked potentially sensitive input in argument '{arg.Key}'");
                throw new UnauthorizedAccessException($"Argument '{arg.Key}' contains potentially sensitive information");
            }
        }

        await next(context);
    }
}
```

> **What you're doing:** Filters intercept function calls before and after execution. The logging filter tracks performance and inputs/outputs, while the security filter prevents sensitive data from being processed.

10. **Update `Program.cs`** to include vector store and filters:

```csharp
using Microsoft.SemanticKernel;
using Microsoft.SemanticKernel.ChatCompletion;
using Microsoft.SemanticKernel.Connectors.AzureOpenAI;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.DependencyInjection;

// Configuration for Azure Government
const string azureEndpoint = "https://your-resource.openai.usgovcloudapi.net/";
const string apiKey = "your-azure-openai-api-key"; // Replace with your actual API key
const string chatDeploymentName = "gpt-35-turbo"; // Your chat model deployment name
const string embeddingDeploymentName = "text-embedding-ada-002"; // Your embedding model deployment name

// Create a kernel builder
var builder = Kernel.CreateBuilder();

// Add Azure OpenAI chat completion service for Azure Government
builder.AddAzureOpenAIChatCompletion(
    deploymentName: chatDeploymentName,
    endpoint: azureEndpoint,
    apiKey: apiKey);

// Add console logging
builder.Services.AddLogging(c => c.AddConsole().SetMinimumLevel(LogLevel.Information));

// Add filters
builder.Services.AddSingleton<IFunctionInvocationFilter, LoggingFilter>();
builder.Services.AddSingleton<IFunctionInvocationFilter, SecurityFilter>();

// Add prompt-based plugins to builder
var pluginsDirectory = Path.Combine(Directory.GetCurrentDirectory(), "Plugins");
builder.Plugins.AddFromPromptDirectory(pluginsDirectory);

// Build the kernel
Kernel kernel = builder.Build();

Console.WriteLine("✅ Kernel created with Azure OpenAI for Azure Government!");
Console.WriteLine("✅ Filters enabled!");

// Add native plugins
kernel.Plugins.AddFromType<MathPlugin>("MathPlugin");

// Initialize vector store service with Azure OpenAI embeddings
var vectorStoreService = new VectorStoreService(azureEndpoint, apiKey, embeddingDeploymentName);

// Save some information to vector store
await vectorStoreService.SaveInformationAsync("fact1", "The capital of France is Paris");
await vectorStoreService.SaveInformationAsync("fact2", "Python is a programming language created by Guido van Rossum");
await vectorStoreService.SaveInformationAsync("fact3", "The Semantic Kernel is a Microsoft framework for AI orchestration");
await vectorStoreService.SaveInformationAsync("fact4", "Azure Government provides cloud services for US government agencies");

Console.WriteLine("\n🧠 Testing Vector Store with Azure OpenAI Embeddings:");
var vectorSearchResult = await vectorStoreService.SearchInformationAsync("What is the capital of France?");
Console.WriteLine($"Vector search result:\n{vectorSearchResult}");

Console.WriteLine("\n🧮 Testing Filters with Math Plugin:");
try
{
    var result = await kernel.InvokeAsync("MathPlugin", "Add", new() { ["number1"] = "10", ["number2"] = "5" });
    Console.WriteLine($"Math result: {result}");
}
catch (Exception ex)
{
    Console.WriteLine($"Error: {ex.Message}");
}

Console.WriteLine("\n🛡️ Testing Security Filter:");
try
{
    // This should trigger the security filter
    await kernel.InvokeAsync("MathPlugin", "Add", new() { ["number1"] = "mypassword123", ["number2"] = "5" });
}
catch (UnauthorizedAccessException ex)
{
    Console.WriteLine($"Security filter activated: {ex.Message}");
}

Console.WriteLine("\n🎉 C# Lab with Azure Government completed successfully!");
```

> **What you're doing:** This final version demonstrates the complete integration with Azure OpenAI in Azure Government, including vector store services using Azure OpenAI embeddings and all filtering capabilities.

## Running the C# Lab

1. **Navigate to your C# project folder** in VS Code
2. **Ensure your Azure OpenAI credentials are set** in the code:
   - Replace `"https://your-resource.openai.usgovcloudapi.net/"` with your actual Azure Government endpoint
   - Replace `"your-azure-openai-api-key"` with your actual API key
   - Verify your deployment names match (`gpt-35-turbo`, `text-embedding-ada-002`)
3. **Run the application:**
   ```bash
   dotnet run
   ```
4. **Expected output:** You should see successful creation of kernel, plugins, memory operations with Azure OpenAI embeddings, and filter demonstrations

## Key Learning Points

### What You've Built

**Kernel Creation:**
- The kernel is the central orchestration engine that manages AI services, plugins, and execution
- Configured specifically for Azure Government Cloud compliance requirements

**Chat Completion Service:**
- Integrated Azure OpenAI's GPT models deployed in Azure Government for natural language processing
- Configured for function calling to enable AI to use your plugins automatically
- Uses Azure Government specific endpoints (`.usgovcloudapi.net`) for compliance requirements

**Plugin Types:**
- **Native Plugins:** Code-based functions (C# methods) that AI can call
- **OpenAPI Plugins:** Integration with external REST APIs (PetStore API example)
  - Automatically converts REST endpoints into callable functions
  - Enables AI to interact with external services and data sources
  - Supports GET, POST, PUT, DELETE operations based on OpenAPI specification
  - Handles authentication, parameter validation, and response parsing

**In-Memory Vector Store:**
- Semantic search using Azure OpenAI vector embeddings deployed in Azure Government
- Modern Vector Store approach (replaces legacy Memory Store)
- In-memory storage ideal for development, testing, and HOL scenarios
- Enables AI to remember and retrieve relevant context while maintaining government compliance requirements
- Easily scales to persistent storage solutions (SQLite, Redis, etc.) for production use

**Filters:**
- Pre/post-processing of function calls
- Logging, security, performance monitoring
- Extensible pipeline for custom behaviors

### Development Workflow in VS Code

**File Organization:**
```text
C# Project:
├── Program.cs (main application with Azure Government config)
├── MathPlugin.cs (native plugin)
├── VectorStoreService.cs (vector store operations with Azure OpenAI embeddings)
└── LoggingFilter.cs (filters)
```

**Key VS Code Features Used:**
- Integrated terminal for package management and running applications
- IntelliSense for code completion and error detection
- File explorer for organizing plugin structures
- Extensions for C# development support

### Package Version Compatibility

**Important Notes:**
- **Semantic Kernel Versioning:** Uses Semantic Kernel .NET 1.0+ with current stable APIs and improved Azure Government support.
- **Vector Store Migration:** Legacy Memory Store packages have been replaced by modern Vector Store connectors. This HOL uses the in-memory vector store for simplicity and reliability, which is perfect for development and testing scenarios.
- **Azure Package Dependencies:** The Azure OpenAI connectors automatically handle compatible versions of Azure SDKs.
- **.NET Version:** Requires .NET 8.0 or later for optimal Semantic Kernel support.

**Verifying Installation:**
After installing packages, verify they're working correctly:

**C# Verification:**
```bash
dotnet list package
```

### Key Updates in This Version

- **Updated to Semantic Kernel .NET 1.0+**: Leveraging the latest stable APIs and improvements for Azure Government
- **.NET 8.0+ requirement**: Ensuring compatibility with modern .NET features and performance optimizations
- **Vector Store approach**: Modern Vector Store using in-memory storage for demonstrations, replacing legacy Memory Store with improved performance and simpler setup
- **Simplified package structure**: Updated package dependencies to reflect current Semantic Kernel organization
- **Current API patterns**: Updated service registration and kernel building to follow latest recommended practices

### Troubleshooting Tips

**Common Issues:**
1. **Azure OpenAI Credential Errors:** Ensure your Azure OpenAI resource is properly deployed in Azure Government and API key is valid
2. **Endpoint Configuration:** Verify you're using the correct Azure Government endpoint (`.usgovcloudapi.net`)
3. **Model Deployment Names:** Check that your deployment names exactly match those in Azure OpenAI Studio
4. **Package Versions:** Semantic Kernel is rapidly evolving; ensure Azure connector packages are compatible
5. **Path Issues:** Verify plugin directory structures match exactly as shown
6. **Async/Await:** Both implementations use asynchronous patterns; ensure proper async handling
7. **Government Cloud Access:** Ensure your subscription has access to Azure Government and required services are enabled
8. **OpenAPI Plugin Errors:** Public APIs like PetStore may require authentication or have rate limiting. The JSON parsing errors (like "'a' is an invalid start of a value") typically indicate the API returned an error message instead of JSON data. This is expected behavior in restricted environments and demonstrates the plugin loading concept even when API calls fail.

**Azure Government Specific Considerations:**
- **Compliance:** Azure Government provides FedRAMP High and DoD IL2-IL5 compliance
- **Data Residency:** All data processing occurs within US government data centers
- **Network Isolation:** Government cloud services are isolated from commercial Azure
- **Model Availability:** Check Azure Government documentation for available AI models and regions

**Best Practices:**
- Use environment variables for Azure OpenAI credentials (never hardcode secrets)
- Implement proper error handling for AI service calls
- Structure plugins in separate files/modules for maintainability
- Test each component individually before integration
- Use logging to debug function call flows
- Follow Azure Government security and compliance guidelines
- Regularly update Semantic Kernel packages for latest Azure Government compatibility
- Monitor usage and costs through Azure Government portal

**Troubleshooting Package Issues:**
- **C#:** If package conflicts occur, try `dotnet clean` and `dotnet restore`
- **Version Conflicts:** Check the Semantic Kernel GitHub releases page for compatible package versions

**Additional Azure Government Resources:**
- [Azure Government Documentation](https://docs.microsoft.com/en-us/azure/azure-government/)
- [Azure OpenAI in Government](https://docs.microsoft.com/en-us/azure/cognitive-services/openai/overview#azure-government)
- [Government Cloud Compliance](https://azure.microsoft.com/en-us/global-infrastructure/government/)

This lab provides a comprehensive foundation for building AI-powered applications with Semantic Kernel using Azure OpenAI in Azure Government, demonstrating core concepts that scale to production scenarios while maintaining government compliance requirements. The Azure Government configuration ensures data sovereignty and regulatory compliance for government and defense applications.