# electron-trpc notes

[Electron in Action](https://www.manning.com/books/electron-in-action) is a great introduction to Electron, with one big exception. The book was published in 2018, and Electron underwent some big security changes a few years later that greatly restrict renderer process access to OS-level methods.

The Electron documentation recommends using [inter-process communication](https://www.electronjs.org/docs/latest/tutorial/ipc) to let the renderer call methods in the main process. The recommended approach has a few downsides, however:

* It's clunky and involves a lot of boilerplate in a privileged preload file.
* IPC calls aren't type safe and don't do any validation on arguments.

Several people in [r/electronjs](https://www.reddit.com/r/electronjs/search/?q=electron-trpc) have expressed enthusiasm about using [tRPC](https://trpc.io/), a popular RPC library, via [electron-trpc](https://github.com/jsonnull/electron-trpc) as an alternative. tRPC is well-documented, but the documentation for the Electron integration is sparse, especially if you're not using React. Here's how I set up `electron-trpc` with vanilla TypeScript.

## Installation

tRPC uses [Zod](https://zod.dev/) to validate RPC arguments, so we have to install that, too

```bash
cd myappname
npm install -D electron-trpc
npm install -D zod  # used for validation of rpc arguments
```

### Simple usage

The simplest use of electron-trpc is pretty straightforward.

#### Writing a router

First, we'll create a `router` directory in `myappname/src`, since the router will need to be imported by both the main and renderer processes.

We'll put our router in `myappname/src/router/index.ts`.

Our `greeting` method expects an object with a string property called `name`. We use `tRPC` to do parameter validation via tRPC's input parser (the `.input` method call), then we indicate to tRPC that our method call is idempotent and has no side effects via the `.query` method.

```typescript
import { initTRPC } from '@trpc/server';
import { z } from 'zod';

const t = initTRPC().create({
  isServer: true,
});

export const appRouter = t.router({
  greeting: t.procedure
    .input(z.object({ name: z.string() }))
    .query(({ input }) => {
    return `Hello, ${input.name}!`;
  }),
});

export type AppRouter = typeof appRouter;
```

#### Wiring up the router to IPC in main

Next we need a way to attach an IPCHandler to a BrowserWindow when it is constructed and to detach the IPCHandler when the window is closed. We'll put this in `myappname/src/main/ipc_setup.ts` (code below is from [here](https://github.com/Alduino/Muzik/blob/980b070e80d2ab2477ecbc7a65d0d44ca2d08687/apps/desktop2/electron/main/ipc-setup.ts))

```typescript
import { BrowserWindow } from 'electron';
import { createIPCHandler } from 'electron-trpc/main';
import { appRouter } from '../router';

let ipcHandler: ReturnType<typeof createIPCHandler> | undefined;

export function attachWindow(window: BrowserWindow): void {
  if (ipcHandler) {
    ipcHandler.attachWindow(window);
  } else {
    ipcHandler = createIPCHandler({
      router: appRouter,
      windows: [window],
    });
  }
}

export function detachWindow(window: BrowserWindow): void {
  if (!ipcHandler) return;
  ipcHandler.detachWindow(window);
}
```

In `myappname/src/main/init.ts`, we'll call `attachWindow` when we create new `BrowserWindow`s and `detachWindow` when we close them.

```typescript
import { attachWindow, detachWindow } from './ipc_setup';

function createWindow(): void {
  const window = new BrowserWindow({
    width: 800,
    height: 600,
    show: false,
    autoHideMenuBar: true,
    webPreferences: {
      nodeIntegration: false,
      contextIsolation: true,
      sandbox: false,
      preload: join(__dirname, '../preload/index.js'),
    },
  });

  // Attach the window to the IPC handler
  attachWindow(window);

  // Remove the IPC handler when the window is closed
  window.once('close', () => {
    detachWindow(window);
  });

  window.on('ready-to-show', () => {
    window.show();
  });

  // Your code here
}
```

#### Exposing the router to the renderer in preload

You need just a little bit of boilerplate in `myappname/src/preload/index.ts` to expose TRPC to the renderer:

```typescript
import { exposeElectronTRPC } from 'electron-trpc/main';

process.once('loaded', async () => {
  exposeElectronTRPC();
});
```

### Calling a router method in the renderer

In `renderer.ts`, you'll need to create a tRPC client object which you can then use to call router methods:

```typescript
import { createTRPCProxyClient } from '@trpc/client';
import { ipcLink } from 'electron-trpc/renderer';
import type { AppRouter } from '../../router/index';

// create the client object
export const client = createTRPCProxyClient<AppRouter>({
  links: [ipcLink()],
});

// use the client to call a router method
const myButton = document.getElementById('my-button')!;

myButton.addEventListener('click', () => {
  client.greeting.query({ name: 'John' }).then((result) => {
    console.log(result);
  });
});
```

Voila!

### More advanced usage: context objects

If you've used the standard Electron IPC calls, you'll notice one thing missing: the Electron IPC calls automatically pass an `event` object through to methods in main, and you can use this object to get the calling renderer window (which you'll need for some OS-level functionality) via `BrowserWindow.fromWebContents(event.sender)`. In contrast, router methods in `electron-trpc` just get the arguments passed in explicitly by the caller. What do we do if we need access to the calling renderer window in main?

The solution is to use a tRPC `context` object, an object that holds data that will be made available to all your router methods such as the ID of the renderer BrowserWindow, authentication information, etc.

#### Creating a context object

We'll create a context object in `myappname/src/router/context.ts`:

```typescript
import { BrowserWindow } from 'electron';

// Define a type for the context
export interface Context {
  currentWindow: BrowserWindow | null;
}

// Create a function to initialize the context
export const createContext = async ({
  event,
}: {
  event: Electron.IpcMainInvokeEvent;
}): Promise<Context> => {
  // Get the BrowserWindow from the event
  const currentWindow = BrowserWindow.fromWebContents(event.sender);
  return Promise.resolve({ currentWindow }); // Store the BrowserWindow in the context
};
```

#### Exposing the context object to tRPC

We'll need two small changes to make our context object available in tRPC:

First, we need to define the context object's type when we initialize tRPC in `myappname/src/router/index.ts`:

```typescript
const t = initTRPC.context<Context>().create({
  isServer: true,
});
```

Second, we update `attachWindow` method in `myappname/src/main/ipc_setup.ts` so that a context object gets created when the handler is attached to a window:

```typescript
import { createContext } from '../router/context';

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
```

Now we can use the context in our router methods. For example,
the new setWindowTitle method below has access to the calling
renderer `BrowserWindow` via the context.

Two things to note:

1) `setWindowTitle` has side effects, so we declare it as a tRPC
`mutation` rather than a `query`
2) While the `context` object is available to router methods,
it is an optional argument in method signatures. Our `greeting`
method doesn't use the context, so we don't add it to the signature.

```typescript
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
      if (!ctx.currentWindow) throw new Error('No window found!');
      ctx.currentWindow.setTitle(input); // Set the window title
      return `Title changed to: ${input}`;
    }),
});

export type AppRouter = typeof appRouter;
```
