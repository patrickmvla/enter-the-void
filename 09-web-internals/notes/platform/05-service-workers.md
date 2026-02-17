# Service Workers — The Programmable Network Proxy

## What a Service Worker Is

A service worker is a script that the browser runs in the background,
separate from the page. It acts as a **programmable network proxy** —
it intercepts every network request from its scope and can respond
from cache, from the network, or from generated data.

Service workers enable:
- Offline-capable web apps
- Background data synchronization
- Push notifications
- Request/response interception and transformation

They do NOT have access to the DOM — they communicate with pages
via `postMessage`.

---

## Lifecycle

The service worker lifecycle is deliberately separate from the page
lifecycle. This ensures that a new version of your site doesn't break
in-flight requests from the old version.

```
┌──────────┐     ┌──────────┐     ┌──────────┐
│ Register │────▶│ Install  │────▶│ Waiting  │
└──────────┘     └────┬─────┘     └────┬─────┘
                      │                │
                 install event    (old SW still active)
                      │                │
                 caching assets        │ old SW dies or
                      │                │ skipWaiting()
                      │                │
                      ▼                ▼
                 ┌──────────┐     ┌──────────┐
                 │ Error    │     │ Activate │
                 │ (retry)  │     └────┬─────┘
                 └──────────┘          │
                                  activate event
                                       │
                                  clean old caches
                                       │
                                       ▼
                                  ┌──────────┐
                                  │ Active   │◀──── fetch events
                                  │ (idle)   │◀──── push events
                                  └────┬─────┘      sync events
                                       │
                                  ┌────┴─────┐
                                  │Terminated│  (browser can kill idle SW
                                  │          │   and restart on next event)
                                  └──────────┘
```

### Registration

```javascript
// main.js — register the service worker
if ('serviceWorker' in navigator) {
  const registration = await navigator.serviceWorker.register('/sw.js', {
    scope: '/',           // Controls pages under this path
    updateViaCache: 'none' // Always check server for updates (default in modern browsers)
  });

  console.log('SW registered, scope:', registration.scope);
}
```

**Scope**: a service worker controls all pages whose URL starts with
the scope path. `/sw.js` with scope `/` controls the entire origin.
A SW at `/app/sw.js` with default scope controls only `/app/*`.

You cannot set a scope **broader** than the service worker's location
unless the server sends a `Service-Worker-Allowed` header.

### Install Event

Fires when a new service worker version is detected. Use this to
pre-cache essential resources:

```javascript
// sw.js
const CACHE_NAME = 'app-v2';
const PRECACHE_URLS = [
  '/',
  '/index.html',
  '/styles/main.css',
  '/scripts/app.js',
  '/images/logo.png',
];

self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE_NAME)
      .then(cache => cache.addAll(PRECACHE_URLS))
  );
  // If any URL fails to cache, the install fails
  // and the SW stays in the "install" state (retries later)
});
```

`event.waitUntil()` extends the event's lifetime. The SW doesn't
move to the next state until the Promise resolves. If it rejects,
the installation fails.

### Waiting State

After install, the new SW enters **waiting** state. It does NOT take
control until all pages using the OLD service worker are closed. This
prevents a page from running with a mix of old and new cached resources.

```javascript
// Skip waiting — take control immediately (use with caution):
self.addEventListener('install', (event) => {
  self.skipWaiting();
  event.waitUntil(cacheAssets());
});
```

### Activate Event

Fires when the SW takes control. This is the time to clean up old
caches:

```javascript
self.addEventListener('activate', (event) => {
  event.waitUntil(
    caches.keys().then(cacheNames => {
      return Promise.all(
        cacheNames
          .filter(name => name !== CACHE_NAME)
          .map(name => caches.delete(name))
      );
    })
  );

  // Claim all clients immediately (don't wait for reload):
  self.clients.claim();
});
```

`clients.claim()` makes the SW take control of all in-scope pages
immediately, without waiting for navigation. Paired with
`skipWaiting()`, this enables instant updates.

### Update Flow

The browser checks for SW updates:
- Every time a navigation occurs to an in-scope page
- Every 24 hours (at minimum)
- When `registration.update()` is called manually

**Byte-for-byte comparison**: the browser compares the new SW script
(and imported scripts) byte-by-byte with the current one. Any
difference triggers a new install.

```javascript
// Update detection in the page:
registration.addEventListener('updatefound', () => {
  const newWorker = registration.installing;

  newWorker.addEventListener('statechange', () => {
    if (newWorker.state === 'installed') {
      if (navigator.serviceWorker.controller) {
        // New SW installed but waiting — prompt user to refresh
        showUpdateBanner();
      }
    }
  });
});
```

---

## Fetch Interception

The `fetch` event is the core of service worker power. Every network
request from a controlled page passes through the SW:

```javascript
self.addEventListener('fetch', (event) => {
  // event.request — the Request object
  // event.respondWith() — provide a custom Response
  // If you don't call respondWith(), the request goes to the network normally

  event.respondWith(
    handleRequest(event.request)
  );
});
```

### Request Object Details

```javascript
self.addEventListener('fetch', (event) => {
  const request = event.request;

  request.url;         // Full URL
  request.method;      // GET, POST, etc.
  request.headers;     // Headers object
  request.mode;        // 'navigate', 'cors', 'no-cors', 'same-origin'
  request.destination; // 'document', 'script', 'style', 'image', etc.
  request.credentials; // 'same-origin', 'include', 'omit'

  // Navigation requests (page loads):
  if (request.mode === 'navigate') { /* ... */ }

  // API requests:
  if (request.url.includes('/api/')) { /* ... */ }

  // Image requests:
  if (request.destination === 'image') { /* ... */ }
});
```

---

## Caching Strategies

### 1. Cache First (Cache Falling Back to Network)

Best for: static assets (CSS, JS, images, fonts) — immutable or
versioned resources.

```javascript
async function cacheFirst(request) {
  const cached = await caches.match(request);
  if (cached) return cached;

  const response = await fetch(request);
  if (response.ok) {
    const cache = await caches.open(CACHE_NAME);
    cache.put(request, response.clone());
    // Must clone — response body can only be consumed once
  }
  return response;
}
```

```
Request → Cache hit? → YES → Return cached response
                 │
                 NO
                 │
                 ▼
          Fetch from network → Cache response → Return response
```

### 2. Network First (Network Falling Back to Cache)

Best for: frequently updated resources where freshness matters (API
responses, dynamic HTML).

```javascript
async function networkFirst(request) {
  try {
    const response = await fetch(request);
    if (response.ok) {
      const cache = await caches.open(CACHE_NAME);
      cache.put(request, response.clone());
    }
    return response;
  } catch (error) {
    const cached = await caches.match(request);
    if (cached) return cached;

    // Both network and cache failed — return offline page
    return caches.match('/offline.html');
  }
}
```

```
Request → Fetch from network → Success → Cache + Return
                 │
              Failure
                 │
                 ▼
          Cache hit? → YES → Return cached (stale)
                 │
                 NO → Return offline fallback
```

### 3. Stale-While-Revalidate

Best for: resources that should be fast but also fresh (avatars,
non-critical API data, semi-dynamic content).

```javascript
async function staleWhileRevalidate(request) {
  const cache = await caches.open(CACHE_NAME);
  const cached = await cache.match(request);

  // Always fetch in background to update cache:
  const fetchPromise = fetch(request).then(response => {
    if (response.ok) {
      cache.put(request, response.clone());
    }
    return response;
  });

  // Return cached immediately if available, otherwise wait for network:
  return cached || fetchPromise;
}
```

```
Request → Cache hit? → YES → Return cached immediately
              │                    │
              │              (background: fetch → update cache)
              │
              NO → Wait for fetch → Cache + Return
```

### 4. Cache Only

Best for: pre-cached assets during install, offline-first data.

```javascript
async function cacheOnly(request) {
  return caches.match(request);
}
```

### 5. Network Only

Best for: non-GET requests, analytics, real-time data that can't
be cached.

```javascript
async function networkOnly(request) {
  return fetch(request);
}
```

### Strategy Selection Pattern

```javascript
self.addEventListener('fetch', (event) => {
  const { request } = event;
  const url = new URL(request.url);

  // Skip cross-origin requests
  if (url.origin !== self.location.origin) {
    return; // Let the browser handle it
  }

  // Navigation requests — network first with offline fallback
  if (request.mode === 'navigate') {
    event.respondWith(networkFirst(request));
    return;
  }

  // Static assets — cache first (versioned by build hash)
  if (url.pathname.match(/\.(css|js|woff2?|png|jpg|svg)$/)) {
    event.respondWith(cacheFirst(request));
    return;
  }

  // API calls — stale-while-revalidate
  if (url.pathname.startsWith('/api/')) {
    event.respondWith(staleWhileRevalidate(request));
    return;
  }

  // Everything else — network first
  event.respondWith(networkFirst(request));
});
```

---

## Cache API Details

The Cache API is a key-value store of Request → Response pairs,
available to both service workers and pages.

```javascript
// Open (or create) a named cache:
const cache = await caches.open('my-cache-v1');

// Store a response:
await cache.put(request, response);

// Store from URL (fetches and caches):
await cache.add('/styles.css');

// Store multiple:
await cache.addAll(['/a.js', '/b.js', '/c.css']);

// Retrieve:
const response = await cache.match(request);

// Match with options:
const response = await cache.match(request, {
  ignoreSearch: true,    // Ignore URL query params
  ignoreMethod: true,    // Match regardless of HTTP method
  ignoreVary: true,      // Ignore Vary header
});

// Search across ALL named caches:
const response = await caches.match(request);

// List all cache names:
const names = await caches.keys();

// Delete a cache:
await caches.delete('old-cache');

// Delete a specific entry:
await cache.delete(request);

// List all entries:
const requests = await cache.keys();
```

### Cache Storage Limits

The browser allocates storage per origin:

- Chrome: up to 80% of total disk space per origin (with overall 60%
  limit across all origins)
- Firefox: up to 2GB per origin (configurable)
- Safari: ~50MB initially, then prompts the user; purges after 7 days
  without site visit

Check available storage:

```javascript
const estimate = await navigator.storage.estimate();
console.log(`Used: ${estimate.usage} bytes`);
console.log(`Available: ${estimate.quota} bytes`);
console.log(`Percent: ${(estimate.usage / estimate.quota * 100).toFixed(1)}%`);
```

Request persistent storage (won't be evicted under storage pressure):
```javascript
const persisted = await navigator.storage.persist();
// true = granted, false = denied (browser decides based on engagement)
```

---

## Navigation Preload

Problem: with a network-first strategy, the service worker must boot
up (~50-100ms) before it can make the fetch. This adds latency.

**Navigation preload** starts the network request in parallel with
the service worker boot:

```javascript
// activate event — enable navigation preload
self.addEventListener('activate', (event) => {
  event.waitUntil(
    self.registration.navigationPreload.enable()
  );
});

// fetch event — use the preloaded response
self.addEventListener('fetch', (event) => {
  if (event.request.mode === 'navigate') {
    event.respondWith(
      (async () => {
        // Try the preloaded response first:
        const preloadResponse = await event.preloadResponse;
        if (preloadResponse) return preloadResponse;

        // Fall back to cache:
        const cached = await caches.match(event.request);
        if (cached) return cached;

        // Fall back to regular fetch:
        return fetch(event.request);
      })()
    );
  }
});
```

Timeline comparison:
```
Without preload:
[SW boot: 80ms] → [fetch: 200ms] → Response at 280ms

With preload:
[SW boot: 80ms  ]
[fetch:   200ms       ] → Response at 200ms (fetch started in parallel)
```

---

## Background Sync

Defers actions until the user has connectivity. If the user submits a
form offline, background sync retries when the connection returns.

```javascript
// Page — register a sync:
async function submitForm(data) {
  // Store the data for later:
  await saveToIndexedDB('outbox', data);

  // Register sync tag:
  const registration = await navigator.serviceWorker.ready;
  await registration.sync.register('submit-form');
}
```

```javascript
// sw.js — handle the sync:
self.addEventListener('sync', (event) => {
  if (event.tag === 'submit-form') {
    event.waitUntil(
      (async () => {
        const items = await getFromIndexedDB('outbox');
        for (const item of items) {
          await fetch('/api/submit', {
            method: 'POST',
            body: JSON.stringify(item),
          });
          await removeFromIndexedDB('outbox', item.id);
        }
      })()
    );
  }
});
```

If `waitUntil` rejects, the browser retries with exponential backoff.
After several failures, it gives up.

### Periodic Background Sync

Periodic sync runs at browser-determined intervals (minimum ~12 hours):

```javascript
// Register:
const registration = await navigator.serviceWorker.ready;
await registration.periodicSync.register('update-articles', {
  minInterval: 24 * 60 * 60 * 1000, // 24 hours (minimum)
});

// sw.js
self.addEventListener('periodicsync', (event) => {
  if (event.tag === 'update-articles') {
    event.waitUntil(fetchAndCacheArticles());
  }
});
```

The browser decides the actual frequency based on:
- Site engagement score
- Whether the device is on Wi-Fi
- Battery level
- Whether the user has the site installed as a PWA

---

## Push Notifications

Service workers can receive push messages even when the page is closed.

### The Push Flow

```
Your Server → Push Service (FCM/APNs/Mozilla) → Browser → Service Worker
```

```javascript
// 1. Subscribe (in page):
const registration = await navigator.serviceWorker.ready;
const subscription = await registration.pushManager.subscribe({
  userVisibleOnly: true,  // Must show a notification (Chrome requirement)
  applicationServerKey: urlBase64ToUint8Array(VAPID_PUBLIC_KEY),
});

// subscription.endpoint → the push service URL
// subscription.keys.p256dh → encryption key
// subscription.keys.auth → auth secret
// Send these to your server
```

```javascript
// 2. Push from server (using web-push library):
// POST to subscription.endpoint with encrypted payload
// Uses VAPID (Voluntary Application Server Identification)
// and RFC 8291 (Message Encryption for Web Push)
```

```javascript
// 3. Receive in service worker:
self.addEventListener('push', (event) => {
  const data = event.data?.json();

  event.waitUntil(
    self.registration.showNotification(data.title, {
      body: data.body,
      icon: '/icon-192.png',
      badge: '/badge-72.png',
      data: { url: data.url },
      actions: [
        { action: 'open', title: 'Open' },
        { action: 'dismiss', title: 'Dismiss' },
      ],
    })
  );
});

// 4. Handle notification click:
self.addEventListener('notificationclick', (event) => {
  event.notification.close();

  if (event.action === 'open' || !event.action) {
    event.waitUntil(
      clients.openWindow(event.notification.data.url)
    );
  }
});
```

### Push Encryption

Push payloads are encrypted end-to-end using:
- **ECDH** (Elliptic Curve Diffie-Hellman) for key agreement
- **HKDF** (HMAC-based Key Derivation) for deriving content encryption key
- **AES-128-GCM** for encrypting the payload

The push service (FCM, Mozilla, etc.) cannot read your notification
content — it just relays encrypted bytes.

---

## Client Communication

Service workers communicate with controlled pages through MessageChannel:

```javascript
// sw.js — send message to all controlled pages:
async function notifyClients(message) {
  const clientList = await self.clients.matchAll({
    type: 'window',
    includeUncontrolled: false,
  });

  for (const client of clientList) {
    client.postMessage(message);
  }
}

// Page — receive:
navigator.serviceWorker.addEventListener('message', (event) => {
  console.log('From SW:', event.data);
});
```

```javascript
// Page — send to SW and get response:
function sendToSW(message) {
  return new Promise((resolve) => {
    const channel = new MessageChannel();
    channel.port1.onmessage = (event) => resolve(event.data);
    navigator.serviceWorker.controller.postMessage(message, [channel.port2]);
  });
}

// sw.js — respond on the port:
self.addEventListener('message', (event) => {
  const port = event.ports[0];
  port.postMessage({ result: 'done' });
});
```

---

## Service Worker Termination

The browser aggressively terminates idle service workers to save memory:

- Terminated after ~30 seconds of idle time
- Terminated after ~5 minutes of continuous execution
- Any `event.waitUntil()` Promise keeps the SW alive
- State in global variables is lost on termination

```javascript
// BAD — state lost on termination:
let visitCount = 0;
self.addEventListener('fetch', (event) => {
  visitCount++;  // Resets to 0 every time SW restarts
});

// GOOD — use IndexedDB or Cache API for persistent state:
self.addEventListener('fetch', (event) => {
  // Read/write from IndexedDB
});
```

---

## Debugging Service Workers

```
Chrome DevTools:
  Application → Service Workers
  - See registered SWs, their state, and update controls
  - "Update on reload" checkbox for development
  - "Bypass for network" to disable SW during debugging

chrome://serviceworker-internals
  - Low-level SW debugging across all origins
  - Force unregister, stop, inspect
```

```javascript
// Unregister all service workers:
const registrations = await navigator.serviceWorker.getRegistrations();
for (const reg of registrations) {
  await reg.unregister();
}
```

---

## Key Takeaways

| Concept | Why It Matters |
|---------|---------------|
| Lifecycle is separate from page | Prevents partial cache states during updates |
| Install → Wait → Activate | Ensures clean transitions between SW versions |
| skipWaiting + clients.claim | Instant updates (risky — can break in-flight requests) |
| Cache strategies are per-resource | Static: cache-first. Dynamic: network-first. Semi-fresh: stale-while-revalidate |
| Response.clone() | Response body is a stream — can only be consumed once |
| Navigation preload | Eliminate SW boot latency for navigation requests |
| Background sync | Reliable form submission and data sync despite connectivity |
| Push encryption | End-to-end encrypted — push service can't read content |
| SW terminates aggressively | Don't store state in global variables |
| Scope limits control | SW only intercepts requests within its scope path |
