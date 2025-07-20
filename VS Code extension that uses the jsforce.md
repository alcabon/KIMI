Below is a **minimal but complete** VS Code extension that uses the **jsforce** library to connect to Salesforce, run an arbitrary SOQL query, and show the results in a Web-view panel.  
It is written in **TypeScript**, packaged with **webpack**, and follows the official VS Code extension folder layout.

---

### 1. Prerequisites

- VS Code ≥ 1.90  
- Node ≥ 18 (for built-in fetch)  
- Salesforce CLI (for authentication if you want to use an access-token) – optional  
- A Salesforce org with **API Enabled** permission and a user that can obtain an OAuth access-token or username/password + security-token.

---

### 2. Scaffold the extension

```bash
npm install -g yo generator-code
yo code          # choose "New Extension (TypeScript)"
cd sf-jsforce-ext
npm install
npm install jsforce @types/jsforce
```

---

### 3. `package.json` (only the relevant parts)

```jsonc
{
  "name": "sf-jsforce-ext",
  "displayName": "Salesforce JSForce Helper",
  "description": "Run SOQL via jsforce in VS Code",
  "version": "0.0.1",
  "engines": { "vscode": "^1.90.0" },
  "categories": ["Other"],
  "activationEvents": ["onCommand:sf-jsforce-ext.runSoql"],
  "main": "./out/extension.js",
  "contributes": {
    "commands": [
      {
        "command": "sf-jsforce-ext.runSoql",
        "title": "SF: Run SOQL with jsforce"
      }
    ],
    "configuration": {
      "title": "Salesforce JSForce",
      "properties": {
        "sfJsforce.username": { "type": "string", "default": "" },
        "sfJsforce.password": { "type": "string", "default": "" },
        "sfJsforce.token": { "type": "string", "default": "" },
        "sfJsforce.loginUrl": { "type": "string", "default": "https://login.salesforce.com" }
      }
    }
  },
  "scripts": {
    "vscode:prepublish": "npm run package",
    "compile": "webpack",
    "watch": "webpack --watch",
    "package": "webpack --mode production"
  },
  "devDependencies": {
    "@types/vscode": "^1.90.0",
    "ts-loader": "^9.4.4",
    "typescript": "^5.4.5",
    "webpack": "^5.91.0",
    "webpack-cli": "^5.1.4"
  },
  "dependencies": {
    "jsforce": "^1.11.1"
  }
}
```

---

### 4. `webpack.config.js`

```js
const path = require('path');
module.exports = {
  target: 'node',
  mode: 'none',
  entry: './src/extension.ts',
  output: { path: path.resolve(__dirname, 'out'), filename: 'extension.js', libraryTarget: 'commonjs2' },
  externals: { vscode: 'commonjs vscode' },
  resolve: { extensions: ['.ts', '.js'] },
  module: { rules: [{ test: /\.ts$/, exclude: /node_modules/, use: [{ loader: 'ts-loader' }] }] }
};
```

---

### 5. `src/extension.ts`

```ts
import * as vscode from 'vscode';
import * as jsforce from 'jsforce';

export function activate(context: vscode.ExtensionContext) {
  const cmd = vscode.commands.registerCommand('sf-jsforce-ext.runSoql', async () => {
    const cfg = vscode.workspace.getConfiguration('sfJsforce');
    const username = cfg.get<string>('username');
    const password = cfg.get<string>('password');
    const token = cfg.get<string>('token');
    const loginUrl = cfg.get<string>('loginUrl');

    if (!username || !password) {
      vscode.window.showErrorMessage('Please set sfJsforce.username / password in settings.');
      return;
    }

    const soql = await vscode.window.showInputBox({ prompt: 'SOQL Query', placeHolder: 'SELECT Id, Name FROM Account LIMIT 5' });
    if (!soql) return;

    try {
      const conn = new jsforce.Connection({ loginUrl });
      await conn.login(username, password + token);

      const res = await conn.query(soql);
      const panel = vscode.window.createWebviewPanel('soqlResults', 'SOQL Results', vscode.ViewColumn.One, {});
      panel.webview.html = getHtml(JSON.stringify(res, null, 2));
    } catch (err: any) {
      vscode.window.showErrorMessage(`SF Error: ${err.message}`);
    }
  });

  context.subscriptions.push(cmd);
}

function getHtml(json: string): string {
  return `
  <!DOCTYPE html>
  <html>
  <head>
    <meta charset="UTF-8">
    <title>SOQL Results</title>
    <style>body{font-family:Consolas,monospace;background:#1e1e1e;color:#d4d4d4;white-space:pre}</style>
  </head>
  <body>${json}</body>
  </html>`;
}

export function deactivate() {}
```

---

### 6. Build & Run

```bash
npm run compile
code .               # open the extension folder in VS Code
```

Press `F5` → a new **Extension Development Host** window opens.  
Open any workspace → `Ctrl+Shift+P` → **“SF: Run SOQL with jsforce”**.

---

### 7. Packaging & Publishing (optional)

```bash
npm install -g vsce
vsce package          # creates .vsix
vsce publish          # needs Azure DevOps Personal Access Token
```

---

### 8. Security note

- Username/Password + Token is stored in **user-scoped VS Code settings** (`settings.json`).  
- For production use, switch to **OAuth 2.0** (web-server flow, refresh tokens) – jsforce supports it out of the box.

---

That’s a fully working VS Code extension leveraging **jsforce** for Salesforce connectivity. Happy coding!
