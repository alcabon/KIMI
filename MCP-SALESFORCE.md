Below is a **copy-paste-ready recipe** for building a **Node.js + TypeScript MCP server** that returns **Salesforce metadata** and is immediately usable in **GitHub Copilot inside VS Code**.

---

### 1. Scaffold the project

```bash
mkdir mcp-salesforce-meta && cd mcp-salesforce-meta
npm init -y
npm i -D typescript @types/node tsx
npm i @modelcontextprotocol/sdk jsforce dotenv
```

`tsconfig.json`

```jsonc
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "Node16",
    "moduleResolution": "node16",
    "outDir": "dist",
    "strict": true,
    "esModuleInterop": true
  },
  "include": ["src/**/*"]
}
```

---

### 2. Create the MCP server (`src/index.ts`)

```ts
#!/usr/bin/env node
import { Server } from '@modelcontextprotocol/sdk/server/index.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
} from '@modelcontextprotocol/sdk/types.js';
import * as jsforce from 'jsforce';

/* ---------- 1.  read Salesforce creds from env ---------- */
require('dotenv').config();
const {
  SF_LOGIN_URL = 'https://login.salesforce.com',
  SF_USERNAME,
  SF_PASSWORD,
  SF_TOKEN,
} = process.env;

if (!SF_USERNAME || !SF_PASSWORD || !SF_TOKEN) {
  console.error('Missing Salesforce env vars');
  process.exit(1);
}

/* ---------- 2.  build jsforce connection ---------- */
async function getConn() {
  const conn = new jsforce.Connection({ loginUrl: SF_LOGIN_URL });
  await conn.login(SF_USERNAME, SF_PASSWORD + SF_TOKEN);
  return conn;
}

/* ---------- 3.  MCP server boilerplate ---------- */
const server = new Server(
  {
    name: 'salesforce-meta-mcp',
    version: '1.0.0',
  },
  {
    capabilities: {
      tools: {},
    },
  }
);

/* ---------- 4.  expose one tool: describe metadata ---------- */
server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [
    {
      name: 'describe_sobject',
      description: 'Return field-level metadata for a Salesforce object',
      inputSchema: {
        type: 'object',
        properties: {
          sobject: { type: 'string', description: 'API name of the object (e.g. Account)' },
        },
        required: ['sobject'],
      },
    },
  ],
}));

server.setRequestHandler(CallToolRequestSchema, async (req) => {
  if (req.params.name !== 'describe_sobject') throw new Error('Unknown tool');

  const sobject = req.params.arguments?.sobject as string;
  const conn = await getConn();
  const meta = await conn.sobject(sobject).describe();

  return {
    content: [
      {
        type: 'text',
        text: JSON.stringify(
          {
            label: meta.label,
            fields: meta.fields.map((f: any) => ({
              name: f.name,
              label: f.label,
              type: f.type,
              length: f.length,
              required: !f.nillable,
            })),
          },
          null,
          2
        ),
      },
    ],
  };
});

/* ---------- 5.  run with stdio transport ---------- */
const transport = new StdioServerTransport();
server.connect(transport);
```

Add the shebang and make it executable:

```bash
chmod +x dist/index.js
```

---

### 3. Build & test locally

```bash
npx tsx src/index.ts        # quick smoke test
npm run build               # tsc
```

---

### 4. Wire it into VS Code / GitHub Copilot

Create `.vscode/mcp.json` in your repo root:

```json
{
  "inputs": [
    {
      "type": "promptString",
      "id": "sf-username",
      "description": "Salesforce username"
    },
    {
      "type": "promptString",
      "id": "sf-password",
      "description": "Salesforce password + security token",
      "password": true
    }
  ],
  "servers": {
    "salesforce-meta": {
      "type": "stdio",
      "command": "node",
      "args": ["./dist/index.js"],
      "env": {
        "SF_USERNAME": "${input:sf-username}",
        "SF_PASSWORD": "${input:sf-password}"
      }
    }
  }
}
```

> Copilot → Agent Mode → “Add More Tools…” will automatically discover this file and start the server  .

---

### 5. Use it in chat

> **Prompt:**  
> `@github describe_sobject("Opportunity")`

Copilot will invoke your MCP server and return a concise JSON summary of every Opportunity field, types, and whether it is required.

---

### 6. Packaging & publishing (optional)

- `npm pack` or containerize with a 10-line Dockerfile.  
- Add a second tool for SOQL queries (`query_records`) by reusing the same connection logic.

Happy shipping!
