# NestJS MCP Server Module

[![smithery badge](https://smithery.ai/badge/@rekog-labs/Nest-MCP)](https://smithery.ai/server/@rekog-labs/Nest-MCP)

A NestJS module for exposing your services as an MCP (Model Context Protocol) server with Server-Sent Events (SSE) transport. This package simplifies exposing tools that clients can discover and execute via SSE.

## Features

- **SSE Transport**: Built-in `/sse` endpoint for streaming and `/messages` for handling tool execution
- **Tool Discovery**: Automatically discover and register tools using decorators
- **Tool Request Validation**: Define Zod schemas to validate tool requests.
- **Progress Notifications**: Send continuous progress updates from tools to clients.

## Installation

### Installing via Smithery

To install NestJS MCP Server Module for Claude Desktop automatically via [Smithery](https://smithery.ai/server/@rekog-labs/Nest-MCP):

```bash
npx -y @smithery/cli install @rekog-labs/Nest-MCP --client claude
```

### Manual Installation
```bash
npm install @rekog/mcp-nest reflect-metadata @modelcontextprotocol/sdk zod
```

## Quick Start

### 1. Import Module

```typescript
// app.module.ts
import { Module } from '@nestjs/common';
import { McpModule } from '@rekog/mcp-nest';
import { GreetingTool } from './greeting.tool';

@Module({
  imports: [
    McpModule.forRoot({
      name: 'my-mcp-server',
      version: '1.0.0',
      capabilities: {}
    })
  ],
  providers: [GreetingTool]
})
export class AppModule {}
```

### 2. Define Tools

```typescript
// greeting.tool.ts
import { Injectable } from '@nestjs/common';
import { Tool } from '@rekog/mcp-nest';
import { z } from 'zod';
import { Context } from '@rekog/mcp-nest/dist/services/mcp-tools.discovery';
import { Progress } from "@modelcontextprotocol/sdk/types.js";

@Injectable()
export class GreetingTool {
  constructor() {}

  @Tool({
    name: 'hello-world',
    description: 'Returns a greeting and simulates a long operation with progress updates',
    parameters: z.object({
      name: z.string().default('World'),
    }),
  })
  async sayHello({ name }, context: Context) {

      const greeting = `Hello, ${name}!`;

      const totalSteps = 5;
      for (let i = 0; i < totalSteps; i++) {
        await new Promise((resolve) => setTimeout(resolve, 500));

        // Send a progress update.
        await context.reportProgress({
          progress: (i + 1) * 20,
          total: 100,
        } as Progress);
      }

      return {
        content: [{ type: 'text', text: greeting }]
      };
  }
}
```

### 3. Start Server

```typescript
// main.ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(3000);
}
bootstrap();
```

## Client Connection

Clients can connect using the MCP SDK:

```typescript
import { Client } from '@modelcontextprotocol/sdk/client';
import { SSEClientTransport } from '@modelcontextprotocol/sdk/client/sse';
import { z } from 'zod';

const client = new Client(
  { name: 'client-name', version: '1.0.0' },
  { capabilities: {} }
);

await client.connect(
  new SSEClientTransport(new URL('<http://localhost:3000/sse>'))
);

// Execute the 'hello-world' tool with progress tracking
const greetResult = await client.callTool(
  {
    name: 'hello-world',
    arguments: {
      name: 'MCP User',
    },
  },
  undefined, // responseSchema is optional, or you can define a Zod schema here
  {
    onprogress: (progress) => {
      console.log(
        `Progress: ${progress.progress}/${progress.total}`
      );
    },
  },
);

console.log(greetResult.content[0].text); // Output: Hello MCP User! I'm Test User from Test Org.
```

## API Endpoints

- `GET /sse`: SSE connection endpoint
- `POST /messages`: Tool execution endpoint

## Configuration Reference

### `McpOptions`

| Property             | Type                      | Description                                                                 | Default Value |
|----------------------|---------------------------|-----------------------------------------------------------------------------|---------------|
| `name`               | string                    | Server name                                                                 | -             |
| `version`            | string                    | Server version                                                              | -             |
| `capabilities`       | Record<string, any>       | Server capabilities, defines what the server can do.                        | `{}`          |
| `sseEndpoint`        | string (optional)         | Endpoint for SSE connections.                                               | `'sse'`       |
| `messagesEndpoint`   | string (optional)         | Endpoint for handling tool execution.                                        | `'messages'`  |
| `globalApiPrefix`    | string (optional)         | Global API prefix for all endpoints.                                        | `''`          |

### Tool Decorator

The `@Tool` decorator is used to define a method as an MCP tool.

```typescript
@Tool({ name: string, description: string, parameters?: z.ZodObject<any> })
```

- `name`: The name of the tool. This will be used to list it in the `listTools` request.
- `description`: A description of the tool.
- `parameters`: (Optional) A Zod schema defining the expected structure of the tool's input arguments.
