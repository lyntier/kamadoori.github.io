---
title: Changing the Referer in Electron
cover: /covers/changing-the-referer-in-electron.png
coverHeight: 210
coverWidth: 400
date: 2023-12-04 23:00:00
---

Sometimes you want to change things you shouldn't. Sometimes you are actively prevented from changing those things by the software you're using. One of those things is the `Referer` header.

The `Referer` header, which is not a word and supposed to be spelled 'referrer' (check RFC 1945 section 10.13 [here](https://datatracker.ietf.org/doc/html/rfc1945#section-10.13)), tells the webpage that is receiving a request from which hostname that request originated. As per the words of this RFC, "This allows a server to generate lists of back-links to resources for interest, logging, optimized caching, etc.". I am not interested in those things per se; what I am more interested in is sites using the `Referer` header to allow/block traffic. When I build a scraper for example, I am performing a request that comes directly from my client, and this will be reflected in the header as being sent from `https://localhost:{port}/`. When this happens, the server returns a 403 and I am sad.

So what are your options? First I tried setting the header in the request:

```ts
const response = fetch(url, {
  headers: [["Referer", someValidReferer]],
}); // 403 Forbidden
```

Still forbidden, because the header is still actually set to `https://localhost:{port}/`..

I could list all the things I tried, such as setting the `window` history but really Chrome just doesn't _allow_ you to set the referrer. You can use an extension, but that's not really feasible in Electron. So, what _is_ feasible?

The problem you're running into is that Electron's browser is not allowing you to change the referrer, but what if you didn't make your network call from the browser? Simply let Electron's main thread handle the network call. If you'd like to read more about IPCs, I've made two posts before about it [here](https://kamadoori.github.io/2023/11/18/type-safe-electron-ipcs/) and [here](/2023/11/24/even-better-type-safe-electron-ipcs/).

I add a new channel:

```ts
export interface ElectronAPI {
  // ...
  fetchWithReferer(
    event: IpcMainInvokeEvent,
    url: string,
    referer: string
  ): Promise<ArrayBufferLike | undefined>;
}

// ...

export const api: ElectronMainAPI = {
  // ...
  fetchWithReferer: async (
    _event,
    args: any[]
  ): Promise<ArrayBufferLike | undefined> => {
    const url = args[0];
    const referer = args[1];

    // `net` is an Electron module for performing network actions in the NodeJS landscape
    const response = await net.fetch(url, {
      headers: [["Referer", referer]],
    });

    if (!reponse.ok) {
      return undefined;
    }

    const buffer = await response.arrayBuffer();

    return buffer;
  },
};
```

And call it:

```vue
<script setup>
const data = await window.ElectronAPI.fetchWithReferer('https://data-site.fake/resource.png', 'https://partner-site.fake/')
// 200 !
<script>
```

That works for my use-case. If you find a way in which you can modify the Electron instance to accept custom referrer headers that would very likely be better, but this works just fine and doesn't require you to do anything you wouldn't do with Electron in general anyway.
