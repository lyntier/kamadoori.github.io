---
title: Even better type-safe Electron IPCs
cover: /covers/even-better-type-safe-electron-ipcs.png
coverHeight: 210
coverWidth: 400
date: 2023-11-24 20:00:00
---

Woo! Type-safety has been achieved! And I managed to also completely generalize it. If I want to add a method to the API, I simply need to add the signature to the interface and the implementation to an object. Everything else remains as it has been!

The last time I looked at this, I closed the ordeal by saying "You cannot use any functions with names that aren't in this type, hooray" and basically left it at that. Thing is, Typescript's type system is SO much more flexible than that, and there's so much more to discover. So what changed since last time?

## Type mapping

I look a look at type mapping. Let's say I write a function signature:

```ts
type doThing = (foo: Foo, bar: Bar) => Baz;
```

What if I want to have a version of this signature, but _not_ include the `foo` parameter? The simple solution would just be to create another type:

```ts
type doAnotherThing = (bar: Bar) => Baz;
```

This works.. but if I have multiple functions that I want to treat this way, obviously that's not really a way to go about it. This is where type mapping comes handy. With type mapping, you can create types that are based on other types:

```ts
type HasPropsWithTypeString = {
  [key: string]: string;
  foo: string;
};

// We can map this to give back other data types instead:
type HasPropsWithTypeNumber = {
  [someKey in keyof HasPropsWithTypeString]: number;
};
```

What this code does is take all keys from `HasPropsWithTypeString` and say that an object with this new type will return numbers when these keys are addressed. Typescript now treats objects with this type as having a prop `foo` that's a number. Even better when constructing an object of this type, having a `foo` number prop becomes a **requirement**. If you do not include it, Typescript will let you know: `Property 'foo' is missing in type '{}' but required in type 'givesN'.`

So what if we apply that to our own API?

```ts
export interface ElectronAPI {
  doThing(event: IpcMainInvokeEvent, someArgument: string): boolean;
  doAnotherThing(event: IpcMainInvokeEvent, anotherArgument: number);
  doThirdThing(event: IpcMainInvokeEvent);
}
```

Relatively standard API example. Now, I've got this magic prop eraser type, it'll remove the first prop from these methods:

```ts
type RemoveEvent<Fun> = Fun extends (
  event: IpcMainInvokeEvent,
  ...args: infer Param
) => infer Result
  ? (...args: Param) => Result
  : never;
```

This is a bit more of an advanced mapping, so let's go over it.

`RemoveEvent<Fun>` => Generic types. This generic type is named `Fun` here, after Function. Not after enjoyment, we're still programmers, so we don't get to have fun.

`Fun extends (/* snip */) =>` => Conditional typing. This allows you to map a type depending on what's incoming. In this case, if the incoming type is a type that starts with an `event` parameter of type `IpcMainInvokeEvent`, then it maps to the type after the `?`, otherwise it maps to the type after the `:`. This is exactly the same as a ternary operator.

`infer Param`, `infer Result` => Tells Typescript to infer the type that these two things are based on the context. So if a function with a signature of `(event: IpcMainInvokeEvent, foo: number): string` came in, `Param` will be inferred to be `(foo: string)`, and `Result` will be inferred to be `string`.

`never` => declare that this situation should not occur. In other words, if our function does not match the pattern of the `extends` type, we are intentionally making this type useless.

So how do we use this now? We can simply pass in functions to the type parameter:

```ts
// Note: Can't access these properties with dot notation. However, there is type safety on the linter for bracket notation.
type doThingMapped = RemoveEvent<ElectronAPI["doThing"]>;
type doAnotherThingMapped = RemoveEvent<ElectronAPI["doAnotherThing"]>;
```

If a function now uses this as its type, it'll show the exact same parameters and return type as the original, _without_ the event parameter. Cool, huh? Though, this could be applied a little bit more generic instead...

```ts
export type ElectronRendererAPI = {
  [key in keyof ElectronAPI]: RemoveEvent<ElectronAPI[key]>;
};
```

Beautiful. This will essentially map all existing functions in `ElectronAPI` to the `RemoveEvent` type, passing the current function in as parameter. It turns out that `key` is actually defined (and usable) as variable here, so using it in this manner means that you simply select the property!

Now I can do the same thing for the Main thread of the application. When Electron ipc methods are called, they put all arguments in a rest parameter, which is an array of type `any`. You have to destructure this yourself again, but you do want the right type hints to exist in the main-thread part of the code still. If you don't type `args` as `any[]`, you can't access it as an array with type hinting. So, let's do the same trick, but instead of mapping it so it removes the `event` parameter, let's map it so it instead removes every other parameter and adds an `...args` parameter:

```ts
type RemoveParams<Fun> = Fun extends (
  event: IpcMainInvokeEvent,
  ...args: infer Param
) => infer Result
  ? (event: IpcMainInvokeEvent, ...args: any[]) => Result
  : never;

export type ElectronMainAPI = {
  [key in keyof ElectronAPI]: RemoveParams<ElectronAPI[key]>;
};
```

And that's what my `channels.ts` file now looks like! To define the API on the main thread side:

```ts
const api: ElectronMainAPI = {
  // Note: Automatic type hinting on the parameters and return type!
  doThing: async (_event, args) => {
    const arg1 = args[0];
    const result = await doSomethingAsync(arg1);
    return result.success;
  },
  /* Snip: define the other methods here too */
};
```

In my `main.ts`, to register handlers:

```ts
Object.keys(api).forEach((key) => {
  ipcMain.handle(key, api[key as keyof ElectronAPI]);
});
```

Then in my `preload.ts`, to expose the methods in the renderer thread:

```ts
// This is simply an object we load up with `invoke` calls for all the ipc channels by reading the original `api` object and applying the functions from that to this.
const exposed = {} as ElectronRendererAPI;

Object.keys(api).forEach((key) => {
  exposed[key as keyof ElectronRendererAPI] = (...args: any[]) =>
    ipcRenderer.invoke(key.toString(), args);
});

contextBridge.exposeInMainWorld("ElectronAPI", exposed);
```

And that's it. Completely generalized and (almost) completely typed Electron ipc API. The only missing link is easier typing for called ipc methods in the main thread, where now everything is stuffed inside of an `args` argument, but I am unsure on how to fix that. Maybe instead pass in an `options` object, and always destructure the first argument in the `args` object to match a predefined type for every function in `ElectronAPI`? It might be possible, but I am going to stop here. Parsing arguments is a very minor holdup and not very sensitive to _syntax_ errors, instead being sensitive to _symantic_ errors. I'm happy with what I achieved here.
