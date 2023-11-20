---
title: Type-safe Electron IPCs
cover: /covers/type-safe-electron-ipcs.png
coverHeight: 1200
coverWidth: 630
date: 2023-11-18 20:00:00
---

Typescript is awesome. Electron, not so much I think. Tauri is generally considered the superior option, as it's faster, you can write code in a _supposedly_ more enjoyable language and it provides smaller package sizes. However, Electron's benefit is that your codebase can exist with only a single language used in it.

There is a problem whenever you try to incorporate a browser window as an application with any software, and that is protection of the user's device. With Chromium-based browsers and applications, you are sandboxing the rendering window from the rest of your OS, only allowing certain calls to send data to and retrieve data from the OS. Electron enables this with their own IPC implementation. IPC, or Inter-Process Communication, allows two separate processes to share data with one another. Electron needs to do this, as the part of the package that handles interaction with the computer (application menu, dialogs) is otherwise completely separated from the rendering part of the application (your website-turned-windows-app).

The electron IPC implementation is not too difficult to wrap your head around. You have three components to it: your `main` file (main process), your `preload` file (main process) and your html/js side (renderer process). To create an IPC channel between your main process and rendering process, you have to insert code in all three of these files. There are two ways to do so, but I will only explore the modern two-way approach of sending/receiving data to and from the main process.

First, the `main.js` file. This is what a cut-down version of your `main.js` may look like:

```js
import { app, BrowserWindow, ipcMain } from "electron";
import path from "node:path";

app.whenReady().then(() => {
  ipcMain.handle("doThing", async (event, args) => {
    await doThing(); /* Do something that requires access to your OS (filesystem, registry etc) here */
    return "done";
  });
  const mainWindow = new BrowserWindow({
    webPreferences: {
      preload: path.join(__dirname, "preload.js"),
    },
  });
});
```

Quite simple. You have what's essentially an event listener waiting for a call to the `doThing` channel, it does a thing, then it sends back the result. By making it async we don't have to fiddle around with callbacks quite the same way; bless modern Javascript features.

Now, `preload.js`. This is not actually a cut-down version, your preload can be quite small:

```js
import { contextBridge, ipcRenderer } from "electron";

contextBridge.exposeInMainWorld("electronAPI", {
  doThing: () => ipcRenderer.invoke("doThing"),
});
```

So what does this do? The function name `exposeInMainWorld` should serve as somewhat of an indication, but to be specific, this puts an `electronAPI` object onto the renderer process's `window` object. The `electronAPI` object will contain all properties of the object passed as the second parameter, allowing the renderer thread to call these specific methods (and _ONLY_ these methods, for safety).

Usage of these methods becomes clear if we look at any file in the actual website that wants to use it, so let's make a very simple `index.html`:

```html
<!DOCTYPE html>
<html>
  <head>
    <title>Site</title>
  </head>
  <body>
    <p id="stuff-in-here">Stuff!</p>
    <button type="button" id="btn">Click me!</button>
    <script defer>
      const p = document.getElementById("stuff-in-here");
      const btn = document.getElementById("btn");

      btn.addEventListener("click", async () => {
        const returnedValue = await window.electronAPI.doThing();
        p.innerText = returnedValue;
      });
    </script>
  </body>
</html>
```

A simple setup; we have a button, and when you click it, `doThing()` is called, and the value you get back from it is put in the only paragraph element. These are the three elements there are to Electron IPC.

There's one big problem to this approach however and that is type safety. I'm creating a Nuxt 3 application, which uses Typescript for everything. I want to make sure that the types I send and retrieve are of a certain value type, and I want to be able to use that type safety in my frontend code as well as in my Electron code. How do I go about this?

I have not gotten too far into this, but I have found a semi-solution that works for me. I have added a few pieces of code, some of it specifically available to Electron:

```ts
// in Electron's main.ts:
function addIpcMainHandler(
  channel: AllowedChannels,
  func: ElectronApiFunction
) {
  ipcMain.handle(channel, func);
}

// Then, in the bootstrapping process:
addIpcMainHandler("doThing", async () => {
  const result = await doSomethingWithLocalFs();
  return result;
});

// in Electron's preload.ts:
function addIpcRendererHandler(api: any, channel: AllowedChannels) {
  api[channel] = (...args: any[]) => ipcRenderer.invoke(channel, args);
}

const api = {};
addIpcRendererHandler(api, "doThing");

contextBridge.exposeInMainWorld("ElectronAPI", api);
```

This doesn't look _too_ type-safe yet, does it? Here is where my `shared/` folder comes in:

```ts
// shared/ipc/channels.ts:
export type AllowedChannels = "doThing" | "doOtherThing";

export interface ElectronApiFunction {
  (...args: any[]): Promise<any>;
}

declare global {
  interface Window {
    ElectronAPI: {
      [key in AllowedChannels]: ElectronApiFunction;
    };
  }
}
```

If you don't know a lot about Typescript, this may be a bit confusing, so let's break down what's happening here.

```ts
export interface ElectronApiFunction {
  (...args: any[]): Promise<any>;
}
```

This is the signature that Electron IPC methods follow if you're using asynchronous methods for them; anything can be a parameter, there can be any parameters, and anything can be returned from them (but in my case, it has to be a Promise). This is because I am only using Electron IPCs with two-way binding. If I used one-way binding, they'd all return `void` since they're just calls to the main process to do something. This is mostly for niceness in `main.ts` and in any of my rendered files. It doesn't actually _do_ much in `main.ts` though, since `ipcMain.handle` already provides the typings for its own method.

```ts
export type AllowedChannels = "doThing" | "doOtherThing";
```

This provides some really nice type-checking on my IPC channels though. Let's look back at my `preload.ts` for a second:

```ts
// no error
addIpcRendererHandler(api, "doThing");

// Argument of type '"foo"' is not assignable to parameter of type 'AllowedChannels'.
//                         vvvvv
addIpcRendererHandler(api, "foo");
```

Great. I don't have to worry about misspelling this string throughout my application anymore. If I do, my linter will just spit out an error!

```ts
declare global {
  interface Window {
    ElectronAPI: {
      [key in AllowedChannels]: ElectronAPIFunction;
    };
  }
}
```

This is _really_ nice for your renderer process code. This piece of code essentially tells your linter "Hey, actually the window object _will_ have an `ElectronAPI` property object attached to it, and this object will contain properties according to all strings in the `AllowedChannels` type, and these are all functions". This means that in my renderer thread I can simply call `window.ElectronAPI.doThing()` and my linter will be okay with it despite me never actually having directly added anything to the `window` object. If I _don't_ add the above piece of code to my codebase, my linter will say "Property `ElectronAPI` does not exist on type `Window & typeof globalThis`". I can still call these functions though. The reason they still work is because we did put those functions on the window object, in `preload.ts`. Our editor just doesn't know about that.

So, what does this look like when I actually use these typed calls? Anywhere in my renderer process, I can simply call

```ts
const result = await window.ElectronAPI.doThing();
```

...and everything just works now.

One weird thing that I did run into is my default Electron configuration not working properly for setting up ipc channels. My Electron BrowserWindow object actually had `contextIsolation` set to `false` by default, which meant I could not use `contextBridge.exposeInMainWorld` in my application. I could also not directly set the `window` properties in `preload.ts`, which supposedly was the reason `contextIsolation` existed as an option in the first place. I had to re-enable it in the options to be able to expose the functions to the `window` object, but then it worked fine.

For now, that's as far as I have gotten with type safety in Electron. What I want to work on more is being able to define the correct signature for the functions I'm calling and have those return types show up in my editor as well (if `window.ElectronAPI.doThing()` is supposed to return a string, then I want vscode to give me an error if I don't treat the return value as one). That's something left for next time though. Now I have to deep dive into CSS hell to figure out how I am going to style this application I'm working on.
