We're getting ready to ship RedwoodJS v5.
The priority this major is React 18.
Things were going smoothly till it came to serving prerendered pages; when we navigate directly to a prerendered page, we're seeing double rendering, as in there's two instances of the page:

<img width="1522" alt="image" src="https://user-images.githubusercontent.com/32992335/224161860-70aafa2d-70be-43b5-b4c4-a54d056c86ff.png">

(Here's the corresponding Routes file; note that the AboutPage has the `prerender` prop)

https://github.com/jtoar/redwood-react-hydration-debugging/blob/b75f47c7bf46685bd85e0f983ba021ba7b75bf7a/web/src/Routes.tsx#L12-L20

Instead of hydrating the HTML from the server, React seems to be ignoring it and mounting a new instance of the App after it. Why?

While this double-rendering behavior is new, I think the key insight I've made so far is that the root cause of it—a hydration error—isn't.
As in, we've always had a hydration error, it's just that React 17 gave us a break by patching the DOM and React 18 has no breaks to give.
(We didn't notice it before because when we build and serve, we're using the production build of React, which strips out warnings and errors.)
This issue in the React repo provides more details, but basically the React team viewed hydration errors as legit bugs to be fixed, and don't want to patch the DOM anymore for security reasons: https://github.com/facebook/react/issues/23381#issuecomment-1079494600:

> In general, rendering different things as a strategy was never supported. Before React 16 it didn't work at all. After React 16, it worked with an asterisk but it was always considered a bug in user code that should be fixed. The documentation said that as well.

Ok, but why is there a hydration error in the first place? That has to do with the Router and how we build the web side.
We code split aggressively.
We take full advantage of webpack code splitting on dynamic import statements (`import()`).
Every page is it's own bundle.
This means that when the browser receives the main bundle (which contains the App component), there are no pages in it.
They're in separate bundles, and are acquired asynchronously.
This is normally what we want cause most users don't go to most pages.
But even prerendered pages have to be imported... which means the client's first render is `null`.
And when server-rendered HTML doesn't match up with the first client-side render, you have a hydration error:

<img width="1522" alt="image" src="https://user-images.githubusercontent.com/32992335/224169411-054b5a8c-a49e-4047-ba6a-2e9786510700.png">

### Reproducing the hydration error in RedwoodJS latest (React 17)

> **Note** Here's an annotated Replay
>
> https://app.replay.io/recording/react-17--250e7db5-48de-4530-a61c-5694ea0c94ab

- clone this repo
- checkout the react-17 branch (`git checkout react-17`)
- `yarn install`
- `git checkout node_modules/@redwoodjs/core/config/webpack.common.js`
  - this adds back the change I made to Redwood's webpack config so that we're using the development build of React (the error doesn't show up in the production build of React)
- `yarn rw build`
- `yarn rw serve`
- go to http://localhost:8910/about
- open the console; you should see...

```
Warning: Did not expect server HTML to contain the text node "
    " in <div>.
```

<img width="1522" alt="image" src="https://user-images.githubusercontent.com/32992335/224127973-58f3b6d9-b676-4a94-9195-bd3f667218d9.png">

### Making the hydration error in RedwoodJS canary (React 18) more like one in RedwoodJS latest

Is there any way we can get React to help us out like it did in 17?
It doesn't seem like we can get it to do what it did before one to one, but it does seem like we can get it to do something, via Suspense:

> Similarly, when an error is thrown during hydration on the client, React will discard the server-rendered HTML and revert to a clean client render.

(Taken from this RFC discussing hydration errors: https://github.com/reactjs/rfcs/blob/ba9bd5744cb922184ec9390515910cd104a30c6e/text/0215-server-errors-in-react-18.md#summary)

- clone this repo
- checkout the react-18 branch (`git checkout react-18`)
- `yarn install`
- `git checkout node_modules/@redwoodjs/core/config/webpack.common.js`
  - this adds back the change I made to Redwood's webpack config so that we're using the development build of React (the error is minified in the production build of React)
- `git checkout node_modules/@redwoodjs/router/dist/active-route-loader.js`
  - this wraps the ActivePageLoader in Suspense which makes React treat hydration errors more like how it did in v17
- `yarn rw build`
- `yarn rw serve`
- go to http://localhost:8910/about
- open the console; you should see...

<img width="1522" alt="image" src="https://user-images.githubusercontent.com/32992335/224136421-c8abbd44-4aba-4dcf-8eb5-0f806b5fef76.png">

Like how things are now, these errors won't show up in a production build.
But while there's no double-rendering, the behavior here is slightly different in a very undesirable way.
There's a flash. (It's hard to see, so I throttled the network):

https://user-images.githubusercontent.com/32992335/224136797-559231c0-8947-435b-9fae-41ecd411dbed.mov

### Towards a real solution: fixing the hydration errors

Really, we have to fix the hydration errors.
That means the first client render needs to be the same as the server-rendered HTML.
And that means the router needs to be rewritten a bit, so that the initial page specified here is the prerendered one:

https://github.com/redwoodjs/redwood/blob/8a437218f32afae72e7740ea3d12ee339f8d222b/packages/router/src/active-route-loader.tsx#L49-L51

But to do that, the prerendered page needs to be included in the main bundle so that we don't have to async-await it.

This branch isn't a full-fledged solution, but at least demonstrates it's possible:

- clone this repo
- checkout the react-17-solution branch (`git checkout react-17-solution`)
- `yarn install`
- `git checkout node_modules/@redwoodjs/core/config/webpack.common.js`
  - this adds back the change I made to Redwood's webpack config so that we're using the development build of React (the error is minified in the production build of React)
- `git checkout node_modules/@redwoodjs/router/dist/active-route-loader.js`
  - this naively changes the ActiveRouteLoader so that it synchronously imports pages included in the bundle
- `git checkout node_modules/@redwoodjs/router/dist/util.js`
  - right now, normalizeSpec always makes the loader async; this makes it synchronous
- `yarn rw build`
- `yarn rw serve`
- go to http://localhost:8910/about
- open the console; you should see... (nothing!)

<img width="1522" alt="image" src="https://user-images.githubusercontent.com/32992335/224179499-6d4e9112-7a1b-4c31-bdeb-b4f15c7205a6.png">

Now the server-rendered HTML matches up with the first client-side render:

<img width="1522" alt="image" src="https://user-images.githubusercontent.com/32992335/224180176-2fdf1221-7eb1-46f1-9e50-c10d452af594.png">
