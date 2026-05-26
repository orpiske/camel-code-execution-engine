# Code Generation Tools API Contract

**Feature Branch**: `001-codegen-support-tools`
**Date**: 2026-02-02

## Overview

These tools are exposed via gRPC through the existing ToolInvoker service. Tools are invoked using the standard `ToolInvokeRequest`/`ToolInvokeReply` messages defined in the Wanaku SDK.

## Tool Definitions

### searchServicesTool

**URI**: `searchServicesTool` (or `<namespace>/searchServicesTool` if a namespace is configured)

**Description**: (Configurable) Default: "Searches for services to perform the tasks"

**Input Schema**:
```json
{
  "type": "object",
  "properties": {},
  "required": []
}
```

**Response**:
```
# Context
- The list below contains Kamelets that can be used to assemble the orchestration.
- A Kamelet is a snippet for a Camel route.
- It is a snippet that can be used to invoke the service containing the information you are looking for.
- It cannot be used on its own. It MUST be a part of an orchestration flow.
- Kamelets can have arguments. Read the Kamelets before you use them.
- If you don't know what to do, get help.

---
kamelet:service1
kamelet:service2
kamelet:service3
```

**Error Responses**:
| Condition | Error Message |
|-----------|---------------|
| Properties file not found | "Code generation package not loaded" |
| Properties file malformed | "Invalid configuration: {details}" |

---

### readKamelet

**URI**: `readKamelet` (or `<namespace>/readKamelet` if a namespace is configured)

**Description**: "Reads the content of a Kamelet by name"

**Input Schema**:
```json
{
  "type": "object",
  "properties": {
    "name": {
      "type": "string",
      "description": "The name of the Kamelet to read (without .kamelet.yaml suffix)"
    }
  },
  "required": ["name"]
}
```

**Response**: Raw YAML content of the requested Kamelet

**Example Request**:
```json
{
  "arguments": {
    "name": "http-source"
  }
}
```

**Example Response**:
```yaml
apiVersion: camel.apache.org/v1alpha1
kind: Kamelet
metadata:
  name: http-source
spec:
  definition:
    title: HTTP Source
    description: Periodically fetches data from an HTTP endpoint
    properties:
      url:
        title: URL
        description: The URL to fetch
        type: string
  template:
    from:
      uri: timer:tick
      steps:
        - to:
            uri: "http://{{url}}"
```

**Error Responses**:
| Condition | Error Message |
|-----------|---------------|
| Kamelet not found | "Kamelet '{name}' not found" |
| Invalid Kamelet name | "Invalid Kamelet name: {name}" |
| Kamelets directory missing | "Code generation package not loaded" |
| Kamelet file contains malformed YAML | "Invalid Kamelet file: {name}" |

---

### generateOrchestrationCode

**URI**: `generateOrchestrationCode` (or `<namespace>/generateOrchestrationCode` if a namespace is configured)

**Description**: "Returns the orchestration template for code generation"

**Input Schema**:
```json
{
  "type": "object",
  "properties": {},
  "required": []
}
```

**Response**: Raw content of the orchestration template file

**Error Responses**:
| Condition | Error Message |
|-----------|---------------|
| Template file not found | "Orchestration template not found" |
| Code gen package not loaded | "Code generation package not loaded" |

---

## Tool Registration Schema

Tools are registered with Wanaku using the Wanaku SDK's `ToolReference` format. Each tool includes:

- **name**: The tool identifier (e.g., "searchServicesTool")
- **description**: Human-readable description of what the tool does
- **inputSchema**: JSON Schema defining the tool's parameters
- **namespace** (optional): Grouping namespace for the tool

Example registration for `searchServicesTool`:

```
name: searchServicesTool
description: Searches for services to perform the tasks
namespace: ai.wanaku.codegen (if namespace is set in config.properties)
inputSchema: {} (no parameters required)
```

## gRPC Message Reference

Uses existing Wanaku SDK messages:

### ToolInvokeRequest
```protobuf
message ToolInvokeRequest {
  string uri = 1;
  string body = 2;
  map<string, string> arguments = 3;
}
```

### ToolInvokeReply
```protobuf
message ToolInvokeReply {
  string content = 1;
  string error = 2;
  bool is_error = 3;
}
```

## Invocation Flow

```
Agent
  │
  ▼ ToolInvokeRequest(uri="searchServicesTool")
  │
Wanaku Router
  │
  ▼ Route to CCE based on tool registration
  │
CCE (CodeGenToolInvokerService)
  │
  ├── Parse URI to identify tool
  ├── Dispatch to appropriate handler (SearchServicesTool, ReadKameletTool, or GenerateOrchestrationTool)
  ├── Read from extracted package resources
  └── Return ToolInvokeReply(content=result)
```
