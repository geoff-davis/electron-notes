# Setting up an Electron app with Quasar, Vue, and Tailwind CSS

## Installing Quasar

```bash
npm install -g @quasar/cli

mkdir boilerplate
cd boilerplate
quasar create boilerplate-app
npm init quasar
```

```text
✔ What would you like to build? › App with Quasar CLI, let's go!
✔ Project folder: … boilerplate-app
✔ Pick Quasar version: › Quasar v2 (Vue 3 | latest and greatest)
✔ Pick script type: › Typescript
✔ Pick Quasar App CLI variant: › Quasar App CLI with Vite 5 (BETA | next major version - v2)
✔ Package name: … boilerplate-app
✔ Project product name: (must start with letter if building mobile apps) … Boilerplate app
✔ Project description: … A Quasar Project
✔ Author: … Your Name <your-email@address>
✔ Pick a Vue component style: › Composition API with <script setup>
✔ Pick your CSS preprocessor: › Sass with SCSS syntax
✔ Check the features needed for your project: › Linting (vite-plugin-checker + ESLint + vue-tsc), State Management (Pinia), axios, vue-i18n
✔ Pick an ESLint preset: › Prettier
```

```bash
cd boilerplate-app
quasar mode add electron
```

### Commands

Some commands you may need during development:

* Run the app from the command line: `quasar dev -m electron`
* Build the app from the command line: `quasar build -m electron`

## Installing Tailwind CSS

```bash
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
```

`tailwind.config.js`:

```js
/** @type {import('tailwindcss').Config} */
module.exports = {
  content: [
    './src/**/*.vue',
    './src/**/*.js',
    './src/**/*.ts',
    './src/**/*.jsx',
    './src/**/*.tsx',
    './src/**/*.html',
  ],
  theme: {
    extend: {},
  },
  plugins: [],
};
```

## VSCode

### Useful VSCode extensions

* [Vue - Official](https://marketplace.visualstudio.com/items?itemName=Vue.volar)
* [ESLint](https://marketplace.visualstudio.com/items?itemName=dbaeumer.vscode-eslint)
* [Npm Intellisense](https://marketplace.visualstudio.com/items?itemName=christian-kohler.npm-intellisense)
* [Tailwind CSS IntelliSense](https://marketplace.visualstudio.com/items?itemName=bradlc.vscode-tailwindcss)
* [markdownlint](https://marketplace.visualstudio.com/items?itemName=DavidAnson.vscode-markdownlint)
* [Prettier - Code formatter](https://marketplace.visualstudio.com/items?itemName=esbenp.prettier-vscode)
* [Prettier ESLint](https://marketplace.visualstudio.com/items?itemName=rvest.vs-code-prettier-eslint)
* [EditorConfig](https://marketplace.visualstudio.com/items?itemName=EditorConfig.EditorConfig)
* [TODO Highlight](https://marketplace.visualstudio.com/items?itemName=wayou.vscode-todo-highlight)

### VSCode configuration files

These files let you launch and debug your app from within VSCode:

`.vscode/launch.json` (from [here](https://github.com/hawkeye64/electron-quasar-file-explorer-v2/blob/72011a5dedb47b543c7aa2fe9f4e354cc857e1f7/.vscode/launch.json#L6))

```json
{
  // Use IntelliSense to learn about possible attributes.
  // Hover to view descriptions of existing attributes.
  // For more information, visit: https://go.microsoft.com/fwlink/?linkid=830387
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Launch quasar dev -m electron",
      "program": "${workspaceFolder}/node_modules/@quasar/app-vite/bin/quasar.js",
      "args": ["dev", "-m", "electron"],
      "cwd": "${workspaceFolder}",
      "request": "launch",
      "skipFiles": ["<node_internals>/**"],
      "sourceMaps": true,
      "type": "node",
      "preLaunchTask": "quasar dev",
      "env": {
        "ELECTRON_DISABLE_SECURITY_WARNINGS": "true"
      },
      "outputCapture": "std",
      "internalConsoleOptions": "openOnSessionStart"
    },
    {
      "name": "Launch quasar build -m electron",
      "program": "${workspaceFolder}/node_modules/@quasar/app-vite/bin/quasar.js",
      "args": ["build", "-m", "electron"],
      "cwd": "${workspaceFolder}",
      "request": "launch",
      "skipFiles": ["<node_internals>/**"],
      "type": "node"
    }
  ]
}
```

`.vscode/tasks.json`:

```json
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "quasar dev",
      "type": "shell",
      "command": "quasar dev",
      "problemMatcher": [],
      "isBackground": true
    }
  ]
}
```

Add the following file in `./src-electron` to prevent an error alert when launching debug from VSCode:

`src-electron/electron-main.cjs`:

```js
require('ts-node').register();
require('./electron-main.ts');
```

## Configurations

### prettier

Some tweaks to Prettier's JS preferences (prefer single to double quotes, use semicolons at the ends of lines, add trailing commas at the end of lists, enable the Tailwind CSS plugin)

`.prettierrc`:

```json
{
  "singleQuote": true,
  "semi": true,
  "trailingComma": "all",
  "plugins": ["prettier-plugin-tailwindcss"]
}
```

## Setting up TRPC

### Install dependencies

```bash
npm install @trpc/server @trpc/client zod
```

### Install and build a patched electron-trpc

`electron-trpc` lets you use [`trpc`](https://trpc.io/) for inter-process communication. One headache: the current version of `electron-trpc` has problems with the ECMAScript Module settings used by Quasar, and until it's fixed, we need to use a patched version.

Outside of your project, clone a recent fork of `electron-trpc`
and build the package:

```bash
git clone https://github.com/acolvin-h1/electron-trpc

cd electron-trpc/packages/electron-trpc
npm install
npm -r run build
```

### Add the patched electron-trpc to your package dependencies

In your own package, add your patched `electron-trpc` package as a dependency:

```bash
npm install -S ../../electron-trpc/packages/electron-trpc
```

Note that the package above is called electron-trpc-patched in your `package.json`!

Now you can add RPC methods in `src-electron/router`:

`context.ts`

```ts
import { BrowserWindow } from 'electron';

// Define a type for the context
export interface Context {
  window_id: number | null;
}

// Create a function to initialize the context
export const createContext = async ({
  event,
}: {
  event: Electron.IpcMainInvokeEvent;
}): Promise<Context> => {
  const window = BrowserWindow.fromWebContents(event.sender);
  const window_id = window?.id ?? null;
  // Return a Promise that resolves to a Context object
  return Promise.resolve({ window_id });
};
```

`router.ts`:

```ts
import { BrowserWindow } from 'electron';
import { initTRPC } from '@trpc/server';
import { z } from 'zod';
import { Context } from './context';

const t = initTRPC.context<Context>().create({
  isServer: true,
});

export const appRouter = t.router({
  greeting: t.procedure
    .input(z.object({ name: z.string() }))
    .query(({ input }) => {
      return `Hello, ${input.name}!`;
    }),

  setWindowTitle: t.procedure
    .input(z.string().min(1, 'Title cannot be empty'))
    .mutation(({ ctx, input }: { ctx: Context; input: string }) => {
      if (!ctx.window_id) throw new Error('No window found!');
      const window = BrowserWindow.fromId(ctx.window_id)!;
      window.setTitle(input); // Set the window title
      return `Title changed to: ${input}`;
    }),
});

export type AppRouter = typeof appRouter;
```

`ipc-setup.ts`:

```ts
import { BrowserWindow } from 'electron';
import { createIPCHandler } from 'electron-trpc-patched/main';
import { appRouter } from './router';
import { createContext } from './context';

let ipcHandler: ReturnType<typeof createIPCHandler> | undefined;

export function attachWindow(window: BrowserWindow): void {
  if (ipcHandler) {
    ipcHandler.attachWindow(window);
  } else {
    ipcHandler = createIPCHandler({
      router: appRouter,
      windows: [window],
      createContext,
    });
  }
}

export function detachWindow(window: BrowserWindow): void {
  if (!ipcHandler) return;
  ipcHandler.detachWindow(window);
}
```

Wire up a handler to each window in main:

`electron-main.ts`:

```ts
import { attachWindow, detachWindow } from './router/ipc-setup';


// Construct mainWindow here
mainWindow = new BrowserWindow({
  // ...
  webPreferences: {
    contextIsolation: true,
    nodeIntegration: false,
    // You must disable sandboxing if nodeIntegration=false to
    // allow the preload script to set up TRPC
    sandbox: false,  
    ...
  },
  // ...
});
// Attach the window to the IPC handler
attachWindow(mainWindow);
// Remove the IPC handler when the window is closed
mainWindow?.once('close', () => {
  if (mainWindow) {
    detachWindow(mainWindow);
  }
});
```

Expose the handler in preload:

`electron-preload.ts`:

```ts
import { exposeElectronTRPC } from 'electron-trpc-patched/main';

process.once('loaded', async () => {
  exposeElectronTRPC();
});
```

Construct a client in the renderer to call router methods:

```ts
import { createTRPCProxyClient } from '@trpc/client';
import { ipcLink } from 'electron-trpc-patched/renderer';
import type { AppRouter } from '../../src-electron/router/router';

const client = createTRPCProxyClient<AppRouter>({ links: [ipcLink()] });

function changeWindowName() {
  client.setWindowTitle.mutate(window_name.value);
}
```

## Testing

Here we'll set up some basic unit tests with [mocha](https://mochajs.org/) / [chai](https://www.chaijs.com/guide/) / [sinon](https://sinonjs.org/).

[!NOTE]
Jest is another popular testing framework, but [it isn't fully supported by `vite`](https://jestjs.io/docs/getting-started#using-vite). There is a workaround via [vite-jest](https://github.com/haoqunjiang/vite-jest/tree/main), but it has [some important limitations](https://github.com/haoqunjiang/vite-jest/tree/main/packages/vite-jest#limitations-and-differences-with-commonjs-tests).

### Install packages

```bash
npm install -D tsx
npm install -D mocha @types/mocha
npm install -D chai @types/chai
npm install -D sinon @types/sinon
npm install -D sinon-chai @types/sinon-chai
npm install -D chai-as-promised @types/chai-as-promised
npm install -D electron-mocha
```

### Configure mocha

Tell `mocha` to use `tsx` to load TypeScript files. `tsx` greatly simplifies dealing with mixtures of CommonJS and ESM. [More on tsx](https://tsx.is)

`.mocharc.json`:

```json
{
  "$schema": "https://json.schemastore.org/mocharc.json",
  "require": "tsx"
}
```

### Add test scripts

We assume the following file structure:

```text
my-electron-app/
│
├── src-electron/
│   ├── main/                    # Main process code
│   ├── preload/                 # Preload scripts for Electron
│   ├── router/                  # Router and related modules
├── test-electron/               # Place tests under test-electron
│   ├── unit/
│   │   ├── main/                # Unit tests for main process
│   │   ├── preload/             # Unit tests for preload scripts
│   │   └── router/              # Unit tests for router
│   └── integration/             # Integration tests for modules
├── package.json
└── other files...
```

Add the following to `package.json`:

```json
"scripts": {
    "test:unit": "electron-mocha -c test-electron/unit/**/*.test.ts",
    "test:integration": "electron-mocha -c test-electron/integration/**/*.test.ts",
    "test": "npm run test:unit && npm run test:integration"
},
```

### Write some tests

Here are unit tests for the router methods above.

`test-electron/unit/router/router.test.ts`:

```ts
const chai = await import('chai'); // dynamic import because no default
const { expect } = chai;
import sinon from 'sinon';
import sinonChai from 'sinon-chai';
import chaiAsPromised from 'chai-as-promised';
import { BrowserWindow } from 'electron';
import { appRouter } from '../../../src-electron/router/router';
import { Context } from '../../../src-electron/router/context';

// Use Sinon-Chai to enable sinon assertions
chai.use(sinonChai);
chai.use(chaiAsPromised);

describe('appRouter', () => {
  let mockContext: Context;

  beforeEach(() => {
    mockContext = { window_id: 1 }; // Simulate a valid window ID
  });

  afterEach(() => {
    sinon.restore(); // Restore Sinon stubs after each test
  });

  describe('greeting', () => {
    it('should return a greeting message with the given name', async () => {
      const result = await appRouter
        .createCaller(mockContext)
        .greeting({ name: 'Alice' });
      expect(result).to.equal('Hello, Alice!');
    });
  });

  describe('setWindowTitle', () => {
    it('should set the window title when a valid window ID is present', async () => {
      const mockSetTitle = sinon.stub();
      sinon
        .stub(BrowserWindow, 'fromId')
        .returns({ setTitle: mockSetTitle } as unknown as InstanceType<
          typeof BrowserWindow
        >);

      const result = await appRouter
        .createCaller(mockContext)
        .setWindowTitle('New Title');
      expect(mockSetTitle).to.have.been.calledWith('New Title');
      expect(result).to.equal('Title changed to: New Title');
    });

    it('should throw an error if no window ID is present in the context', async () => {
      mockContext.window_id = null;

      await expect(
        appRouter.createCaller(mockContext).setWindowTitle('Title'),
      ).to.be.rejectedWith('No window found!');
    });
  });
});
```

### Run the tests

```bash
npm run test  # run all tests
npm run test:unit  # run unit tests
npm run test:integration  # run integration tests
```
