# api-client-generator

Generate typed API clients from OpenAPI specifications.

## Why?

In microservice architectures, keeping client types synchronized with server implementations is a constant struggle. This tool generates fully typed API clients from OpenAPI specs, with built-in caching and error normalization so you never have to manually update client code again.

## Features

- **OpenAPI to typed client**: Generate TypeScript/JavaScript clients from OpenAPI 3.0 specs
- **Built-in caching**: Automatic request/response caching with configurable TTL
- **Error normalization**: Standardized error types across all generated endpoints
- **Zero configuration**: Sensible defaults that work out of the box, customizable when needed
- **CI/CD ready**: Generated clients can be committed or generated as part of your build pipeline

## Quick Start

```bash
# Install dependencies
pnpm install

# Generate client from OpenAPI spec
pnpm generate --spec ./openapi.json --output ./src/api/client.ts

# Or use the programmatic API
import { generateClient } from 'api-client-generator';

const client = generateClient({
  spec: openApiSpec,
  outputPath: './src/api/client.ts',
});
```

## Documentation

- [API Reference](./docs/api.md)
- [Code Generation Workflow](./docs/workflow.md)
- [Migration Guide](./docs/migration.md)

## Roadmap

- v0.1.0: Core OpenAPI parsing + client generation + caching
- v0.2.0: Error normalization + plugin system for custom templates
- v1.0.0: Full CI/CD integration + visual schema editor + TypeScript configuration

## License

MIT

<p align="center">
  <a href="https://github.com/zkzWilliamzhou/api-client-generator">
    <img src="https://img.shields.io/github/stars/zkzWilliamzhou/api-client-generator?style=flat-square" alt="GitHub stars">
  </a>
</p>