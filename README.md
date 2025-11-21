# Aviation Weather MCP Server

This is a Model Context Protocol (MCP) server that provides aviation weather information for flight planning. It connects to the Aviation Weather API to fetch METARs, TAFs, PIREPs, and other data.

## DISCLAIMER

DO NOT USE THIS TOOL FOR FLIGHT PLANNING OR IN-FLIGHT DECISION MAKING.

**IMPORTANT DISCLAIMER**:
This Aviation Weather MCP server provides weather data sourced from aviationweather.gov for informational purposes only. The information provided by this tool should NEVER be used as the sole source for flight planning or in-flight decision making.

Weather data may be incomplete, delayed, or inaccurate. Additionally, the large language model interpreting this data may misunderstand or incorrectly represent critical information. Always consult official aviation weather sources and obtain a proper weather briefing from authorized providers before any flight.

This tool is not FAA-approved, is not a replacement for certified weather services, and should be used only as a supplementary reference. The developers assume no liability for decisions made based on information provided by this tool.

ALWAYS verify critical weather information through official channels.

## Features

- **Type-safe API client** automatically generated from the official Aviation Weather API Swagger definition
- **MCP tools for weather data**:
  - `get-metar`: Get current weather observations
  - `get-taf`: Get terminal aerodrome forecasts
  - `get-pireps`: Get pilot reports near an airport
  - `get-route-weather`: Get comprehensive weather for a route between two airports

## Setup

### Prerequisites

- Node.js 18 or higher
- npm or yarn
- curl (for fetching the Swagger YAML)

### Installation

1. Clone this repository:

   ```bash
   git clone https://github.com/jschabowsky/aviation-mcp.git
   cd aviation-mcp
   ```

2. Install dependencies:

   ```bash
   npm install
   ```

3. Build the server (this will fetch the latest Swagger definition, generate the API client, and compile the TypeScript):

   ```bash
   npm run build
   ```

4. Start the server:

   ```bash
   npm start
   ```



## Example Queries

Once connected to Claude, you can ask questions like:

- "What's the current weather at KJFK?"
- "Is there a TAF available for KORD?"
- "I'm planning to fly from KBOS to KPHL tomorrow. What's the weather looking like?"
- "Are there any PIREPs near KDEN?"


## Deploying to AWS Bedrock Agent Core
### Docker Deployment

1. Create ECR Repository
```
aws ecr create-repository --repository-name mcp-server --region us-east-1
```
2. Build and Push Image to ECR
```
# Get login token
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin [account-id].dkr.ecr.us-east-1.amazonaws.com

docker-buildx build --platform linux/arm64 \
  -t [account-id].dkr.ecr.us-east-1.amazonaws.com/aviation-mcp:latest --push .
```

3. Deploy to Bedrock AgentCore

    - Go to AWS Console → Amazon Bedrock AgentCore → Agent Runtime → Host Agent
      - Provide Name: aviation-mcp
      - Choose ECR Container
        - Image URI: [account-id].dkr.ecr.us-east-1.amazonaws.com/aviation-mcp:latest
    - Inbound Auth:
      - Choose MCP as the protocol
      - Choose: Use JSON Web Tokens (JWT)
    - Host Agent


4. Construct the Encoded ARN MCP URL
Under the "View invocation code", locate the "agentRuntimeArn" variable and copy the value. This is the Agent ARN. In order to consturct the URL you need to URL encode the ARN with the following command

```
agent_arn=<value you copied>
echo $agent_arn | sed 's/:/%3A/g; s/\//%2F/g'
```
The output of the above provides you with an encoded_arn you can use below in order to have the full URL to invoke the hosted MCP agent
```
https://bedrock-agentcore.{region}.amazonaws.com/runtimes/{encoded_arn}/invocations?qualifier=DEFAULT
```

5. Test the Deployed MCP Server
- Use the MCP url with [MCP inspector](https://github.com/modelcontextprotocol/inspector).
- Use a CURL command if security issues prevent you from using inspector. The following example showcases if you setup agent core to use JWT token based authentication: 
```bash
curl -i \
  -X POST \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{
    "jsonrpc": "2.0",
    "id": "1",
    "method": "tools/call",
    "params": {
      "name": "get-metar",
      "arguments": {
        "airport": "KSLC"
      },
      "_meta": {
        "progressToken": 0
      }
    }
  }' \
  "https://bedrock-agentcore.{region}.amazonaws.com/runtimes/{encoded_arn}/invocations?qualifier=DEFAULT"
```

## Development

### Project Structure

- `src/index.ts`: Main server code
- `packages/aviation-weath-api`: Autogenerate API client for Aviation Weather .gov

### Building the aviation weather client

The build process follows these steps:

1. `npm run aviation-weather-api:clean`: delete the existing client
1. `npm run aviation-weather-api:fetch`: Fetches the latest Swagger definition from aviationweather.gov
2. `npm run aviation-weather-api:generate`: Generates a typed TypeScript client from the Swagger definition

### Building and running the app

1. `npm run build`: Build the javascript client
1. `npm run start`: Run the MCP server

### Adding More Tools

To add new tools to the server, follow this pattern:

```typescript
server.tool(
  "tool-name",
  {
    // Zod schema for parameters
    param1: z.string().describe("Parameter description"),
    param2: z.number().optional().describe("Optional parameter")
  },
  async ({ param1, param2 }) => {
    try {
      // Implementation
      return {
        content: [{
          type: "text",
          text: "Response text"
        }]
      };
    } catch (error) {
      return {
        content: [{
          type: "text",
          text: `Error: ${error.message}`
        }],
        isError: true
      };
    }
  }
);
```

## How It Works

1. The server fetches the latest Swagger definition from aviationweather.gov
2. The OpenAPI Generator creates a type-safe client from this definition
3. The server uses this client to make API calls with proper typing
4. Error handling and response formatting ensure a smooth experience

## License

MIT
