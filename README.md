# eShopSupport 

https://www.youtube.com/watch?v=pM9bTeaaAlQ&list=PLdo4fOcmZ0oX7Yg1cixIj6hXjz9C5MHJR&index=3

## How to set up OpenAI instead of Ollama

Modify the Program.cs file:

![image](https://github.com/user-attachments/assets/c6c68424-88a1-4e85-a8ca-73afa8a46dc6)

Include this code in the middleware:

```
var chatCompletion = builder.AddConnectionString("chatcompletion");
```

And also define the OpenAI connection string in the appSettings.json file:

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning",
      "Aspire.Hosting.Dcp": "Warning"
    }
  },
  "Parameters": {
    "PostgresPassword": "dev"
  },
  "ConnectionStrings": {
    "chatcompletion": "ApiKey=XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX;Endpoint=https://api.openai.com/v1;Model=gpt-4o"
  }
}
```

To obtain an OpenAI key navigate to this URL: https://platform.openai.com/api-keys

Also modify the following files: 

![image](https://github.com/user-attachments/assets/bf0f5250-99e4-4e5b-b064-e0f50fcc3f0a)

**ChatCompletionServiceExtensions.cs**

```csharp
using Microsoft.Extensions.AI;

namespace Microsoft.Extensions.Hosting;

public static class ChatCompletionServiceExtensions
{
    public static void AddChatCompletionService(this IHostApplicationBuilder builder, string serviceName)
    {
        var pipeline = (ChatClientBuilder pipeline) => pipeline
            .UseFunctionInvocation()
            .UseCachingForTest()
            .UseOpenTelemetry(configure: c => c.EnableSensitiveData = true);

        if (builder.Configuration[$"{serviceName}:Type"] == "ollama")
        {
           // builder.AddOllamaChatClient(serviceName, pipeline);
        }
        else
        {
            builder.AddOpenAIChatClient(serviceName, pipeline);
        }
    }
}
```

**ServiceCollectionChatClientExtensions.cs**

```csharp
using Azure.AI.OpenAI;
using System.ClientModel;
using Microsoft.Extensions.AI;
using Microsoft.Extensions.DependencyInjection;
using OpenAI;
using System.Data.Common;
using Microsoft.Extensions.Configuration;

namespace Microsoft.Extensions.Hosting;

public static class ServiceCollectionChatClientExtensions
{
    public static IServiceCollection AddOpenAIChatClient(
        this IHostApplicationBuilder hostBuilder,
        string serviceName,
        Func<ChatClientBuilder, ChatClientBuilder>? builder = null,
        string? modelOrDeploymentName = null)
    {
        // Retrieve the connection string from configuration
        var connectionString = hostBuilder.Configuration.GetConnectionString(serviceName);
        if (string.IsNullOrWhiteSpace(connectionString))
        {
            throw new InvalidOperationException($"No connection string named '{serviceName}' was found. Ensure a corresponding Aspire service was registered.");
        }

        // Parse the connection string to extract API key, endpoint, and model/deployment name
        var connectionStringBuilder = new DbConnectionStringBuilder();
        connectionStringBuilder.ConnectionString = connectionString;

        // Extract necessary parameters from the connection string
        var apiKey = (string?)connectionStringBuilder["ApiKey"] ?? throw new InvalidOperationException($"The connection string named '{serviceName}' does not specify a value for 'ApiKey', but this is required.");
        var endpoint = (string?)connectionStringBuilder["Endpoint"];
        modelOrDeploymentName ??= (string?)connectionStringBuilder["Model"] ?? throw new InvalidOperationException($"The connection string named '{serviceName}' does not specify a value for 'Model', and no value was passed for {nameof(modelOrDeploymentName)}.");

        // Create the endpoint URI if provided
        var endpointUri = string.IsNullOrEmpty(endpoint) ? null : new Uri(endpoint);

        // Register the OpenAI chat client
        return hostBuilder.Services.AddOpenAIChatClient(apiKey, modelOrDeploymentName, endpointUri, builder);
    }

    public static IServiceCollection AddOpenAIChatClient(
        this IServiceCollection services,
        string apiKey,
        string modelOrDeploymentName,
        Uri? endpoint = null,
        Func<ChatClientBuilder, ChatClientBuilder>? builder = null)
    {
        // Register the OpenAI client depending on whether an endpoint is provided
        return services
            .AddSingleton(_ => endpoint is null
                ? new OpenAIClient(apiKey)  // Use OpenAI's client with API key
                : new AzureOpenAIClient(endpoint, new ApiKeyCredential(apiKey))) // Azure-specific client
            .AddChatClient(pipeline =>
            {
                // Apply additional configurations if provided
                builder?.Invoke(pipeline);

                // Use the OpenAI or Azure OpenAI client as the chat client
                var openAiClient = pipeline.Services.GetRequiredService<OpenAIClient>();
                return pipeline.Use(openAiClient.AsChatClient(modelOrDeploymentName));
            });
    }
}
```

## General Description

A sample .NET application showcasing common use cases and development practices for build AI solutions in .NET (Generative AI, specifically). This sample demonstrates a customer support application for an e-commerce website using a services-based architecture with .NET Aspire. It includes support for the following AI use cases:

* Text classification, applying labels based on content
* Sentiment analysis based on message content
* Summarization of large sets of text
* Synthetic data generation, creating test content for the sample
* Chat bot interactions with chat history and suggested responses

<img width=400 align=top src=https://github.com/user-attachments/assets/5a41493f-565b-4dd0-ae31-1b5c3c2f6d22>

<img width=400 align=top src=https://github.com/user-attachments/assets/7930a940-bb31-4dc0-b5f6-738d43dfcfe5>

This sample also demonstrates the following development practices:

* Developing a solution locally, using small local models
* Evaluating the quality of AI responses using grounded Q&A data
* Leveraging Python projects as part of a .NET Aspire solution
* Deploying the application, including small local models, to the Cloud (coming soon)

## Architecture

![image](https://github.com/user-attachments/assets/3c339d0d-507a-416b-94ba-0e179d6ff2f5)

## Getting Started

### Prerequisites

- A device with an Nvidia GPU (see [workaround for running on the CPU](https://github.com/dotnet/eShopSupport/issues/19))
- Clone the eShopSupport repository: https://github.com/dotnet/eshopsupport
- [Install & start Docker Desktop](https://docs.docker.com/engine/install/)
- [Install Python 3.12.5](https://www.python.org/downloads/release/python-3125/)

#### Windows with Visual Studio
- Install [Visual Studio 2022 version 17.10 or newer](https://visualstudio.microsoft.com/vs/)
  - Select the following workloads:
    - `ASP.NET and web development` workload.
    - `Python Development` workload.
    - `.NET Aspire SDK` component in `Individual components`.

#### Mac, Linux, & Windows without Visual Studio
- Install the latest [.NET 8 SDK](https://dot.net/download?cid=eshop)
- Install the [.NET Aspire workload](https://learn.microsoft.com/dotnet/aspire/fundamentals/setup-tooling?tabs=dotnet-cli%2Cunix#install-net-aspire) with the following commands:

  ```powershell
  dotnet workload update
  dotnet workload install aspire
  dotnet restore eShopSupport.sln
  ```
- (Optionally) Install [Visual Studio Code](https://code.visualstudio.com) with the [C# Dev Kit extension](https://marketplace.visualstudio.com/items?itemName=ms-dotnettools.csdevkit)

#### Install Python requirements

From the Terminal, at the root of the cloned repo, run:

```powershell
pip install -r src/PythonInference/requirements.txt
```

**Note:** If the above command doesn't work on Windows, use the following command:

```powershell
py -m pip install -r src/PythonInference/requirements.txt
```

### Running the solution

> [!WARNING]
> Remember to ensure that Docker is started.

* (Windows only) Run the application from Visual Studio:
  - Open the `eShopSupport.sln` file in Visual Studio
  - Ensure that `AppHost` is your startup project
  - Hit Ctrl-F5 to launch .NET Aspire

* Or run the application from your terminal:

  ```powershell
  dotnet run --project src/AppHost
  ```

  then look for lines like this in the console output in order to find the URL to open the Aspire dashboard:

  ```sh
  Login to the dashboard at: http://localhost:17191/login?t=uniquelogincodeforyou
  ```

> You may need to install ASP.NET Core HTTPS development certificates first, and then close all browser tabs. Learn more at https://aka.ms/aspnet/https-trust-dev-cert

# Contributing

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/). For more information see the [Code of Conduct FAQ](https://opensource.microsoft.com/codeofconduct/faq/) or contact [opencode@microsoft.com](mailto:opencode@microsoft.com) with any additional questions or comments.

# Sample data

The sample data is defined in [seeddata](https://github.com/dotnet/eShopSupport/tree/main/seeddata). All products/descriptions/brands, manuals, customers, and support tickets names are fictional and were generated using [GPT-35-Turbo](https://learn.microsoft.com/en-us/azure/ai-services/openai/how-to/chatgpt) using the included [DataGenerator](https://github.com/dotnet/eShopSupport/tree/main/seeddata/DataGenerator) project.
