# api-client-generator - SPEC v0.1.0

## Project Identity

| Field | Value |
|-------|-------|
| **Name** | api-client-generator |
| **GitHub** | zkzWilliamzhou/api-client-generator |
| **Description** | Generate typed API clients from OpenAPI specifications with built-in caching, error normalization, and zero configuration |
| **Target Stars** | 10K-20K |
| **License** | MIT |
| **Language** | TypeScript (strict) |

---

## Core Value Proposition

**"OpenAPI spec in, production-ready typed client out — zero config, built-in caching, normalized errors."**

- **Problem**: Microservice teams manually write/maintain API clients. Types drift from server. No standard caching. Error handling inconsistent. OpenAPI generators exist (openapi-generator, orval, swagger-codegen) but produce verbose, framework-specific code requiring manual wrapper for caching/error handling.
- **Solution**: CLI + programmatic API that generates a **single-file, dependency-free** client with: typed methods per endpoint, automatic request/response validation, built-in caching (memory + IndexedDB), normalized error classes, retry logic, and interceptors.
- **Unique Angle**: Zero-config opinionated defaults + single-file output + caching built-in + error normalization + works with any fetch implementation (browser, Node, edge runtimes).

---

## User Personas

| Persona | Pain Point | Success Metric |
|---------|------------|----------------|
| **Frontend Lead** | Client types out of sync with API, manual wrapper maintenance | 100% type coverage, zero manual sync |
| **Full-stack Solo Dev** | Wants typed client without learning complex generator config | Generates client in <30 seconds |
| **Platform Team** | Enforce standards across 20+ services | Consistent client patterns org-wide |
| **Mobile/Edge Dev** | Needs client that works in React Native, Cloudflare Workers, Deno | Same client runs everywhere |

---

## Feature Specification

### Phase 1 (v0.1.0) - MVP

#### 1. OpenAPI Parser & Validator
- Support **OpenAPI 3.0.x** and **3.1.x** (JSON/YAML)
- Validate spec completeness: operations, schemas, servers, security
- **Strict mode**: Fail on ambiguous types (e.g., `any`, inline schemas without titles)
- **Reference resolution**: Handle `$ref` across files, external URLs (with cache)

#### 2. Client Generator (Core)
- **Single-file output**: One `.ts` file containing all types + client class
- **Zero runtime dependencies**: Uses native `fetch`, `AbortController`, `Headers`
- **Target environments**: Browser, Node 18+, Deno, Bun, Cloudflare Workers, Vercel Edge
- **Module format**: ESM + CJS dual package exports

#### 3. Generated Client API
```typescript
// Generated structure
class ApiClient {
  constructor(config: ClientConfig);
  
  // Per-operation methods
  getUsers(params: GetUsersParams): Promise<PaginatedResponse<User>>;
  createUser(data: CreateUserRequest): Promise<User>;
  getUserById(id: string): Promise<User>;
  updateUser(id: string, data: UpdateUserRequest): Promise<User>;
  deleteUser(id: string): Promise<void>;
  
  // Interceptors
  requestInterceptors: RequestInterceptor[];
  responseInterceptors: ResponseInterceptor[];
  
  // Cache control
  cache: CacheController;
  
  // Error handling
  onError: (error: ApiError) => void;
}

// Config
interface ClientConfig {
  baseUrl: string;
  fetch?: typeof fetch;           // Custom fetch (for testing, edge runtimes)
  headers?: Record<string, string>; // Default headers
  timeout?: number;               // ms, default 30000
  retry?: RetryConfig;            // Default: 3 retries, exponential backoff
  cache?: CacheConfig;            // Default: memory + IndexedDB
  validateResponse?: boolean;     // Default: true (throw on schema mismatch)
}
```

#### 4. Built-in Caching (`CacheController`)
- **Strategies**: `none` | `memory` | `indexeddb` | `both` (default)
- **Key Generation**: Deterministic from method + path + query + body hash
- **TTL**: Per-operation configurable, default 5min for GET, 0 for mutations
- **Invalidation**: `client.cache.invalidate(pattern)`, `client.cache.clear()`
- **Stale-While-Revalidate**: Return cached immediately, fetch in background
- **Storage Quota**: Auto-evict LRU when IndexedDB > 50MB

#### 5. Error Normalization
```typescript
// All errors extend this base
class ApiError extends Error {
  constructor(
    public readonly status: number,
    public readonly code: string,      // e.g., 'VALIDATION_ERROR', 'NOT_FOUND'
    public readonly message: string,
    public readonly details?: unknown, // Original response body
    public readonly requestId?: string // From response headers
  ) {}
}

// Specific subclasses
class ValidationError extends ApiError { /* 400, field-level details */ }
class NotFoundError extends ApiError { /* 404 */ }
class UnauthorizedError extends ApiError { /* 401 */ }
class ForbiddenError extends ApiError { /* 403 */ }
class ServerError extends ApiError { /* 5xx */ }
class NetworkError extends ApiError { /* fetch failed, timeout */ }
class SchemaValidationError extends ApiError { /* response doesn't match schema */ }
```

#### 6. Type Generation Rules
- **Components → Interfaces**: Each `components.schemas` entry → TypeScript interface
- **Naming**: PascalCase from schema title or normalized key
- **Enums**: TypeScript `enum` for string enums, union types for small sets
- **Arrays**: `T[]` not `Array<T>`
- **Optional**: `?` for optional properties, not `undefined` unions
- **Nullability**: Explicit `T | null` for nullable schemas
- **Discriminated Unions**: `oneOf`/`anyOf` with discriminator → union types
- **Formats**: `date-time` → `string` (ISO), `uuid` → `string`, `binary` → `Blob`/`Uint8Array`

#### 7. CLI Interface
```bash
# Basic usage
api-client-gen --spec ./openapi.yaml --output ./src/api/client.ts

# With config file
api-client-gen --config api-client.config.json

# Watch mode (dev)
api-client-gen --spec ./openapi.yaml --output ./src/api/client.ts --watch

# Options
--spec, -s        OpenAPI spec file (local or URL)
--output, -o      Output file path
--config, -c      Config file path
--watch, -w       Watch for spec changes
--strict          Fail on warnings
--no-cache        Disable cache in generated client
--runtime         Target runtime (browser|node|edge|universal)
--format          Output format (ts|js|d.ts)
--help, -h        Show help
```

#### 8. Programmatic API
```typescript
import { generateClient, GenerateOptions } from 'api-client-generator';

const options: GenerateOptions = {
  spec: openApiSpec,           // Parsed JSON object or string
  specPath: './openapi.yaml',  // For relative $ref resolution
  output: {
    path: './src/api/client.ts',
    format: 'typescript',
    runtime: 'universal',
  },
  client: {
    name: 'MyApiClient',
    baseUrl: 'https://api.example.com',
    // ... ClientConfig options
  },
  generation: {
    strict: true,
    includeValidation: true,
    includeCache: true,
    includeRetries: true,
  },
};

await generateClient(options);
```

#### 9. Config File Schema (`api-client.config.json`)
```json
{
  "spec": "./openapi.yaml",
  "output": "./src/api/client.ts",
  "client": {
    "name": "MyApiClient",
    "baseUrl": "https://api.example.com",
    "timeout": 30000,
    "retry": { "attempts": 3, "backoff": "exponential" },
    "cache": { "strategy": "both", "defaultTtl": 300000 }
  },
  "generation": {
    "strict": true,
    "includeValidation": true,
    "includeCache": true,
    "includeRetries": true,
    "customTemplates": "./templates"
  },
  "hooks": {
    "preGenerate": "./scripts/pre-generate.js",
    "postGenerate": "./scripts/post-generate.js"
  }
}
```

---

### Phase 2 (v0.2.0) - Enhanced

- **React Query / TanStack Query Integration**: `generateClient({ plugins: ['tanstack-query'] })` → hooks
- **Zod/Valibot Schema Validation**: Runtime validation using generated schemas
- **Authentication Helpers**: Built-in Bearer token, API key, OAuth2 flow helpers
- **Request/Response Logging**: Structured logging with configurable levels
- **Mock Server Generation**: Generate MSW handlers from spec for testing
- **Version Diff**: CLI command to compare two specs, show breaking changes

---

### Phase 3 (v1.0.0) - Production

- **Plugin System**: Custom templates, custom type mappers, custom interceptors
- **Monorepo Support**: Generate multiple clients from multiple specs in one run
- **CI/CD Integration**: GitHub Action for auto-regeneration on spec change
- **Visual Schema Explorer**: Web UI to browse spec, test endpoints, view generated types
- **Performance Benchmarks**: Bundle size, parse time, runtime overhead metrics

---

## Technical Architecture

### Stack
- **Language**: TypeScript 5.x (strict mode)
- **Parser**: Custom OpenAPI 3.0/3.1 parser (zero-dep, fast)
- **Code Gen**: AST-based using `typescript` compiler API
- **Testing**: Vitest (unit), Vitest + temp fs (integration)
- **CLI**: Commander.js (consistent with existing projects)
- **Build**: tsup (ESM+CJS, d.ts, minified)
- **Docs**: VitePress

### Project Structure
```
api-client-generator/
├── packages/
│   ├── parser/               # OpenAPI 3.0/3.1 parser, validator, resolver
│   ├── generator/            # Core code generation logic
│   │   ├── templates/        # EJS/string templates for client parts
│   │   ├── type-mapper/      # OpenAPI schema → TypeScript types
│   │   ├── client-builder/   # Assembles final client class
│   │   └── plugins/          # Plugin system (future)
│   ├── cli/                  # Commander CLI, config loader
│   ├── cache/                # CacheController implementation (shared with generated client)
│   ├── errors/               # Error class definitions (shared)
│   └── runtime/              # Runtime utilities (fetch wrapper, retry, etc.)
├── apps/
│   └── playground/           # Web playground (Vite + Vue)
├── tests/
│   ├── fixtures/             # OpenAPI specs for testing
│   ├── unit/                 # Parser, generator, type-mapper tests
│   ├── integration/          # Full generation → compile → run tests
│   └── golden/               # Golden file tests (expected output)
├── docs/                     # VitePress
├── package.json              # pnpm workspace
└── README.md
```

---

## Data Models

### Parsed OpenAPI (Internal)
```typescript
interface ParsedSpec {
  openapi: string;
  info: InfoObject;
  servers: ServerObject[];
  paths: Map<string, PathItemObject>;
  components: {
    schemas: Map<string, SchemaObject>;
    responses: Map<string, ResponseObject>;
    parameters: Map<string, ParameterObject>;
    securitySchemes: Map<string, SecuritySchemeObject>;
  };
  security: SecurityRequirementObject[];
  tags: TagObject[];
}

interface OperationMetadata {
  method: 'get' | 'post' | 'put' | 'patch' | 'delete' | 'head' | 'options' | 'trace';
  path: string;
  operationId: string;
  summary?: string;
  description?: string;
  parameters: ParameterObject[];
  requestBody?: RequestBodyObject;
  responses: Map<string, ResponseObject>;
  security?: SecurityRequirementObject[];
  tags: string[];
  deprecated: boolean;
}
```

### Generation Context
```typescript
interface GenerationContext {
  spec: ParsedSpec;
  config: ResolvedConfig;
  types: Map<string, TypeDefinition>;  // Schema name → TS type
  operations: OperationMetadata[];
  imports: Set<string>;                 // Required imports
}
```

---

## Testing Strategy

### Unit Tests (Vitest) - Target: 95% coverage
- **Parser**: Reference resolution, allOf/oneOf/anyOf, circular refs, external refs
- **Type Mapper**: All OpenAPI types → TS types, edge cases (nullable, discriminators)
- **Client Builder**: Method signatures, generics, interceptors, cache integration
- **Templates**: Snapshot testing for each template output
- **Config**: Validation, defaults, merging, environment overrides

### Integration Tests
- **Full Generation**: Spec fixture → generate → TypeScript compile → no errors
- **Runtime Behavior**: Generated client + mock server (MSW) → requests work, caching works, errors normalized
- **Multi-runtime**: Test generated client in Node, Vitest (jsdom), Cloudflare Workers (miniflare)

### Golden File Tests
- 20+ OpenAPI fixtures (petstore, real-world APIs, edge cases)
- Expected output committed, diff on regression
- Auto-update flag for intentional changes

### Fuzzing
- Random valid OpenAPI specs → generate → must compile
- Property-based testing for type mapper

---

## Security Considerations

- **No Code Execution During Generation**: Pure TypeScript AST manipulation
- **Spec URL Fetching**: Opt-in, timeout 10s, max 5MB, no redirects to private IPs
- **Template Injection**: Templates are pre-compiled, no user template execution in core
- **Generated Client**: No `eval`, no `Function` constructor, strict CSP compatible
- **Dependency Audit**: Zero runtime deps, dev deps audited in CI

---

## Performance Budgets

| Metric | Target |
|--------|--------|
| Parse 10k line spec | < 500ms |
| Generate client | < 1s |
| Output file size (typical API) | < 200 KB |
| Generated client parse time | < 50ms |
| Runtime overhead per call | < 1ms |
| Bundle size (generated client) | < 30 KB gzipped |

---

## CI/CD Pipeline

### GitHub Actions
```yaml
# .github/workflows/ci.yml
jobs:
  lint-typecheck-test:
    runs-on: ubuntu-latest
    steps: [checkout, setup-node, pnpm install, pnpm lint, pnpm typecheck, pnpm test]
  
  integration-test:
    needs: lint-typecheck-test
    runs-on: ubuntu-latest
    strategy:
      matrix:
        runtime: [node, deno, bun, cloudflare]
    steps: [checkout, setup-runtime, pnpm install, pnpm test:integration]
  
  golden-tests:
    needs: lint-typecheck-test
    runs-on: ubuntu-latest
    steps: [checkout, pnpm test:golden]
  
  publish:
    needs: [integration-test, golden-tests]
    if: github.event_type == 'release'
    runs-on: ubuntu-latest
    steps: [checkout, setup-node, pnpm install, pnpm build, npm publish]
```

### Release Process
1. `pnpm changeset` - interactive changelog entry
2. `pnpm version` - version bump, tag
3. CI publishes to npm (provenance, attestations)
4. GitHub Release with changelog
5. VitePress docs auto-deploy

---

## Documentation Structure

```
docs/
├── guide/
│   ├── quick-start.md
│   ├── cli-usage.md
│   ├── programmatic-api.md
│   ├── configuration.md
│   ├── caching.md
│   ├── error-handling.md
│   ├── authentication.md
│   ├── testing.md
│   └── migration.md
├── api/
│   ├── generate-client.md
│   ├── parser.md
│   ├── type-mapper.md
│   └── plugins.md
├── reference/
│   ├── config-schema.md
│   ├── cli-options.md
│   ├── error-classes.md
│   └── cache-controller.md
├── examples/
│   ├── react-query.md
│   ├── nextjs.md
│   ├── cloudflare-workers.md
│   └── testing-with-msw.md
└── contributing.md
```

---

## Review Gates (8+ Rounds)

| Round | Focus | Gate Criteria |
|-------|-------|---------------|
| 1 | Spec Review | This SPEC approved |
| 2 | Parser | Parses all test fixtures, zero errors |
| 3 | Type Mapper | 100% schema coverage, correct TS output |
| 4 | Client Generator | Generated client compiles, basic calls work |
| 5 | Caching | CacheController passes all scenarios |
| 6 | Error Normalization | All error types thrown correctly |
| 7 | CLI + Config | All CLI flags, config file, watch mode work |
| 8 | Multi-runtime | Client works in Node, Browser, Edge, Deno, Bun |
| 9 | Performance | Budgets met, no memory leaks |
| 10 | Docs + Release | All guides complete, CI green, npm publish works |

---

## Risk Mitigation

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| OpenAPI spec complexity | High | High | Strict mode, comprehensive test fixtures |
| TypeScript version drift | Medium | Medium | Pin TS version, test against beta |
| Runtime fetch differences | Medium | High | Abstract fetch wrapper, test matrix |
| Generated code bloat | Medium | Medium | Tree-shaking friendly, dead code elimination |
| Breaking changes in OpenAPI | Low | High | Version detection, migration guide |

---

## Success Metrics

| Metric | 30 Days | 90 Days | 1 Year |
|--------|---------|---------|--------|
| GitHub Stars | 1,000 | 5,000 | 15,000 |
| npm Downloads/week | 5,000 | 25,000 | 100,000 |
| Generated Clients in Prod | 50 | 200 | 1,000 |
| Community Plugins | 0 | 5 | 20 |
| Contributors | 3 | 10 | 30 |

---

**SPEC Status**: DRAFT v0.1 - Pending PM Review Gate 1  
**Next Action**: PM approval → Phase 2 implementation begins