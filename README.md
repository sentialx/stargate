# Stargate

## Node.js workers

### Example usage

Let's first define our tunnels between Worker and Main thread, there are two ways to do that.
First one is to directly use `ContextTunnel` without any implementation-specific logic and manually define
its specifics.

```typescript
const workerClassTunnel = new ContextTunnel<WorkerClass>("worker_class");
const mainClassTunnel = new ContextTunnel<MainClass>("main");

// Separate library for WorkerPool, although this library was designed with worker pools in mind too.
const workerPool = new WorkerPool(4);

class WorkerClass {
  public myFunction(x: number): number {
    const mainClass = mainClassTunnel.connect(workerPool.main.port);
    return await mainClass.myMainFunction() + x;
  }
}

class MainClass {
  public myMainFunction() {
    return 2;
  }
}

if (workerPool.isMain()) {
  workerPool.spawn();

  const worker = workerPool.getFreeWorker();

  // We handle incoming tunnels on the worker object.
  const gate = new WorkerGate(worker)
    .incoming(mainClassTunnel, new MainClass());

  // Establish a connection to the worker and get the resulting proxy object.
  const myClass = myClassTunnel.connect(gate);

  // Finally, make a call to the function defined in the worker.
  await myClass.myFunction(2);
} else {
  const mainGate = new MainThreadGate(workerPool.main.port);

  // Handle incoming tunnels on the parentPort object.
  mainGate
    .incoming(workerClassTunnel, new WorkerClass());
}
```

Our `WorkerPool` library provides with a `TunneledWorkerPool` API which simplifies the process of creating a worker pool with tunneled communication.

```typescript
const workerPool = new TunneledWorkerPool(4, {
  main: MainClass,
  worker: WorkerClass,
});

class WorkerClass {
  public myFunction(x: number): number {
    const mainClass = workerPool.main.api();
    return await mainClass.myMainFunction() + x;
  }
}

class MainClass {
  public myMainFunction() {
    return 2;
  }
}

if (workerPool.isMain()) {
  workerPool.spawn();

  workerPool.getFree().api().myFunction(2);
}

```

## Electron renderer and main processes

This time, proxied objects cannot be used in context of Electron IPC, as the wrapped objects and functions reside in side-specific scopes, e.g. a function that does something with the main process,
has imports that are not available in the renderer process. They cannot be imported from both sides.

This is why `ContextTunnel` was implemented. A `ContextTunnel` defines a tunnel which is a specification of functions that can be invoked or handled by the other side. In some cases they also specify their behavior e.g. which methods are synchronous or asynchronous in `ElectronTunnel`.

The defined `ContextTunnel` is later used to establish a connection between two contexts using an appropriate `ContextGate`.

A `ContextGate` is a side-specific implementation of a unidirectional communication channel between two contexts and represents a specific side that is currently being used.

Typically, a `ContextTunnel` and its corresponding interface are defined in a shared file that can be safely imported by both sides.
```typescript
export interface MyMainInterface {
  myFunction: (x: number) => number;
  mySyncFunction: (x: number) => number;
}

export const tunnel = new ElectronTunnel<MyMainInterface>('my_portal', {
  sync: ['mySyncFunction'],
});

export const myClassTunnel = new ElectronTunnel<MyClass>('my_class_portal');
```

Main process example:
```typescript
// |ElectronMainHandler| is an interface that transforms a user-defined interface, 
// i.e. injects an Electron-specific |Electron.IPCMainEvent| to each function and makes them async.
class MyMainHandler implements ElectronMainHandler<MyMainInterface> {
  public async myFunction(e: Electron.IPCMainEvent, x: number): Promise<number> {
    return x + 1;
  }

  public mySyncFunction(e: Electron.IPCMainEvent, x: number): number {
    return x + 2;
  }
}

const mainGate = new ElectronMainGate(ipcMain)

mainGate
  // Defines a handler for the tunnel, a handler is a special case of object which inherits from ContextHandler
  // so that function calls have platform-specific metadata about the message being sent.
  .handle(tunnel, new MyReceiverHandler())
  // Exposes an object to the tunnel, so that it can be accessed from the renderer process as-is.
  .expose(myClassTunnel, new MyClass());

// When tunnel is a ContextTunnel
tunnel.connect(mainGate, new ElectronRendererGateAddress(webContents)).myFunction(1);
// When tunnel is an ElectronTunnel
tunnel.connect(mainGate, webContents).myFunction(1);

```

Renderer process example:
```typescript
import { ipcRenderer } from 'electron';

// |ElectronRendererGate| is a side-specific implementation of a |ContextGate| for communication 
// from the renderer process to the main process in Electron.
const rendererGate = new ElectronRendererGate(ipcRenderer);

// |EstablishedTunnel| is an established unidirectional connection between both contexts,
// which is essentialy a proxy to the main process functions.
const tunnelToMain: EstablishedTunnel = tunnel.connect(rendererGate);
// Finally, a type-safe call to a function defined in the main process.
await tunnelToMain.myFunction(1);

const myClass = myClassTunnel.connect(rendererGate);

await myClass.add2Numbers(1, 2);
```

