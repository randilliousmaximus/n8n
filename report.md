# AWS S3 Vectors compatibility plan for n8n Vector Store node

This report describes what it would take to add support for **AWS S3 Vectors** to the vector store nodes in this repository.

## 1) What exists today

The vector store implementation lives in:

- `packages/@n8n/nodes-langchain/nodes/vector_store/`
- Shared factory + operation handling:
  - `packages/@n8n/nodes-langchain/nodes/vector_store/shared/createVectorStoreNode/createVectorStoreNode.ts`
  - `packages/@n8n/nodes-langchain/nodes/vector_store/shared/createVectorStoreNode/types.ts`

Current providers (examples) include PGVector and Redis:

- `packages/@n8n/nodes-langchain/nodes/vector_store/VectorStorePGVector/VectorStorePGVector.node.ts`
- `packages/@n8n/nodes-langchain/nodes/vector_store/VectorStoreRedis/VectorStoreRedis.node.ts`

These nodes are all built with `createVectorStoreNode(...)` and implement provider-specific logic in:

- `getVectorStoreClient(...)`
- `populateVectorStore(...)`
- optional `releaseVectorStoreClient(...)`

## 2) Main compatibility gap for S3 Vectors

There is currently no S3 Vectors node and no S3 Vectors references in the repo.

This report assumes the target is AWS's newly announced **Amazon S3 Vectors** capability. Before coding, confirm the exact public SDK/service naming used by AWS documentation and package names, then align class/file names accordingly.

Before implementation, confirm whether the LangChain package version used by the repo already includes a production-ready Amazon S3 Vectors class. If not, implement a small adapter that conforms to the LangChain `VectorStore` contract and uses the AWS SDK client for Amazon S3 Vectors APIs.

### 2.1 Do we need to add npm packages?

Short answer: **likely yes, but package choice depends on upstream support status**.

- Current repo state in `packages/@n8n/nodes-langchain/package.json`:
  - has `@langchain/aws`
  - does **not** have a direct S3 or S3 Vectors AWS SDK client dependency
- `@aws-sdk/client-s3` exists in `packages/core/package.json`, but that does not guarantee it should be imported from `@n8n/nodes-langchain` (keep package boundaries explicit).

Recommended dependency decision tree:

1. If LangChain adds native Amazon S3 Vectors support:
   - Prefer upgrading/using the LangChain package that exposes it.
   - Add/update only the LangChain package versions needed.
2. If no native LangChain vector store exists yet:
   - Add the official AWS SDK client package for Amazon S3 Vectors (once publicly available and stable).
   - If AWS only exposes S3 Vectors through existing `@aws-sdk/client-s3` APIs, add/use that package directly in `@n8n/nodes-langchain` instead.
3. Do **not** pre-add speculative package names (for example `@aws-sdk/client-<service-name>`) until confirmed in AWS docs/npm.

## 3) Recommended implementation path (minimal + aligned to existing patterns)

### Step A — Create a new vector store node

Add a new node folder:

- `packages/@n8n/nodes-langchain/nodes/vector_store/VectorStoreAwsS3Vectors/`

Create:

- `VectorStoreAwsS3Vectors.node.ts`
- provider icon file (for example `s3.svg`) in the same folder

Use the same pattern as PGVector/Redis:

1. Define node metadata (`displayName`, `name`, `docsUrl`, `operationModes`)
2. Define fields:
   - shared fields (bucket/index identifiers, region behavior)
   - insert options
   - retrieve/load options (topK/filter settings)
   - update options (if S3 Vectors update-by-id is supported)
3. Implement:
   - `getVectorStoreClient(...)`
   - `populateVectorStore(...)`
   - optional `releaseVectorStoreClient(...)`

### Step B — Credentials strategy

Use existing AWS credentials (`name: 'aws'`) unless S3 Vectors needs auth inputs that are not covered by the shared AWS credential.

If shared AWS creds are sufficient, no new credential type file is needed.

If additional auth fields are required, add a dedicated credential type and register it.

### Step C — Register the node

Update:

- `packages/@n8n/nodes-langchain/package.json`

Add to `n8n.nodes`:

- `dist/nodes/vector_store/VectorStoreAwsS3Vectors/VectorStoreAwsS3Vectors.node.js`

If a new credential was created, also add it to `n8n.credentials`.

### Step D — Operation mode support

`createVectorStoreNode` already supports:

- `insert`
- `load`
- `retrieve`
- `update`
- `retrieve-as-tool`

Start with `insert`, `load`, and `retrieve` as MVP. Add `update` only when S3 Vectors semantics are clear and testable.

### Step E — Tests

Add unit tests mirroring existing vector store tests:

- `packages/@n8n/nodes-langchain/nodes/vector_store/VectorStoreAwsS3Vectors/VectorStoreAwsS3Vectors.node.test.ts`

Test at minimum:

1. client initialization from credentials + node params
2. document insertion path (`populateVectorStore`)
3. retrieve/load path with filtering
4. expected error handling for missing required params

## 4) Concrete file-level checklist

- [ ] Add `VectorStoreAwsS3Vectors.node.ts` (or final AWS official service-aligned name)
- [ ] Add icon file for node UI
- [ ] Register node in `packages/@n8n/nodes-langchain/package.json`
- [ ] (Optional) add S3-specific credential type + register in same package.json
- [ ] Add `VectorStoreAwsS3Vectors.node.test.ts` (or final AWS official service-aligned name)
- [ ] Add docs URL target for the new node page

## 5) Implementation notes to avoid rework

- Reuse `createVectorStoreNode` so operation wiring and node UX stay consistent with all existing vector store nodes.
- Prefer strong types and avoid `any` in new code.
- Keep filter handling explicit (PGVector and Redis both use provider-specific filter behavior).
- Ensure cleanup/disconnect logic is implemented if the S3 Vectors client requires teardown.
- Keep first PR small: provider node + tests + registration. Avoid refactoring shared factory code unless strictly required.

## 6) Validation commands (from package root)

Run from:

- `packages/@n8n/nodes-langchain`

Commands:

```bash
corepack pnpm lint
corepack pnpm typecheck
corepack pnpm test nodes/vector_store/VectorStoreAwsS3Vectors/VectorStoreAwsS3Vectors.node.test.ts
```

If cross-package type breakage appears, run from repo root:

```bash
cd <repo-root>
corepack pnpm build > build.log 2>&1
tail -n 20 build.log
```

## 7) Rough effort estimate

- Node implementation + registration: ~0.5–1.5 days
- Tests + edge-case hardening: ~0.5–1 day
- Documentation and review iteration: ~0.5 day

Total: typically **1.5 to 3 days**, depending on maturity of AWS S3 Vectors SDK/LangChain support.
