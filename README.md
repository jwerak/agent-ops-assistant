# Quick Start Guide

## TL;DR

```bash
cd ops-assistant

# Build container
podman build -t ops-incident-assistant .

# Run container (with color output)
podman run -t --network=host --env-file .env ops-incident-assistant
```

You may see deprecation warnings - these are expected and the application will work correctly.

## What to Expect

### Normal Startup Messages

```
Warning: Could not import langchain_litellm (AttributeError: ...)
Falling back to langchain_community.chat_models.ChatLiteLLM (deprecated but functional)
```
**This is normal** - the app automatically uses a fallback for compatibility.

```
Error initializing MCP tools: ...
Running without MCP tools - agent will have limited capabilities
```
**This is expected** if MCP server is not accessible. The agent will still run.

```
LangChainDeprecationWarning: The class ChatLiteLLM was deprecated...
```
**Safe to ignore** - functionality remains the same.

### Successful Startup

When everything is working, you'll see:

```
🚀 Starting Ops Incident Assistant
   Host: 0.0.0.0:5678
   Model: your-model-name
   MCP Server: https://your-mcp-server/mcp
   Recursion Limit: 50
   Tool Retry Limit: 3
Successfully initialized N MCP tools
Tool retry configuration: max 3 retries with exponential backoff
Available tools: ['get_job_templates', 'launch_job_template', ...]
ReAct agent created successfully
INFO: Started server process
INFO: Waiting for application startup
INFO: Application startup complete
```

### Understanding the Colored Output

When the agent processes a question, you'll see **real-time colored output**:

- **🟢 Green**: Questions and system information
- **🟡 Yellow**:
  - `[Step N] 🤖 LLM calling tools:` - When the LLM decides to use tools
  - `[Step N] 🔧 Tool 'name' response:` - Results from tool execution
- **🔵 Cyan**:
  - `[Step N] 💭 LLM response:` - LLM's reasoning and thoughts
  - `✅ FINAL ANSWER:` - The final response to the user
- **🔴 Red**: Errors and exceptions
- **🟣 Magenta**: Warnings and deprecation notices

This helps you see the **ReAct cycle** in action:
1. LLM thinks and decides which tools to call (yellow 🤖)
2. Tools execute and return results (yellow 🔧)
3. LLM processes results and reasons (cyan 💭)
4. Repeat until LLM has enough information to answer (cyan ✅)

## Testing the Agent

### Quick Test

```bash
set -a
source .env
set +a

curl -X POST http://localhost:5678/webhook/$WEBHOOK_PATH \
  -H "Content-Type: application/json" \
  -d '{"question": "What job templates are available?"}'

curl -X POST http://localhost:5678/webhook/$WEBHOOK_PATH \
  -H "Content-Type: application/json" \
  -d @../prompts/01-disk-full.json
```

Response:
```json
{
  "answer": "I called get_job_templates() and found the following templates: ..."
}
```

### Health Check

```bash
curl http://localhost:5678/health
```

Response:
```json
{
  "status": "healthy",
  "agent_initialized": true
}
```

## Environment Variables

Create a `.env` file copy from `env.example`.

## Deployment

### Run with Podman

```bash
# Build
podman build -t ops-incident-assistant .

# Run with color output (recommended)
podman run -t --network=host --env-file .env ops-incident-assistant

# Or run in detached mode
podman run -d --network=host --env-file .env ops-incident-assistant
```

### Deploy on OpenShift/k8s

See more details on Kubernetes deployment in `ocp/ops-assistant/`.

#### Deploy Using Kustomize

Edit `secret.yaml` to add your OpenAI API key:

```bash
oc create secret generic ops-assistant-secrets \
  --from-literal=openai-api-key='your-actual-api-key'
```

Edit `configmap.yaml` to configure your endpoints:

```yaml
data:
  openai-base-url: "https://your-openai-endpoint.com/v1"
  mcp-server-url: "https://mcp-server-aap.apps.your-cluster.com/mcp"
  model-name: "DeepSeek-R1-Distill-Qwen-14B-W4A16"
  webhook-path: "7d1a79c6-2189-47d5-92c6-dfbac5b1fa59"
```

```bash
oc new-project aiops
```

```bash
# Deploy all resources
oc apply -k .

# Or deploy individually
oc apply -f configmap.yaml
oc apply -f secret.yaml
oc apply -f deployment.yaml
oc apply -f service.yaml
oc apply -f route.yaml
```

#### Testing the Deployment

Once deployed, get the route URL and test:

```bash
# Get the route
ROUTE=$(oc get route ops-incident-assistant -o jsonpath='{.spec.host}')

# Test health endpoint
curl https://$ROUTE/health

# Send a test question
curl -X POST https://$ROUTE/webhook/7d1a79c6-2189-47d5-92c6-dfbac5b1fa59 \
  -H "Content-Type: application/json" \
  -d '{"question": "What job templates are available?"}'
```
