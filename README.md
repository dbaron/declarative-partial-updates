# Declarative partial updates

HTML primitives for updating parts of a document without a full page refresh
Addresses https://github.com/whatwg/html/issues/2142 and https://github.com/whatwg/html/issues/2791 in a unified manner.

## Status of this document
This is an living explainer. It is continuously updated as we receive more feedback and update the design and the prototypes.

## Overview
### Full page refresh
Since the early days of the web, developers were seeking ways to work around the "document refresh" experience - having to wait when clicking a link until the next page is brought up.
Solutions to this problem have proliferated in the shape of JS-based "Single page applications" (SPAs), and frameworks that make it easier to build such applications.

Some newer web platform features such as prerendering improve this experience, however there are many situations where unloading a document and loading a new one is still inferior to keeping the document alive, including all its chat widgets, 3rd party scripts, and so on.

### Fragments/views
Web apps that require this complexity are usually richer than an HTML document that shows a single piece of content. Modern web apps are often composed of multiple "fragments" of documents, or "views", presented at the same time and layed out in a flexible way.
However, the primitives that the web platform offers for this fragmentation, i.e. iframes, are not flexible in this way. One cannot use a single stylesheet to style multiple iframes, or select all the elements given a CSS selector in multiple documents at the same time,
and the iframe's contents cannot escape its bounds, to name a few constraints.

The web platform community still uses iframes extensively, however not so much for app fragments. Instead, web frameworks are used to manage that complexity.

### Composable, same-document fragments
In modern web frameworks, composable fragments of an application have the following properties:
* It can be updated independently, without a full-page refresh.
* Its content is loaded on demand, based on the document's URL and other state.
* It shows fallback UI for loading or error conditions
* It transitions between states smoothly

### Developer Experience
All of these properties directly affect the quality of the user experience, in terms of performance and smoothness.
To achieve this kind of user experience today, developers have to rely on JavaScript libraries or frameworks that manage in-document “components”, “pages”, “layout”, loading states etc.

## Design principles
Our approach to partial document updates acknowledges the current dominance of full-stack frameworks in this area. The underlying design principles of this proposal are twofold:
1. *Empowering Native UX*: We aim to enable a powerful, declarative user experience for same-document navigations, mirroring the seamless platform integration seen in cross-document navigations. This functionality is intended to be usable directly, independent of larger web frameworks.
2. *Extensibility and Compatibility*: Concurrently, the proposal should be enhanced to include ample flexibility and low-level extensibility points. This would ensure that current framework ecosystems and custom application logic can integrate with and leverage specific aspects of the solution, avoiding the need for a complete architectural overhaul, or choosing this as an all-or-nothing framework/solution of its own.

As we go along with more specific design, we should make sure that we're honest with ourselves for meeting the bar of these design principles.

## Prior art
### IFrames
IFrames are an existing web-platform mechanism for reusing content. However, IFrames are heavy-handed in terms of UI. They cannot escape their box, they are styled and scripted separately, and their layout is encapsulated from the rest of the page.

### Native mobile
Native apps come with this ability built in, see for example view controllers in iOS, which provides flexible ways to manage interactions between different parts of the app, and Android Fragments. A fragment or a view is a reusable piece of an application, that has its own lifecycle and can be composed with other fragments to produce an app.

### Web Frameworks
Web frameworks, like NextJS, Remix, Nuxt etc, handle this type of problem in nuanced ways, but with a common theme of somehow mapping a URL route to a set of components that get displayed when that route is active, often something like a “page”, and with a more stable “layout” that defines the whole document and leaves spaces for those fragments.

For example, in NextJS:

```jsx
// app/layout.js
export default function RootLayout({ children }) {
  return (
    <html lang="en">
      <body>
        <header>My Next.js App (App Router)</header>
        <nav><!-- links to all pages --></nav>
        <main>{children}</main> {/* children represents the current page */}
      </body>
    </html>
  );
}

// app/blog/[slug]/page.js
// This component would be rendered into the {children} slot above.

export default function BlogPostPage({ params }) {
  const { slug } = params; // 'my-first-post' or 'another-article'

  return (
    <div>
      <h3>Blog Post: {slug} (App Router)</h3>
      <p>Content for {slug} goes here.</p>
    </div>
  );
```
In Remix, the concepts have different names but a similar structure:
```jsx
// app/root.jsx
import { Links, Meta, Outlet, Scripts, ScrollRestoration } from '@remix-run/react';

export default function App() {
  return (
    <html lang="en">
      <head>
        <meta charSet="utf-8" />
        <meta name="viewport" content="width=device-width, initial-scale=1" />
        <Meta />
        <Links />
      </head>
      <body>
        <header>My Remix App</header>
        <nav>
  </nav>
        <Outlet /> {/* This is where nested routes will render */}
        <ScrollRestoration />
        <Scripts />
      </body>
    </html>
  );
}
```

And also in Astro:

```html
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>{title}</title>
  </head>
  <body>
    <header>
      <h1>{title}</h1>
    </header>
    <nav>
      <a href="/">Home</a>
      <a href="/about">About</a>
      <a href="/blog">Blog</a>
    </nav>
    <main>
      <slot /> <!-- This is where the page content will be injected -->
    </main>
    <footer>
      <p>&copy; 2025 Astro Example</p>
    </footer>
  </body>
</html>
```

### HotWire / TurboLinks

See https://hotwired.dev/.

The HotWire approach has a lot of similarities with this proposal, and is a source of inspiration.
In that approach, the navigation response stays the mostly same, and the client side is responsible for replacing parts of the document, mainly the `<body>` modulo "islands".

The approach here is a bit more generic, where the developer decides which parts get replaced and which stay.

## This Proposal
The goal of this proposal is to see whether the platform can help with this kind of experience, by introducing composable fragments (views, because the word fragment is used elsewhere...) and declarative partial document updates as a platform (HTML) primitive.
It consists of three parts. The first two stand on their own, and the last one uses the first two:
* Patching (out-of-order streaming)
* Route matching
* Declarative same-document navigations

### Part 1: Patching (out-of-order streaming)
Currently, the DOM tree is updated in the order in which its HTML content is received from the network. This is a good architecture for an article to be read linearly, however many modern web apps are not built in that way.
Often in modern web apps, the layout of the document is present, with parts of it loaded simultaneously or in some asynchronous order, with some loading indicators or a skeleton UI to indicate this progression. 

The proposal here is to extend the `<template>` element with a `patchfor` (or `for`) attribute that would repurpose it to be a patch that updates the document or shadow root it is inserted into, and a `patchsrc` attribute that would allow fetching the patch content from an external URL.
`<template patchfor>` essentially becomes an HTML-only encoding for a DOM update.

```html
<!-- initial response -->
<section id="photo-gallery">
Loading...
</section>

<!-- later during the loading process, e.g. when the gallery content is ready -->
<template patchfor="photo-gallery">
  ... the actual gallery...
</template>

<!-- When we want to update a document or shadow root with new out-of-order streaming content -->
<script>
async function update_doc() {
  // The response's type would be "text/html".
  const response = await fetch("/new-data");

  // This would stream the response to the body, applying only patches and discarding the rest.
  await document.patch(response);

  // We can also stream patches into a shadow root.
  const element_internal_response = await fetch("/element-data");
  await some_element.shadowRoot.patch(element_internal_response);

  const gallery_container = document.getElementById("photo-gallery");
  gallery_container.addEventListener("patch", () => { /* called when a patch starts */ });

  const {currentPatch} = gallery_container;
  // This will be null if there is no ongoing patch, similar to `navigation.transition`
  if (currentPatch) {
    if (should_abort()) {
       currentPatch.signal.abort();
    }
    currentPatch.finished
        .then(() => { /* patch complete */ })
        .catch(() => { /* error while patching, e.g. request closed */ });
  }
}
</script>

<!-- The patch content can be fetched from an external URL. -->
<!-- The inline content will be shown as fallback if fetching failed -->
<template patchfor="photo-gallery" patchsrc="/gallery-content.php">
  Failed to load
</template>

<!-- this can also be done with script -->
<script>
photoGallery.patch(await fetch("/gallery-content.php"));
</script>
```

#### Details of patching

1. The `patchfor` attribute is an IDREF, and behaves like `<label for>`, pointing to an ID. See https://github.com/whatwg/html/issues/10143 for efforts to make that DX better.
1. When a `<template patchfor>` is discovered, its contents are parsed directly into its target position, without adding the actual `<template>` element to the DOM.
   When the corresponding `</template>` end tag is discovered, parsing resumes as normal.
1. The `IDREF` is resolved using the tree scope where the template was discovered (which can be a shadow root).
1. A patch whose target is not found is parsed as a normal `<template>` element and remains in the DOM. There is no further change detection to try to match it, and the author is responsible for that kind of change detection if they so choose.
   This is equivalent to trying to setting the `innerHTML` of a DOM element that doesn't exist. A `patcherror` event is fired.
1. `node.patchSelf()` streams the content of an HTML stream directly into an element by creating a `WritableStream`. (Alternative names: `patch` or `patchReplace`).
1. `node.patchAll()` creates a `WritableStream`, which in turns parses the HTML contents streamed to it, and extracts `<template patchfor>` elements from it, streaming it to the correct targets. ID lookup is scoped to the target node. (Alternative name: `patchInterleaved` or `patchSplice`.
2. `node.patchBetween`, `node.patchAfter` and `node.patchBefore` patch a range of children of an element, by first removing what's between the given nodes and then inserting the result of the stream.
3. All the `node.patch*` functions return a `WritableStream` that can accept bytes or strings. Other types are stringified. Strings are parsed directly, bytes are decoded using the document's character set.
1. `element.currentPatch` returns (null or) an object that reflects the current status of a patch, and allows aborting it or waiting for it to finish.
1. The `patch` event is fired when an element is being patched, with the same timing as mutation observer callbacks and "slotchange" events. It allows intercepting a patch and injecting a `TransformStream` into it.
1. The `:patching` pseudo-class is activated on the element during patch.
1. The `patchsrc` attribute allows fetching the content of the patch from a different URL, using a cors-anonymous fetch (allowing `crossorigin`/`referrerpolicy` attributes and all that jazz).
1. While the content is being fetched, the element receives a `:patch-loading` pseudo-class.
1. The template's inline content is used for the patch as fallback while it is being fetch, and the element receives a `:loading-error` pseudo-class if fetching failed.

##### Even more details

1. From an HTML parser internals point of view, a new fragment parser is created for each patch with the patch target as the context node, and the output of this parsing is inserted directly to that target.
1. If trusted types are present in the document, parsed contents are buffered (not streaming) and pass through the trusted types system before being inserted to the DOM. Using this without trusted types can be done by capturing the `patch` event and intercepting it.
2. `patchsrc` requires the same CSP privileges as a script.

### Part 2: Route matching

To effectively match between parts of the document and the state of the app, there is commonly a relationship between the document's URL and active/visible components of the app.
This is commonly referred to in frameworks as "routes".

The proposal here is to make routes a first-class citizen in HTML, and using that (at first) for two things:
1. Wrapping certain parts of the documents that belong to a certain route with a `<view>` element that describes that relationship.
2. CSS at-rule conditions that get activated by route-matching, similar to media-queries:

```html
<!-- Similar shape to importmap -->
<script type=routemap>
  {
    "movies": { "pattern": { "pathname": "/movies/:movie_id" } },
    "about":  { "pattern": "/about" },
    "home":   { "pattern": ["/home", "/"] }
  }
</script>

<style>
@route home {
  nav {
    display: none;
  }
}
</style>

<body>
   <view match=home>
   </view>
   <view match=about>
   </view>
   <view match=movies>
   </view>
</body>
```

#### Details of `<view match=...>` and `<script type=routemap>`

1. Every route can be one or more `URLPattern` (more precisely, the parameters to the `URLPattern` constructor)
1. A `view` can be described by using CSS. It is merely an element that has `display: none` (or `content-visibility: hidden`, details TBD) when the route doesn't match.
   The choice to make it an element makes it clear semantically that this part of the document is an app fragment.
1. Views can be extended in the future to support a per-view URL ("HTML includes"), and have additional CSS selectors for links that target a particular route.
1. Multiple views that match the same route can be present at the same time.
1. Similarly, multiple routes can match the same URL at the same time. This is by design.
1. The `match` attribute can be bikeshed... perhaps `matchroute`.
1. A `navigator.routes` [maplike](https://webidl.spec.whatwg.org/#idl-maplike) object reflects the routemap, so that `navigator.routes.get('movies')` returns an object with a `pattern` property (a `URLPattern`) and a `matches` boolean when the route matches. A "change" event is fired on the `navigator.routes` object when the matching route changes.
1. Alternatively, we could start with more of a CSS-centric approach, with `<script type=routemap>` but without a new element. See https://github.com/WICG/declarative-partial-updates/issues/14.

### Part 3: Declarative same-document navigation

Once we gain the ability to stream content into the document, and also declaratively match certain parts of the document to a URL pattern,
many types of same-document navigations would not require complicated JS in order to run, and this type of operation can be done declaratively:

```html

<script type=routemap>
  {
    "article": { "pattern": "/articles/:article_id" },
    "mode": "same-document"
  }
</script>

<view match="article" id=main-article>
Loading...
</view>

<!-- the user clicks a link to `/articles/foo123` -->
<!-- navigation is intercepted -->
<!-- new content includes: -->
<template for="main-article">
  <article>
    Content of article foo123
  </article>
</template>

<style>
/* The declarative view transition would work for the declarative same-document navigation! */
@view-transition {
  navigation: auto
}
</style>
```

In an app that contains matching views, the developer does not have to worry about specifics of the navigation API, or the mechanics of streaming/piping.
All they have to do is list their routes, declare which parts of their document match their URL routes, and then stream the appropriate new content wrapped in `<template for=...>` snippets.

#### Details of declarative interception

1. When a route has a `mode: 'same-document'` (or `intercept: true` or some such) clause, navigations to matching destinations would be intercepted. The navigation request would still take place, but only `<template for>` elements from the response would be spliced-streamed into the document.
1. Declarative view transitions work out of the box.
1. Some non-navigation UI, like auto-closing popovers, can react automatically to this kind of navigation, e.g. by closing in this case. UI is tuned in a way that a "navigation" feels like a cross-document navigation (as in, resets state) in some cases but keeps things alive in other cases. 
1. Views matching the old/new route would receive an "unloading" and "loading" state, with appropriate JS events and CSS selectors.
1. While streaming content, target elements would get a pseudo-class (:partial?) activated. This pseudo-class can be used in ordinary document content streaming as well, to avoid the visual effect of streaming.
1. A partial response can include either a full document or just the modified templates, the UA should be able to work with both as valid HTML. 
1. A request for an intercepted partial update contains header information about the views that are about to be updated, and about the fact that it's a partial update. This allows the response to include only the updated part (though we have to be careful about content-negotation trade-offs).
2. Declarative interceptions would come with a JS API that allows hooking into them at certain points, to avoid having to adopt the whole solution with all of its tradeoffs. (Details of this JS API TBD).

## Potential future enhancements
1. Allowing the developer to fine-tune the relationship between routes, e.g. allow some route transition to replace the current history entry rather than push a new one.
1. A reverse relationship: a `<view>` activates a URL when it becomes visible (e.g. scrolled to). This can allow creating gesture-based user interfaces without manually managing them via JS.
1. Using CSP to allow/disallow declarative navigation interception
1. Improve loading of updates via speculation rules / prefetching, and extending this further by publishing a “site map” or some such that specifies relationship between routes.

## Open issues
1. This architecture might lead to sending different responses for the same URL based on headers, e.g. a partial document response. This doesn’t work well cross-browser and cross-CDN. Perhaps the partial responses need a separate URL, with the `src` attribute, and the intercepted navigation responses should be expected to be full?
1. How does this interact with declarative shadow DOM? Can parts of a declarative shadow DOM be streamed in this way?
1. Interaction with templating proposals such as DOM Parts. Should we allow some sort of a "merge" semantic to `<template for>`?
1. Does `view` need to be a new element? Perhaps a new attribute work better, like `popover` or `openable`?
1. ID refs are notoriously footgunny and it's easy to get them wrong. However, they are the current referencing enabler of the web platform. Perhaps there is some way to go about refs that improves things for other features as well?


## Considered Alternatives & Tradeoffs

### HTML "include"

The `<template patchfor>`  proposal puts HTML streaming at the forefront rather than HTML includes, however it allows for HTML includes as well by streaming from an external URL using the `patchsrc` attribute.
This addresses some of the issues found in previous HTML inclusion proposal, as it clarifies the semantics of the included content - it's a streaming "patch" with an inline fallback, and works like other patches with an additional external fetch. 

### IFrames
Too many gotchas and constraints, in term of layout/UI and ergonomics of crossing documents. This approach tries to work with how modern web development apps work, where the fragments are part of the same document.

### Element-to-element
The HTMX approach, where a form or link can target an element for its output, is a really nice concept.
In some ways, it overlaps with other concepts being developed in the web platform, such as invokers.
In any case, the HTMX approach feels more like a development-time concept that the web platform does not need to be opinionated about.
By allowing partial updates together with a link between the document's URL (which is a UX concept, as that URL can be shared, QRed, linked etc) and the composable fragment.

### Enhancing `<slot>`
This is possible, and has similarities with declarative shadow DOM, in terms of out-of-order content. It might be more complex/confusing than its worth as a `<view>`, as in the right context a `<slot>` and a `<view>` behave massively differently.

### Enhancing MPA architecture capabilities
This architecture primarily focuses on extending the ergonomics and capabilities of same-page applications. It is possible to explore other avenues that work in a more MPA/cross-document manner.
However, it is unclear what that’s going to look like, and would potentially lead to an IFrame-based architecture.

### Lower-level JS API
Another way to go about this is to introduce JS primitives that introduce similar functionality but are less opinionated in terms of being declarative.
At least for out-of-order streaming, this can definitely be a useful primitive alongside the more server-driven HTML primitive.

### Using `multipart/form-data` instead of `<template patchfor>`
Multipart form data (e.g. `Response.formData()` is a battle tested encoding for multiple parts, however it is quite antiquated and rigid (has only two attributes that are specified in the RFC).
The main benefits of multipart is the ability to encode multiple mime types in the same response, however this is not useful when encoding HTML only.
The other benefit of encoding HTML patches in HTML itself is that those patches can be part of a normal response, allowing out of order updates after or in between normal HTML.

