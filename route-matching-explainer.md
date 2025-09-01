# Declarative Route Matching

# Motivation and Use Cases
The web platform provides primitives for performing navigations declaratively, without the developer having to listen to events and curate the navigation as it is ongoing.
However, when it comes to same-document navigations, developers can only use event-driven scripts for the task, either through click capturing, navigation API, or by delegating this boilerplate work to frameworks.

While running script to manage navigations has its place and is here to stay, it has some drawbacks:
- A user script that is run during an interaction is more likely to cause jank than browser-internal features.
- Lifecycle management is a bug-prone area of software. Declaring the desired outcome rather than managing it explicitly with events can improve robustness of web apps by removing a lot of these fine details from user code.

## Navigation-aware styling
### Framework routers
While handling route-based style changes in scripting is certainly doable today, the author often has to go out of the way to curate the user experience of the navigation itself.
See unmodified example from [React Router](https://reactrouter.com/start/framework/pending-ui):

```tsx
import { NavLink } from "react-router";

function Navbar() {
  return (
    <nav>
      <NavLink to="/home">
        {({ isPending }) => (
          <span>Home {isPending && <Spinner />}</span>
        )}
      </NavLink>
      <NavLink
        to="/about"
        style={({ isPending }) => ({
          color: isPending ? "gray" : "black",
        })}
      >
        About
      </NavLink>
    </nav>
  );
}
```

- `<NavLink>` (similar to `<Link>`) wraps HTML links to connect them with the framework router, allowing the link itself to reflect state about its navigation.
- In turn, the router matches a URL pattern with a UI component, and also manages the different states of navigation, `isPending` in this case.

What if it could look like this?
```html
<style>
  nav {
    a:navigating-to(home)::after { content: "spinner.gif"; };
    a:navigating-to(about) { color: grey; }
  }
</style>
<nav>
  <a href="/about">About</a>
</nav>
```

### Vanilla

When no framework is used, this is even more complex, as the author has to manage the "pending" state and route changes themselves with script, e.g.:
```js
navigation.addEventListener("navigate", e => {
  e.intercept({
    async handler() {
      document.body.dataset.state = "pending";
      await load_actual_content();
      document.body.dataset.state = "loaded";
      document.body.dataset.route = route_from(e.destination.url);
   }
  });
});
```

## Declarative same-document view transitions

In order to integrate either of the above with view-transitions, the author has to either rely on the framework to bake view-transitions into the router, or add even more complexity to the interception.
The client event handling code can grow wild for things that are essentially curation of style.

```js
navigation.addEventListener("navigate", e => {
  e.intercept({
    async handler() {
      // We don't want to wait until the view transition is over to *start* loading the content.
      const actual_content_promise = load_actual_content();

      // This would display a transition until the pending state.
      // More complex curation requires more complex set up here.
      await document.startViewTransition(() => {
        document.body.dataset.state = "pending";
      }).updateCallbackDone;
      await actual_content_promise;
      document.body.dataset.state = "loaded";
      document.body.dataset.route = route_from(e.destination.url);
   }
  });
});
```

## Style based on current route
In addition to the curation of the navigation itself, it is a common technique in modern web apps to have a "shell" that is common between pages and mostly static, and an "outlet" area for the dynamic content.
However, some parts of the shell often still have some dynamic parts that appear on "some" pages or in some scenarios, or appear different based on the current page.

For example, a chat widget or members area might only appear in certain pages. A "related" `<aside>` element might only appear in article pages.

## Mapping between "Open/closed" UI elements and URLs
Dialog/popover and other "openable" UI elements are currently openable by a button and something like a command invoker.
However, sometimes an author would want to reflect this UI state in the URL, and have that URL lead to that UI state.

For example, having the URL `/settings` open the settings dialog, and having the settings dialog open shareable as the `/settings` URL.

To do this today, this mapping has to be done using events:
```js
settingsDialog.addEventListener("open", () => navigation.push("/settings");
settingsDialog.addEventListener("close", () => navigation.back());
navigation.addEventListener("event", e => {
  if (new URL(e.destination.url).pathname === "/settings") {
    e.intercept({ handler: () => settingsDialog.open() });
  }
});
```

## Scroll/gesture-based navigation with lazy-loading
Some modern UIs (e.g. Instagram, TikTok) use scroll-snapping or carousels to navigate between app fragments in a way that maps nicely to URL navigation.
For example, a URL retrieved from a QR code should not only scroll to the right app fragment but also render the correct state from the server.

While the web platform allows matching between scrolling and URLs using ID mapping, this is:
- limited to hash-fragments only, while some of these URL changes are better expressed by URLs that are seen by the server (like path changes).
- Uni-directional. The user scrolling to a fragment doesn't automatically change the URL
- Loading content lazily based on element proximity to the viewport is cumbersome, and requires careful use of `IntersectionObserver` or `content-visiblity` (including the `contentvisibilityautostatechange` event).

# The proposed solution: 

## Overview
Proposing to include a few specific client-side routing capabilities in the web platform, that should enable:
- CSS-first support for optimistic, pending and transition UI, as well as for some simple route-matching styles. This is designed to be useful either with or without a framework client-side router.
- HTML-first support for integration between user interactions (scroll-snapping, popovers/dialogs) and URL navigation.
- A set of rudimentary out-of-the-box routing capabilities for apps that don't rely on a full-fledged framework-based router.
- Together with declarative patching, provide a complete declarative lifecycle for updating a document based on navigations without a full refresh.
- Work in the context of the whole document or scoped to a component

## The basics
- A "router" is a set of rules.
- A rule can either apply to a single URL, or to a transition between two URLs (a navigation)
- A router is scoped to the whole document or to an element.

Each rule would always have the following:
- A name (to be able to identify it in CSS or JS)
- A "matcher" or "selector". Either a URL pattern that matches it to the current URL, or 1-2 URL patterns that match it with an ongoing navigation.

```html
<head>
  <!-- This routemap applies to the whole document. It doesn't interfere with a framework router because it doesn't intercept navigations. -->
  <script type=routemap>
     {"rules": [
        {"name": "home", "pattern": {"pathname": "/" } }}, 
        {"name": "about", "pattern": {"pathname": "/about" }},
        {"name": "article", "pattern": {"pathname": "/article/:article-id" } }
        {"name": "between-articles", "between": ["article", "article"] } }
     ]}
  </script>
</head>
<body>
  <section id=dashboard>
    <!-- This routemap applies to the section -->
    <script type=routemap>
     {"rules": [
        {"name": "settings", "pattern": {"pathname": "/dashboard/settings" } }}
     ]} 
    </script>
  </section>
</body>
```

## CSS reflection

Naming a set of these rules in HTML already gives us something that CSS can build on:
```css
@route(home) {
  #chat-widget { display: none; } 
}

#dashboard:route(settings) {
  .settings-button { display:none; }
}

nav {
  a:navigation-to(pending) .spinner {
    animation: spin;
  }
}

/* navigation-based view-transition can work out of the box because we can count on the final CSS state */
@view-transition {
  navigation: auto;
}

/* or be route-specific */
@view-transition {
  navigation: route(between-articles);
  types: slide-3d;
}
```

Having the simple route-matching with CSS reflection is likely the first self-contained deliverable from this proposal.

## Element/route binding

While CSS is a great fit for some use cases, including ones that display and hide visual elements based on route, this doesn't work well
with UI elements that are more than visual, like dialogs and popovers, or elements that can be scrolled to.

To achieve the uses cases for dialog or scroll-based navigation in a way that works well with URL navigation, proposing to allow "binding" an element to a route and route params, in a similar way to command invokers (though bidirectional).
When an element is bound to a route+param, it is:
- opened/scrolled to when navigating to that route+param
- Changes the URL based on the route rules when opened by the user
- Emits events that help lazily load the content for the route if the scrolling element is close to the viewport

```html
<script type=routemap>
[
  {"name": "feed", "pattern": {"pathname": "/feed/:feedid"} },
  {"name": "settings", "pattern": {"search": "?settings=show"}}
]
</script>
<main class="app-carousel">
  <...>
  <section route=feed data-feedid="feed12"></section>
  <...>
</main>
<dialog route=settings>Settings</dialog>
<!-- This would open the settings dialog -->
<a href="?settings=show">Settings</a>
<!-- This would scroll the app carousel to feed 12 -->
<a href="/feed/feed12">Feed 12</a>

<script>
// Lazily load route content
document.routeMap.get("feed").param("feedid").addEventListener("prepare", e =>
    fetch(`/feed-content?id=${e.value}`));
</script>
```

## Basic interception & history-handling

In addition to CSS reflection, some basic navigation interception capabilities can be provided out of the box:
- Intercepting without side effects (navigations that just change style)
- Changing the history mode (e.g. having some routes not add a history entry or not change the URL at all)

This complements the CSS reflection and element binding features, as with those some same-document navigation can have a meaningful UI effect (either style-only or semantic) without necessarily requiring a custom script.

```html
</head>
<body>
  <!-- A scoped router would only intercept navigations that originated from within the scope -->
  <section id=dashboard>
    <!-- This can work without a JS router at all!
         Linking to `/dashboard/settings` would replace the URL and display the settings without event-driven scripting -->
    <script type=routemap>
     {"rules": [
        {"name": "settings", "pattern": {"pathname": "/dashboard/settings"},
         "mode": "intercept", "history": "replace" }]} 
    </script>
  </section>
</body>
```

## Declarative patch-based document updates

(Future vision of putting it all together)

Together with the [patching](https://github.com/WICG/declarative-partial-updates/blob/main/patching-explainer.md) feature, routes can provide a fully declarative mechanism for updating the document, with the decision of what content goes in each route offloaded to a server or service worker:
```html
<head>
  <script type=routemap>
     {
        "rules": [
        {"pattern": {"pathname": "/*" }, "patchSource": "/content/patch", "mode": "same-document" }, 
        {"name": "home", "pattern": {"pathname": "/" } }}, 
        {"name": "about", "pattern": {"pathname": "/about" }},
        {"name": "article", "pattern": {"pathname": "/article/:article-id" } }
        {"name": "between-articles", "between": ["article", "article"] } }
     ]}
  </script>
</head>
```

By providing a `patchSource` to a rule or set of rules, which is a URL or a `serviceWorker` (exact semantics TBD), updating the document is performed by the browser, without interaction-time javascript.
The stream of interleved patches is fetched from the URL (or from the service worker using the navigation URL), and applied to the document or scope element, while reflecting the intermediate states to CSS and performing view transitions etc.

When combined with scroll-based navigation, the patch stream can be fetched lazily as the bound element approaches the viewport, making lazy loading of scrollable content easier.


# Summary
- Declarative route matching is about mapping between UI (style, openables, scroll) and URL navigation.
- It is done by using an element or document-scoped set of rules that apply to a `URLPattern` or a pair,
  and then mapping those rules to CSS declarations or scrollable/openable elements.
- It can be used to offload the pending/optimistic aspect of navigations to the browser, also when using a framework router for content updates.
- It is also designed in a way that can be used as a standalone router, when the route changes are limited to UI/style, alongside patching, or by integrating JS-based routing with its new events.




