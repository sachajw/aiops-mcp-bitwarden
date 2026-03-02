# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Quick Reference

```bash
npm run build          # Compile TypeScript to dist/
npm test               # Run all tests
npm run test:watch     # Watch mode for TDD
npm test -- security   # Run specific test file
npm run lint           # Check code style
npm run lint:fix       # Auto-fix linting issues
npm run inspect        # Test with MCP Inspector (interactive debugging)
```

## Architecture

This MCP server provides two interfaces to Bitwarden:

```
AI Client → MCP Protocol → index.ts (routing)
                              ↓
              ┌───────────────┴───────────────┐
              ↓                               ↓
         CLI Interface                   API Interface
         (vault ops via bw)              (org admin via HTTP)
              ↓                               ↓
         executeCliCommand()            executeApiRequest()
         (src/utils/cli.ts)             (src/utils/api.ts)
```

**CLI Interface**: Wraps Bitwarden CLI (`bw`) for personal vault operations. Requires `BW_SESSION` env var.

**API Interface**: Makes OAuth2-authenticated HTTP requests to Bitwarden Public API. Requires `BW_CLIENT_ID` and `BW_CLIENT_SECRET`.

## Source Structure

```
src/
├── index.ts              # Entry point, tool routing (switch statement)
├── tools/
│   ├── cli.ts            # CLI tool definitions (name, description, inputSchema)
│   ├── api.ts            # API tool definitions
│   └── index.ts          # Exports cliTools and organizationApiTools arrays
├── schemas/
│   ├── cli.ts            # Zod schemas for CLI tool inputs
│   └── api.ts            # Zod schemas for API tool inputs
├── handlers/
│   ├── cli.ts            # CLI tool implementations
│   └── api.ts            # API tool implementations
└── utils/
    ├── cli.ts            # executeCliCommand(), buildSafeCommand()
    ├── api.ts            # executeApiRequest(), getAccessToken()
    ├── security.ts       # sanitizeInput(), validateFilePath()
    ├── validation.ts     # withValidation(), validateInput()
    └── config.ts         # Environment variable extraction
```

## Security Pipeline (MANDATORY)

**Every tool must follow this pattern - NO EXCEPTIONS:**

```typescript
// 1. Define Zod schema (src/schemas/cli.ts or api.ts)
const mySchema = z.object({
  name: z.string().min(1),
});

// 2. Handler with withValidation wrapper (src/handlers/cli.ts or api.ts)
export const handleMyTool = withValidation(mySchema, async (validatedArgs) => {
  // 3. Execute via safe command execution
  const result = await executeCliCommand('command', [validatedArgs.name]);
  // OR for API: await executeApiRequest('/public/endpoint', 'GET');

  // 4. Return formatted response
  return {
    content: [{ type: 'text', text: result.output || result.errorOutput }],
    isError: !!result.errorOutput,
  };
});
```

**Key Security Functions:**
- `executeCliCommand(cmd, args)` - Single entry point for CLI; uses `spawn()` with argument arrays
- `validateFilePath(path)` - Path traversal protection (required for all file operations)
- `executeApiRequest(endpoint, method, data?)` - Authenticated HTTP wrapper
- `withValidation(schema, handler)` - Eliminates validation boilerplate

**Security Rules:**
- NEVER use `exec()` or string interpolation for commands
- ALWAYS validate inputs with Zod schemas before processing
- All file operations MUST go through `validateFilePath()`

## Adding a New Tool

1. **Schema** (`src/schemas/cli.ts` or `api.ts`):
   ```typescript
   export const myToolSchema = z.object({
     field: z.string().min(1),
   });
   ```

2. **Tool Definition** (`src/tools/cli.ts` or `api.ts`):
   ```typescript
   export const myTool: Tool = {
     name: 'my_tool',
     description: 'When to use this tool',
     inputSchema: { type: 'object', properties: { field: { type: 'string' } }, required: ['field'] },
   };
   ```

3. **Export** (`src/tools/index.ts`): Add to appropriate array

4. **Handler** (`src/handlers/cli.ts` or `api.ts`):
   ```typescript
   export const handleMyTool = withValidation(myToolSchema, async (validatedArgs) => {
     // Implementation
   });
   ```

5. **Register** (`src/index.ts`): Add case to switch statement

6. **Tests** (`tests/cli-commands.spec.ts` or `tests/api.spec.ts`)

## Testing

**Test Structure:**
```
tests/
├── core.spec.ts          # Server initialization
├── validation.spec.ts    # Schema validation
├── security.spec.ts      # Input sanitization
├── cli-commands.spec.ts  # CLI tool tests
├── api.spec.ts           # API tool tests
└── utils/                # Utility function tests
```

**Test Environment:** Mocked env vars in `.jest/setEnvVars.js` (`BW_SESSION`, `BW_CLIENT_ID`, `BW_CLIENT_SECRET`)

**Running Specific Tests:**
```bash
npm test -- validation.spec.ts          # Specific file
npm test -- --testNamePattern="schema"  # Pattern match
```

## CLI Tool Patterns

**Simple command:**
```typescript
export const handleSync = withValidation(syncSchema, async () => {
  return executeCliCommand('sync', []);
});
```

**With parameters:**
```typescript
export const handleList = withValidation(listSchema, async (validatedArgs) => {
  const params = [validatedArgs.type];
  if (validatedArgs.search) params.push('--search', validatedArgs.search);
  return executeCliCommand('list', params);
});
```

**Base64-encoded JSON:**
```typescript
const encoded = Buffer.from(JSON.stringify(item)).toString('base64');
return executeCliCommand('create', ['item', encoded]);
```

## API Tool Patterns

**GET request:**
```typescript
return executeApiRequest('/public/collections', 'GET');
```

**With path parameter:**
```typescript
return executeApiRequest(`/public/collections/${validatedArgs.collectionId}`, 'GET');
```

**PUT with body:**
```typescript
const { collectionId, ...body } = validatedArgs;
return executeApiRequest(`/public/collections/${collectionId}`, 'PUT', body);
```

**With query parameters:**
```typescript
const params = new URLSearchParams();
if (validatedArgs.start) params.append('start', validatedArgs.start);
return executeApiRequest(`/public/events?${params}`, 'GET');
```

## Code Style

- **Formatting**: Prettier (runs via husky pre-commit)
- **Linting**: ESLint with `npm run lint` / `npm run lint:fix`
- **Imports**: Explicit `.js` extensions required (ES modules)
- **Naming**: camelCase variables/functions, PascalCase types
- **TypeScript**: Strict mode, no `any` types

## References

- [Bitwarden CLI Reference](https://bitwarden.com/help/cli/)
- [Bitwarden Public API Swagger](https://bitwarden.com/help/public-api/)
- [MCP Specification](https://modelcontextprotocol.io/)
- [Zod Documentation](https://zod.dev/)
