# Quickstart: Code Generation Support Tools

**Feature Branch**: `001-codegen-support-tools`
**Date**: 2026-02-02

## Prerequisites

1. CCE built and available
2. Wanaku instance running with data store configured
3. Code generation package uploaded to data store

## Package Preparation

### 1. Create Package Structure

```bash
mkdir -p code-gen-package/{kamelets,templates}

# Create properties file
cat > code-gen-package/config.properties << 'EOF'
available.services=kamelet:http-source,kamelet:kafka-sink,kamelet:log-action
search.tool.description=Searches for available services to build integrations
EOF

# Create sample Kamelet
cat > code-gen-package/kamelets/http-source.kamelet.yaml << 'EOF'
apiVersion: camel.apache.org/v1alpha1
kind: Kamelet
metadata:
  name: http-source
spec:
  definition:
    title: HTTP Source
    description: Fetches data from HTTP endpoint
    properties:
      url:
        type: string
        title: URL
        description: The URL to fetch
EOF

# Create orchestration template
cat > code-gen-package/templates/orchestration.txt << 'EOF'
# Orchestration Template
# Use the following structure to assemble your integration:

- route:
    from:
      uri: "direct:start"
      steps:
        # Add your Kamelets here
        - to: "{{kamelet:source}}"
        - to: "{{kamelet:sink}}"
EOF
```

### 2. Create Archive

```bash
cd code-gen-package
tar -cjf ../code-gen-package.tar.bz2 config.properties kamelets/ templates/
cd ..
```

### 3. Upload to Data Store

Upload `code-gen-package.tar.bz2` to Wanaku data store.

## Running CCE with Code Generation Tools

### Command Line

```bash
java -jar camel-code-execution-engine-app.jar \
  --registration-url http://localhost:8080 \
  --client-id your-client-id \
  --client-secret your-client-secret \
  --data-dir /tmp/cce \
  --codegen-package datastore-archive://code-gen-package.tar.bz2
```

### Docker

```bash
docker run -it \
  -e REGISTRATION_URL=http://wanaku:8080 \
  -e CLIENT_ID=your-client-id \
  -e CLIENT_SECRET=your-client-secret \
  -e CODEGEN_PACKAGE=datastore-archive://code-gen-package.tar.bz2 \
  -v /tmp/cee:/data \
  camel-code-execution-engine:latest
```

## Verifying Tool Registration

After CCE starts, verify tools are registered by checking the service logs. You should see messages indicating successful tool registration:

```
INFO  Tool registered: searchServicesTool
INFO  Tool registered: readKamelet  
INFO  Tool registered: generateOrchestrationCode
```

You can also query the Wanaku discovery service to list available tools (exact endpoint depends on your Wanaku configuration).

## Using the Tools

### Search for Services

```bash
grpcurl -plaintext -d '{
  "uri": "searchServicesTool"
}' localhost:9190 wanaku.capabilities.ToolInvoker/InvokeTool
```

### Read a Kamelet

```bash
grpcurl -plaintext -d '{
  "uri": "readKamelet",
  "arguments": {"name": "http-source"}
}' localhost:9190 wanaku.capabilities.ToolInvoker/InvokeTool
```

### Get Orchestration Template

```bash
grpcurl -plaintext -d '{
  "uri": "generateOrchestrationCode"
}' localhost:9190 wanaku.capabilities.ToolInvoker/InvokeTool
```

**Note**: If you configured a namespace in `config.properties`, prefix the tool names with the namespace (e.g., `ai.wanaku.codegen/searchServicesTool`).

## Troubleshooting

### Tools Not Registered

Check CCE logs for:
- "Failed to download code generation package" - verify datastore URI and package exists
- "Failed to extract archive" - verify archive is valid tar.bz2

### Kamelet Not Found

Verify:
- Kamelet file exists in `kamelets/` directory with `.kamelet.yaml` extension
- Name matches without the extension (e.g., `http-source` for `http-source.kamelet.yaml`)

### Empty Service List

Check `config.properties` contains `available.services` key with comma-separated values.
