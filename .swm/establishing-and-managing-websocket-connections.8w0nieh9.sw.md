---
title: Establishing and Managing WebSocket Connections
---
This document describes how the system establishes and manages <SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="138:10:10" line-data="  async connect(): Promise&lt;WebSocket&gt; {">`WebSocket`</SwmToken> connections for real-time, multiplexed communication. When a component requests a connection, the system either reuses an existing one or creates a new connection. If reconnecting, all previous subscriptions are restored, ensuring uninterrupted data flow across multiple channels.

# Establishing and Reusing <SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="138:10:10" line-data="  async connect(): Promise&lt;WebSocket&gt; {">`WebSocket`</SwmToken> Connections

<SwmSnippet path="/frontend/src/lib/k8s/api/v2/webSocket.ts" line="138">

---

In <SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="138:3:3" line-data="  async connect(): Promise&lt;WebSocket&gt; {">`connect`</SwmToken>, we first check if there's already an open <SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="138:10:10" line-data="  async connect(): Promise&lt;WebSocket&gt; {">`WebSocket`</SwmToken> connection and return it if so. If a connection is in progress, we wait for it to finish. Otherwise, we mark that we're connecting and build the <SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="138:10:10" line-data="  async connect(): Promise&lt;WebSocket&gt; {">`WebSocket`</SwmToken> URL using <SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="157:9:9" line-data="    const wsUrl = `${getBaseWsUrl()}${MULTIPLEXER_ENDPOINT}`;">`getBaseWsUrl`</SwmToken>, which adapts to the current app context. This is needed before we can actually open a new <SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="138:10:10" line-data="  async connect(): Promise&lt;WebSocket&gt; {">`WebSocket`</SwmToken> connection.

```typescript
  async connect(): Promise<WebSocket> {
    // Return existing connection if available
    if (this.socketMultiplexer?.readyState === WebSocket.OPEN) {
      return this.socketMultiplexer;
    }

    // Wait for existing connection attempt if in progress
    if (this.connecting) {
      return new Promise(resolve => {
        const checkConnection = setInterval(() => {
          if (this.socketMultiplexer?.readyState === WebSocket.OPEN) {
            clearInterval(checkConnection);
            resolve(this.socketMultiplexer);
          }
        }, 100);
      });
    }

    this.connecting = true;
    const wsUrl = `${getBaseWsUrl()}${MULTIPLEXER_ENDPOINT}`;

```

---

</SwmSnippet>

<SwmSnippet path="/frontend/src/lib/k8s/api/v2/webSocket.ts" line="27">

---

<SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="27:4:4" line-data="export function getBaseWsUrl(): string {">`getBaseWsUrl`</SwmToken> takes the app's base URL and swaps out 'http' for 'ws', so we get the right protocol for <SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="138:10:10" line-data="  async connect(): Promise&lt;WebSocket&gt; {">`WebSocket`</SwmToken> connections. We call <SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="28:3:3" line-data="  return getAppUrl().replace(&#39;http&#39;, &#39;ws&#39;);">`getAppUrl`</SwmToken> next to get the current app URL, which could change based on environment or deployment.

```typescript
export function getBaseWsUrl(): string {
  return getAppUrl().replace('http', 'ws');
}
```

---

</SwmSnippet>

<SwmSnippet path="/frontend/src/lib/k8s/api/v2/webSocket.ts" line="159">

---

Back in <SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="138:3:3" line-data="  async connect(): Promise&lt;WebSocket&gt; {">`connect`</SwmToken>, after getting the <SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="160:9:9" line-data="      const socket = new WebSocket(wsUrl);">`WebSocket`</SwmToken> URL and opening the connection, we check if we're reconnecting. If so, we call <SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="168:3:3" line-data="          this.resubscribeAll(socket);">`resubscribeAll`</SwmToken> to restore all previous subscriptions on the new socket. This keeps the client in sync after a disconnect.

```typescript
    return new Promise((resolve, reject) => {
      const socket = new WebSocket(wsUrl);

      socket.onopen = () => {
        this.socketMultiplexer = socket;
        this.connecting = false;

        // Only resubscribe if we're reconnecting after a disconnect
        if (this.isReconnecting) {
          this.resubscribeAll(socket);
        }
        this.isReconnecting = false;

        resolve(socket);
      };

```

---

</SwmSnippet>

## Restoring Subscriptions and Sending Requests

<SwmSnippet path="/frontend/src/lib/k8s/api/v2/webSocket.ts" line="193">

---

In <SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="193:1:1" line-data="  resubscribeAll(socket: WebSocket): void {">`resubscribeAll`</SwmToken>, we loop through all <SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="194:3:3" line-data="    this.activeSubscriptions.forEach(({ clusterId, path, query }) =&gt; {">`activeSubscriptions`</SwmToken>, which are objects with <SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="194:10:10" line-data="    this.activeSubscriptions.forEach(({ clusterId, path, query }) =&gt; {">`clusterId`</SwmToken>, path, and query. For each, we grab the <SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="195:3:3" line-data="      const userId = getUserIdFromLocalStorage();">`userId`</SwmToken> from local storage using <SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="195:7:7" line-data="      const userId = getUserIdFromLocalStorage();">`getUserIdFromLocalStorage`</SwmToken>, so each subscription request is tied to the current user. Next, we build the message to send over the socket.

```typescript
  resubscribeAll(socket: WebSocket): void {
    this.activeSubscriptions.forEach(({ clusterId, path, query }) => {
      const userId = getUserIdFromLocalStorage();
```

---

</SwmSnippet>

<SwmSnippet path="/frontend/src/lib/k8s/api/v2/webSocket.ts" line="196">

---

After getting <SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="200:1:1" line-data="        userId: userId || &#39;&#39;,">`userId`</SwmToken>, we build a <SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="196:6:6" line-data="      const requestMsg: WebSocketMessage = {">`WebSocketMessage`</SwmToken> for each subscription with <SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="197:1:1" line-data="        clusterId,">`clusterId`</SwmToken>, path, query, <SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="200:1:1" line-data="        userId: userId || &#39;&#39;,">`userId`</SwmToken>, and type 'REQUEST', then send it as JSON over the socket. This re-establishes all subscriptions after reconnect. Next, we move to sending multiplexed data over the socket in <SwmPath>[frontend/…/node/NodeShellTerminal.tsx](frontend/src/components/node/NodeShellTerminal.tsx)</SwmPath>.

```typescript
      const requestMsg: WebSocketMessage = {
        clusterId,
        path,
        query,
        userId: userId || '',
        type: 'REQUEST',
      };
      socket.send(JSON.stringify(requestMsg));
    });
  },
```

---

</SwmSnippet>

<SwmSnippet path="/frontend/src/components/node/NodeShellTerminal.tsx" line="225">

---

<SwmToken path="frontend/src/components/node/NodeShellTerminal.tsx" pos="225:3:3" line-data="  function send(channel: number, data: string) {">`send`</SwmToken> in <SwmPath>[frontend/…/node/NodeShellTerminal.tsx](frontend/src/components/node/NodeShellTerminal.tsx)</SwmPath> grabs the <SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="138:10:10" line-data="  async connect(): Promise&lt;WebSocket&gt; {">`WebSocket`</SwmToken>, checks if it's ready, encodes the string data, and prefixes it with the channel number as a single byte. This lets us multiplex several logical channels over one socket, so each message is routed correctly server-side.

```tsx
  function send(channel: number, data: string) {
    const socket = streamRef.current!.getSocket();

    // We should only send data if the socket is ready.
    if (!socket || socket.readyState !== 1) {
      console.debug('Could not send data to exec: Socket not ready...', socket);
      return;
    }

    const encoded = encoder.encode(data);
    const buffer = new Uint8Array([channel, ...encoded]);

    socket.send(buffer);
  }
```

---

</SwmSnippet>

## Handling <SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="138:10:10" line-data="  async connect(): Promise&lt;WebSocket&gt; {">`WebSocket`</SwmToken> Events and Errors

<SwmSnippet path="/frontend/src/lib/k8s/api/v2/webSocket.ts" line="175">

---

After <SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="168:3:3" line-data="          this.resubscribeAll(socket);">`resubscribeAll`</SwmToken> in <SwmToken path="frontend/src/lib/k8s/api/v2/webSocket.ts" pos="138:3:3" line-data="  async connect(): Promise&lt;WebSocket&gt; {">`connect`</SwmToken>, we attach handlers for messages, errors, and close events to manage the socket lifecycle.

```typescript
      socket.onmessage = this.handleWebSocketMessage.bind(this);

      socket.onerror = event => {
        this.connecting = false;
        console.error('WebSocket error:', event);
        reject(new Error('WebSocket connection failed'));
      };

      socket.onclose = () => {
        this.handleWebSocketClose();
      };
    });
  },
```

---

</SwmSnippet>

&nbsp;

*This is an auto-generated document by Swimm 🌊 and has not yet been verified by a human*

<SwmMeta version="3.0.0" repo-id="Z2l0aHViJTNBJTNBdHlwZXNjcmlwdC1oZWFkbGFtcCUzQSUzQXJpY2FyZG9sb3Blemc=" repo-name="typescript-headlamp"><sup>Powered by [Swimm](https://app.swimm.io/)</sup></SwmMeta>
