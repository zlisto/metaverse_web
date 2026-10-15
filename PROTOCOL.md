# The Metaverse Web protocol, mvw/0.1

**Status: draft.** Written after the first two worlds (Yale SOM Floor 2 and Darul Kebab) and the first browser were built and linked.

The Metaverse Web is the World Wide Web with 3D worlds in place of pages:

| The web | The Metaverse Web |
| --- | --- |
| A page | A **world**: a website you walk around in |
| A link | A **portal**: a doorway you walk through |
| A URL, plus `#anchor` | A world's URL, plus `?spawn=<id>` |
| The web browser | The **Metaverse Browser**: carries you, your name and your history between worlds |

The protocol is small on purpose. A world may use any engine (three.js, Babylon, PlayCanvas, Unity WebGL, anything). It only has to do the things in this document. Anyone can build a world, host it anywhere, and link to anyone else's world, without asking permission and without a central server.

## 1. The manifest

Every world serves a JSON file at `.well-known/metaverse.json`, relative to the world's address. For `https://example.com/` that is `https://example.com/.well-known/metaverse.json`; for a world under a path, like `https://zlisto.github.io/yale_som_virtual/`, it is `https://zlisto.github.io/yale_som_virtual/.well-known/metaverse.json`.

```json
{
  "protocol": "mvw/0.1",
  "name": "Darul Kebab",
  "description": "A Cambridge restaurant on Mass Ave: tables, a buffet, and the owner holding biryani.",
  "spawns": {
    "front_door": "Just inside the front door, facing the dining room",
    "default": "Same as front_door"
  },
  "portals": [
    {
      "id": "to_yale",
      "label": "Yale SOM · Evans Hall",
      "href": "http://localhost:5173/",
      "spawn": "darul_kebab_portal",
      "where": "The glass front door on the west wall"
    }
  ]
}
```

| Field | Required | Meaning |
| --- | --- | --- |
| `protocol` | yes | Always `"mvw/0.1"` for this version. |
| `name` | yes | The world's name, shown in the browser. |
| `description` | no | One or two plain sentences: what's here. Browsers show it, and AI agents read it to decide where to go. |
| `spawns` | yes | Arrival points, as `id → plain-text description`. Must include `default`. |
| `portals` | no | The world's outgoing links. `href` is the target world's address, `spawn` is where you arrive there, `label` names the destination, and `where` says where the portal is in this world. |

The manifest is to a world what a page's `<head>` and its links are to a web page. A browser, a search engine or an agent can map the Metaverse Web by following `portals[].href` from world to world, the way a web crawler follows links.

**Serving it from another site.** The browser fetches the manifest from its own origin, so the world's server must allow cross-origin `GET` requests for this one file (`Access-Control-Allow-Origin: *`). GitHub Pages and the Vite dev server already do. If the manifest can't be read, the world still works; the browser just can't show its name and portals.

## 2. Addresses and spawn points

A world's address is its URL, plus an optional `spawn` query parameter:

```
http://localhost:5174/?spawn=front_door
```

The world starts you at that spawn point, or at `default` if it's missing or unknown. This works with or without a browser, so a plain link to a world still works.

A spawn id is a public promise, like a URL: once other worlds link to `?spawn=front_door`, keep that id working. Rename the description freely, never the id.

## 3. Messages

Inside the Metaverse Browser, a world runs in an iframe and talks to the browser with `window.postMessage`. Every message is a plain object with `protocol: "mvw/0.1"` and a `type`.

| Direction | `type` | Fields | Meaning |
| --- | --- | --- | --- |
| world → browser | `ready` | none | The world has loaded and is listening. Send it as early as possible. |
| browser → world | `enter` | `identity: { name, color }` | Who is visiting. Sent in reply to `ready`, and again whenever the visitor changes it. `color` is a CSS hex color. |
| world → browser | `presence` | `where` (text) | Where the visitor is in the world, in plain words, e.g. `"Darul Kebab · at the buffet"`. Send it when it changes. |
| world → browser | `navigate` | `href`, `spawn`, `back` | The visitor walked through a portal. `back` is the spawn id in the world being left, so Back returns the visitor next to that portal. |

**Rules for browsers.**
- Only accept messages from the world currently on screen (`event.source` is that iframe's window).
- Only follow `http:` and `https:` addresses. Treat every field as untrusted text.
- The browser decides whether to follow a `navigate`, the way a web browser decides whether to follow a link.

**Rules for worlds.**
- A world must work on its own, opened directly in a tab. When it isn't in a browser (`window.parent === window`), it follows its own portals by setting `location.href` to `href?spawn=<spawn>`.
- Worlds never receive anything secret. `identity` is a display name and a color, nothing more.

## 4. Portals

How a portal looks is up to the world. The reference portal is a glowing pink frame with a swirl and a sign naming the destination, sized to fit a doorway.

Every portal should:
- **Fire once,** when the visitor walks through its opening.
- **Arm only after the visitor has stepped away** (the reference portal uses 1.2 m), so arriving next to a portal doesn't bounce you straight back.
- **Say where it goes,** with a visible label.

## 5. Arrival and departure

These keep travel feeling like one continuous place instead of a page reload.

- **Leaving:** when a portal fires, the world fades its own music and sounds out over about 0.4 s.
- **The transition belongs to the browser.** The browser fades the screen to black, plays the teleport sound (a rising and falling whoosh), and loads the next world. A world opened on its own plays its own whoosh and waits about 0.45 s before changing `location.href`.
- **Arriving:** the world places the visitor at the requested spawn, facing into the space. Sound and music start on the visitor's first click, because browsers block audio until then.

## 6. The Metaverse Browser

The reference browser (`metaverse_browser/`) shows what a browser is expected to do:

- **Address bar:** type a world address, with or without `?spawn=`.
- **Back and Forward:** a history of `{ href, spawn }` entries. Following a portal rewrites the current entry's spawn to the portal's `back`, so Back lands next to the portal you took.
- **Bookmarks.**
- **Identity:** your name and color, kept in the browser's local storage and sent to every world you enter.
- **World info:** the manifest's name and description, your current `presence`, the portals (clickable), and the spawn points (clickable).
- **Its own address** keeps the current world in `?world=<address>`, so reloading the browser keeps you where you were.

## 7. Make your own world and link it in

1. **Build a world** as a normal website, with any 3D engine.
2. **Add spawn points** and read `?spawn=` on load. Always have `default`.
3. **Copy the client.** Drop [`mvw.js`](https://github.com/zlisto/darul_kebab/blob/main/src/mvw.js) into your project. Call `presence(where)` when the visitor moves to a new area, `navigate(href, spawn, back)` when they walk through a portal, and `onIdentity(fn)` to greet them by name. If you use three.js and React, copy `Portal.jsx` too.
4. **Write your manifest** at `public/.well-known/metaverse.json` (for Vite; anywhere that serves `/.well-known/` for other setups).
5. **Host it** anywhere that serves static files: GitHub Pages, Netlify, Vercel, your own server. On GitHub Pages, folders starting with a dot are skipped unless you deploy with GitHub Actions or add an empty `.nojekyll` file, so check that `.well-known/metaverse.json` actually loads.
6. **Link both ways, and get listed.** Add your world to the [registry](https://github.com/zlisto/metaverse_registry) to get a portal in Metaverse Square. Put a portal in your world whose `href` is someone else's world and whose `spawn` is one of their spawn ids. Send them your address and a spawn id so they can add a portal back to you.

To check a world: open it on its own, then open it in the Metaverse Browser. You should see its name in the info panel, your location updating as you walk, and its portals working in both cases.

## Live worlds and reference code

| What | Live | Code |
| --- | --- | --- |
| Metaverse Browser | https://zlisto.github.io/metaverse_browser/ | [zlisto/metaverse_browser](https://github.com/zlisto/metaverse_browser) |
| Metaverse Square (hub) | https://zlisto.github.io/hub_world/ | [zlisto/hub_world](https://github.com/zlisto/hub_world) |
| Yale SOM, Evans Hall Floor 2 | https://zlisto.github.io/yale_som_virtual/ | [zlisto/yale_som_virtual](https://github.com/zlisto/yale_som_virtual) |
| Darul Kebab | https://zlisto.github.io/darul_kebab/ | [zlisto/darul_kebab](https://github.com/zlisto/darul_kebab) |
| World Starter (template) | https://zlisto.github.io/world_starter/ | [zlisto/world_starter](https://github.com/zlisto/world_starter) |
| Registry of worlds | | [zlisto/metaverse_registry](https://github.com/zlisto/metaverse_registry) |

In every world, `src/mvw.js` is the world-side client and `src/Portal.jsx` the three.js portal; both are the same file everywhere. `src/audio.js` (generated music and the whoosh) is optional and not part of the protocol.

**To be listed:** add your world to the registry by pull request ([how](https://github.com/zlisto/metaverse_registry#add-your-world)). Metaverse Square rebuilds daily and gives every registered world a portal; its spawn point in front of your portal is `by_<your id>`.

## Ports for local development

| Port | What |
| --- | --- |
| 5173 | Yale SOM, Evans Hall Floor 2 |
| 5174 | Darul Kebab |
| 5175 | The Metaverse Browser |
| 5176 | World Starter |
| 5178 | Metaverse Square |

## Versioning

The `protocol` field names the version. A browser that sees a version it doesn't know should still try to show the world, and ignore fields and message types it doesn't recognize. New fields are added in minor versions (`mvw/0.2`); anything that breaks old worlds needs a new major version.

## Open questions for 0.2

- **Avatars:** should identity carry an avatar that worlds render, so other visitors can see you?
- **Shared presence:** seeing the other people in the same world at the same time.
- **A declarative scene format,** so a browser can render simple worlds itself, the way it renders HTML.
- **Agent actions beyond walking:** "talk to the owner", "read the menu", "sit down".
- **Discovery:** a directory or search engine of worlds, built by crawling manifests.
- **Trust:** letting visitors mark which worlds may receive their name.
