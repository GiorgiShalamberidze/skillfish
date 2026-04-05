---
name: websocket-engineer
description: Real-time systems: WebSocket implementation, Socket.io patterns, server-sent events, connection management, reconnection strategies, and scaling with Redis pub/sub.
---

# WebSocket Engineer

Build and operate production-grade real-time communication systems. This skill covers the full spectrum from raw WebSocket protocol mechanics through Socket.io abstractions, server-sent events, connection lifecycle management, reconnection resilience, horizontal scaling with Redis pub/sub, security hardening, and automated testing strategies.

## Table of Contents

1. [WebSocket Fundamentals](#1-websocket-fundamentals)
2. [Socket.io](#2-socketio)
3. [Server-Sent Events](#3-server-sent-events)
4. [Connection Management](#4-connection-management)
5. [Reconnection Strategies](#5-reconnection-strategies)
6. [Scaling](#6-scaling)
7. [Security](#7-security)
8. [Testing](#8-testing)

---

## 1. WebSocket Fundamentals

### When to Use

- The application requires bidirectional, low-latency communication between client and server
- You need push-based updates without polling
- Use cases: chat, live dashboards, collaborative editing, gaming, financial tickers

### Protocol Handshake

WebSocket connections begin as an HTTP/1.1 upgrade request. The client sends an `Upgrade: websocket` header and the server responds with `101 Switching Protocols`.

```
GET /chat HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
```

```
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

### Frame Structure and Opcodes

| Opcode | Type | Purpose |
|--------|------|---------|
| `0x0` | Continuation | Continues a fragmented message |
| `0x1` | Text | UTF-8 encoded text data |
| `0x2` | Binary | Raw binary data |
| `0x8` | Close | Initiates connection close |
| `0x9` | Ping | Heartbeat request |
| `0xA` | Pong | Heartbeat response |

### Native Browser API

```javascript
const ws = new WebSocket('wss://api.example.com/ws');

ws.addEventListener('open', () => {
  console.log('Connected');
  ws.send(JSON.stringify({ type: 'subscribe', channel: 'prices' }));
});

ws.addEventListener('message', (event) => {
  const data = JSON.parse(event.data);
  handleMessage(data);
});

ws.addEventListener('close', (event) => {
  console.log(`Closed: code=${event.code} reason=${event.reason} clean=${event.wasClean}`);
});

ws.addEventListener('error', (event) => {
  console.error('WebSocket error:', event);
});
```

### Node.js ws Library

```javascript
import { WebSocketServer, WebSocket } from 'ws';
import { createServer } from 'http';

const server = createServer();
const wss = new WebSocketServer({ server });

wss.on('connection', (ws, req) => {
  const clientIp = req.headers['x-forwarded-for'] || req.socket.remoteAddress;
  console.log(`New connection from ${clientIp}`);

  ws.isAlive = true;
  ws.on('pong', () => { ws.isAlive = true; });

  ws.on('message', (data, isBinary) => {
    const message = isBinary ? data : data.toString();

    // Broadcast to all connected clients
    wss.clients.forEach((client) => {
      if (client !== ws && client.readyState === WebSocket.OPEN) {
        client.send(message, { binary: isBinary });
      }
    });
  });

  ws.on('close', (code, reason) => {
    console.log(`Disconnected: ${code} ${reason}`);
  });

  ws.on('error', (err) => {
    console.error('Socket error:', err.message);
  });
});

server.listen(8080);
```

### Anti-Patterns

- **Sending unserialized objects** — always `JSON.stringify()` before sending and validate after parsing
- **Ignoring readyState** — check `ws.readyState === WebSocket.OPEN` before calling `send()`
- **No message framing** — always wrap messages in a typed envelope: `{ type, payload, id }`
- **Blocking the event loop** — offload heavy message processing to worker threads

---

## 2. Socket.io

### When to Use

- You need automatic reconnection, multiplexing, and fallback transports out of the box
- The project already uses Node.js on the server
- You want rooms and namespaces without building them yourself

### Server Setup

```javascript
import { createServer } from 'http';
import { Server } from 'socket.io';

const httpServer = createServer();
const io = new Server(httpServer, {
  cors: {
    origin: ['https://app.example.com'],
    methods: ['GET', 'POST'],
    credentials: true,
  },
  pingInterval: 25000,
  pingTimeout: 20000,
  maxHttpBufferSize: 1e6, // 1 MB
  transports: ['websocket', 'polling'],
});

httpServer.listen(3000);
```

### Namespaces

Namespaces partition the communication channel at the server level. Each namespace has its own set of event handlers, rooms, and middleware.

```javascript
// Admin namespace with authentication middleware
const adminNamespace = io.of('/admin');

adminNamespace.use(async (socket, next) => {
  const token = socket.handshake.auth.token;
  try {
    const user = await verifyAdminToken(token);
    socket.data.user = user;
    next();
  } catch (err) {
    next(new Error('Unauthorized'));
  }
});

adminNamespace.on('connection', (socket) => {
  console.log(`Admin connected: ${socket.data.user.email}`);

  socket.on('dashboard:metrics', async (callback) => {
    const metrics = await getSystemMetrics();
    callback({ status: 'ok', data: metrics });
  });
});
```

### Rooms

```javascript
io.on('connection', (socket) => {
  // Join a room based on user's organization
  const orgId = socket.data.user.orgId;
  socket.join(`org:${orgId}`);

  // Join a specific project channel
  socket.on('project:join', (projectId) => {
    socket.join(`project:${projectId}`);
    socket.to(`project:${projectId}`).emit('user:joined', {
      userId: socket.data.user.id,
      name: socket.data.user.name,
    });
  });

  // Leave a project channel
  socket.on('project:leave', (projectId) => {
    socket.leave(`project:${projectId}`);
    socket.to(`project:${projectId}`).emit('user:left', {
      userId: socket.data.user.id,
    });
  });

  // Send to everyone in the room except sender
  socket.on('project:update', (projectId, update) => {
    socket.to(`project:${projectId}`).emit('project:updated', update);
  });

  // Send to everyone in the room including sender
  socket.on('announcement', (orgId, message) => {
    io.to(`org:${orgId}`).emit('announcement', message);
  });
});
```

### Acknowledgements

Acknowledgements provide request-response semantics over the event-driven protocol.

```javascript
// Server
socket.on('message:send', async (payload, callback) => {
  try {
    const message = await saveMessage(payload);
    callback({ status: 'ok', messageId: message.id, timestamp: message.createdAt });
  } catch (err) {
    callback({ status: 'error', message: err.message });
  }
});

// Client
socket.emit('message:send', { text: 'Hello', roomId: '123' }, (response) => {
  if (response.status === 'ok') {
    appendMessage({ id: response.messageId, text: 'Hello', ts: response.timestamp });
  } else {
    showError(response.message);
  }
});
```

### Middleware

```javascript
// Connection-level middleware (runs once per connection)
io.use((socket, next) => {
  const token = socket.handshake.auth.token;
  if (!token) return next(new Error('Authentication required'));

  try {
    socket.data.user = verifyToken(token);
    next();
  } catch {
    next(new Error('Invalid token'));
  }
});

// Event-level middleware (runs on every incoming event)
io.on('connection', (socket) => {
  socket.use(([event, ...args], next) => {
    logEvent(socket.data.user.id, event, args);
    next();
  });
});
```

### Binary Events

```javascript
// Sending a file from client
const fileInput = document.getElementById('file');
fileInput.addEventListener('change', () => {
  const file = fileInput.files[0];
  socket.emit('file:upload', {
    name: file.name,
    type: file.type,
    size: file.size,
  }, file, (response) => {
    console.log('Upload complete:', response.url);
  });
});

// Receiving on server
socket.on('file:upload', async (metadata, fileBuffer, callback) => {
  if (metadata.size > 10 * 1024 * 1024) {
    return callback({ status: 'error', message: 'File too large' });
  }
  const url = await uploadToStorage(metadata, fileBuffer);
  callback({ status: 'ok', url });
});
```

### Anti-Patterns

- **Using the default namespace for everything** — segment by concern with named namespaces
- **Storing state on the socket object directly** — use `socket.data` which is the designated container
- **Missing error handler on client** — always listen for `connect_error` to catch auth failures
- **Not setting `maxHttpBufferSize`** — unbounded message size is a denial-of-service vector
- **Relying on connection order** — messages may arrive out of order after reconnection

---

## 3. Server-Sent Events

### When to Use

- The communication is unidirectional: server pushes data to the client
- You want automatic reconnection built into the browser API
- Use cases: live feeds, notifications, progress updates, log streaming

### SSE vs WebSocket Decision Matrix

| Criteria | SSE | WebSocket |
|----------|-----|-----------|
| Direction | Server to client only | Bidirectional |
| Protocol | HTTP/1.1 or HTTP/2 | Custom framed protocol |
| Reconnection | Built-in with `retry` | Must implement manually |
| Binary data | Not supported | Supported |
| Browser support | All modern browsers | All modern browsers |
| Through proxies | Works naturally (HTTP) | May require configuration |
| Connection limit | 6 per domain (HTTP/1.1) | No hard browser limit |
| Multiplexing | Via HTTP/2 | Via namespaces/rooms |

**Choose SSE when** you only need server-to-client push and want the simplest possible implementation. **Choose WebSocket when** you need bidirectional communication or binary data.

### Server Implementation (Node.js)

```javascript
import express from 'express';

const app = express();
const clients = new Map();

app.get('/events', (req, res) => {
  const clientId = crypto.randomUUID();

  res.writeHead(200, {
    'Content-Type': 'text/event-stream',
    'Cache-Control': 'no-cache',
    'Connection': 'keep-alive',
    'X-Accel-Buffering': 'no', // Disable nginx buffering
  });

  // Send retry interval (milliseconds)
  res.write('retry: 5000\n\n');

  // Send initial connection event
  res.write(`event: connected\ndata: ${JSON.stringify({ clientId })}\n\n`);

  clients.set(clientId, res);

  req.on('close', () => {
    clients.delete(clientId);
  });
});

// Broadcast to all connected SSE clients
function broadcast(event, data) {
  const payload = `event: ${event}\ndata: ${JSON.stringify(data)}\n\n`;
  clients.forEach((res) => {
    res.write(payload);
  });
}

// Send to a specific client
function sendTo(clientId, event, data) {
  const res = clients.get(clientId);
  if (res) {
    res.write(`event: ${event}\nid: ${Date.now()}\ndata: ${JSON.stringify(data)}\n\n`);
  }
}
```

### Client Implementation

```javascript
const eventSource = new EventSource('/events', {
  withCredentials: true, // Send cookies for authentication
});

eventSource.addEventListener('connected', (e) => {
  const { clientId } = JSON.parse(e.data);
  console.log('SSE connected:', clientId);
});

eventSource.addEventListener('notification', (e) => {
  const data = JSON.parse(e.data);
  showNotification(data.title, data.body);
});

// Default message event (no named event)
eventSource.onmessage = (e) => {
  console.log('Message:', e.data);
};

eventSource.onerror = (e) => {
  if (eventSource.readyState === EventSource.CONNECTING) {
    console.log('Reconnecting...');
  } else {
    console.error('SSE connection failed');
    eventSource.close();
  }
};
```

### Resuming from Last Event ID

The browser automatically sends the `Last-Event-ID` header when reconnecting, allowing the server to replay missed events.

```javascript
app.get('/events', (req, res) => {
  const lastEventId = req.headers['last-event-id'];

  // Set up SSE headers...

  if (lastEventId) {
    // Replay missed events from persistent store
    const missedEvents = await getEventsSince(lastEventId);
    for (const event of missedEvents) {
      res.write(`event: ${event.type}\nid: ${event.id}\ndata: ${JSON.stringify(event.data)}\n\n`);
    }
  }
});
```

### Anti-Patterns

- **Not setting `X-Accel-Buffering: no`** — reverse proxies like nginx buffer responses by default, breaking SSE
- **Using SSE for bidirectional communication** — pair SSE with regular POST requests if the client needs to send data
- **Forgetting connection limits** — HTTP/1.1 allows only 6 connections per domain; use HTTP/2 or a dedicated subdomain
- **Not including event IDs** — without IDs the `Last-Event-ID` header is empty and replay is impossible
- **Sending comments as keepalive** — use `:keepalive\n\n` comment lines to prevent proxy timeouts

---

## 4. Connection Management

### When to Use

- Every production WebSocket deployment requires proper connection lifecycle management
- Systems where stale connections consume resources and cause phantom users
- Applications behind load balancers that enforce idle timeouts

### Heartbeat / Ping-Pong

```javascript
import { WebSocketServer } from 'ws';

const wss = new WebSocketServer({ port: 8080 });

// Server-initiated heartbeat
const HEARTBEAT_INTERVAL = 30000;
const CLIENT_TIMEOUT = 35000;

wss.on('connection', (ws) => {
  ws.isAlive = true;

  ws.on('pong', () => {
    ws.isAlive = true;
  });

  ws.on('error', () => {
    ws.isAlive = false;
  });
});

const heartbeat = setInterval(() => {
  wss.clients.forEach((ws) => {
    if (!ws.isAlive) {
      console.log('Terminating stale connection');
      return ws.terminate();
    }
    ws.isAlive = false;
    ws.ping();
  });
}, HEARTBEAT_INTERVAL);

wss.on('close', () => {
  clearInterval(heartbeat);
});
```

### Application-Level Heartbeat

When the WebSocket connection traverses intermediaries that strip control frames, use application-level heartbeats.

```javascript
// Client
let heartbeatTimer;

function startHeartbeat(ws) {
  heartbeatTimer = setInterval(() => {
    if (ws.readyState === WebSocket.OPEN) {
      ws.send(JSON.stringify({ type: 'ping', ts: Date.now() }));
    }
  }, 25000);
}

function stopHeartbeat() {
  clearInterval(heartbeatTimer);
}

// Server
ws.on('message', (raw) => {
  const msg = JSON.parse(raw);
  if (msg.type === 'ping') {
    ws.send(JSON.stringify({ type: 'pong', ts: Date.now() }));
    return;
  }
  // Handle other messages...
});
```

### Connection Pooling

For services that open multiple WebSocket connections to upstream servers, pool and reuse connections.

```javascript
class WebSocketPool {
  constructor(url, { maxConnections = 10, idleTimeout = 60000 } = {}) {
    this.url = url;
    this.maxConnections = maxConnections;
    this.idleTimeout = idleTimeout;
    this.pool = [];
    this.waiting = [];
  }

  async acquire() {
    // Reuse idle connection
    const idle = this.pool.find((c) => c.active === false && c.ws.readyState === WebSocket.OPEN);
    if (idle) {
      idle.active = true;
      idle.lastUsed = Date.now();
      return idle.ws;
    }

    // Create new connection if pool not full
    if (this.pool.length < this.maxConnections) {
      const ws = await this._connect();
      const entry = { ws, active: true, lastUsed: Date.now() };
      this.pool.push(entry);
      return ws;
    }

    // Wait for a connection to become available
    return new Promise((resolve) => {
      this.waiting.push(resolve);
    });
  }

  release(ws) {
    const entry = this.pool.find((c) => c.ws === ws);
    if (!entry) return;

    if (this.waiting.length > 0) {
      const resolve = this.waiting.shift();
      entry.lastUsed = Date.now();
      resolve(entry.ws);
    } else {
      entry.active = false;
      entry.lastUsed = Date.now();
    }
  }

  _connect() {
    return new Promise((resolve, reject) => {
      const ws = new WebSocket(this.url);
      ws.once('open', () => resolve(ws));
      ws.once('error', reject);
    });
  }

  cleanup() {
    const now = Date.now();
    this.pool = this.pool.filter((entry) => {
      if (!entry.active && now - entry.lastUsed > this.idleTimeout) {
        entry.ws.close();
        return false;
      }
      return true;
    });
  }
}
```

### Graceful Shutdown

```javascript
async function gracefulShutdown(wss, server) {
  console.log('Initiating graceful shutdown...');

  // Stop accepting new connections
  server.close();

  // Notify all clients
  const closePromises = [];
  wss.clients.forEach((ws) => {
    if (ws.readyState === WebSocket.OPEN) {
      ws.send(JSON.stringify({ type: 'server:shutdown', reconnectAfter: 5000 }));
      closePromises.push(
        new Promise((resolve) => {
          ws.close(1001, 'Server shutting down');
          ws.on('close', resolve);
          // Force terminate after timeout
          setTimeout(() => {
            ws.terminate();
            resolve();
          }, 5000);
        })
      );
    }
  });

  await Promise.allSettled(closePromises);
  console.log('All connections closed');
}

process.on('SIGTERM', () => gracefulShutdown(wss, server));
process.on('SIGINT', () => gracefulShutdown(wss, server));
```

### Authentication on Connect

```javascript
import { WebSocketServer } from 'ws';
import jwt from 'jsonwebtoken';

const wss = new WebSocketServer({
  noServer: true, // Handle upgrade manually
});

server.on('upgrade', async (request, socket, head) => {
  try {
    // Extract token from query string or cookie
    const url = new URL(request.url, `http://${request.headers.host}`);
    const token = url.searchParams.get('token') || parseCookie(request.headers.cookie)?.token;

    if (!token) {
      socket.write('HTTP/1.1 401 Unauthorized\r\n\r\n');
      socket.destroy();
      return;
    }

    const user = jwt.verify(token, process.env.JWT_SECRET);

    wss.handleUpgrade(request, socket, head, (ws) => {
      ws.user = user;
      wss.emit('connection', ws, request);
    });
  } catch (err) {
    socket.write('HTTP/1.1 403 Forbidden\r\n\r\n');
    socket.destroy();
  }
});
```

### Anti-Patterns

- **No heartbeat** — connections silently go stale behind NATs and load balancers; always implement ping/pong
- **Hard terminate without notification** — send a close frame with a reason before terminating
- **Auth token in URL path** — tokens in URLs are logged by proxies; use query params with short-lived tokens or the first message for auth
- **No connection limits** — always cap max connections per IP and total to prevent resource exhaustion

---

## 5. Reconnection Strategies

### When to Use

- Every client-side WebSocket implementation in production
- Mobile applications where network transitions are frequent
- Systems where message delivery guarantees are required

### Exponential Backoff with Jitter

```javascript
class ReconnectingWebSocket {
  constructor(url, options = {}) {
    this.url = url;
    this.options = {
      initialDelay: options.initialDelay ?? 1000,
      maxDelay: options.maxDelay ?? 30000,
      factor: options.factor ?? 2,
      jitter: options.jitter ?? true,
      maxRetries: options.maxRetries ?? Infinity,
    };
    this.retryCount = 0;
    this.handlers = new Map();
    this.messageQueue = [];
    this.lastEventId = null;
    this.connect();
  }

  connect() {
    const url = new URL(this.url);
    if (this.lastEventId) {
      url.searchParams.set('lastEventId', this.lastEventId);
    }

    this.ws = new WebSocket(url.toString());

    this.ws.addEventListener('open', () => {
      console.log('Connected');
      this.retryCount = 0;
      this.flushQueue();
      this.emit('open');
    });

    this.ws.addEventListener('message', (event) => {
      const data = JSON.parse(event.data);
      if (data.id) {
        this.lastEventId = data.id;
      }
      this.emit('message', data);
    });

    this.ws.addEventListener('close', (event) => {
      this.emit('close', event);
      if (event.code !== 1000) {
        this.scheduleReconnect();
      }
    });

    this.ws.addEventListener('error', () => {
      // The close event will fire after this
    });
  }

  scheduleReconnect() {
    if (this.retryCount >= this.options.maxRetries) {
      this.emit('maxRetriesReached');
      return;
    }

    const delay = this.calculateDelay();
    console.log(`Reconnecting in ${delay}ms (attempt ${this.retryCount + 1})`);

    setTimeout(() => {
      this.retryCount++;
      this.connect();
    }, delay);
  }

  calculateDelay() {
    const base = Math.min(
      this.options.initialDelay * Math.pow(this.options.factor, this.retryCount),
      this.options.maxDelay
    );

    if (this.options.jitter) {
      // Full jitter: random value between 0 and the calculated delay
      return Math.random() * base;
    }

    return base;
  }

  send(data) {
    const message = typeof data === 'string' ? data : JSON.stringify(data);
    if (this.ws.readyState === WebSocket.OPEN) {
      this.ws.send(message);
    } else {
      this.messageQueue.push(message);
    }
  }

  flushQueue() {
    while (this.messageQueue.length > 0 && this.ws.readyState === WebSocket.OPEN) {
      this.ws.send(this.messageQueue.shift());
    }
  }

  on(event, handler) {
    if (!this.handlers.has(event)) this.handlers.set(event, []);
    this.handlers.get(event).push(handler);
  }

  emit(event, ...args) {
    (this.handlers.get(event) || []).forEach((h) => h(...args));
  }

  close() {
    this.options.maxRetries = 0; // Prevent reconnection
    this.ws.close(1000, 'Client closed');
  }
}
```

### State Recovery After Reconnection

```javascript
class StatefulConnection extends ReconnectingWebSocket {
  constructor(url, options) {
    super(url, options);
    this.subscriptions = new Set();

    this.on('open', () => {
      this.resubscribe();
      this.requestMissedMessages();
    });
  }

  subscribe(channel) {
    this.subscriptions.add(channel);
    this.send({ type: 'subscribe', channel });
  }

  unsubscribe(channel) {
    this.subscriptions.delete(channel);
    this.send({ type: 'unsubscribe', channel });
  }

  resubscribe() {
    for (const channel of this.subscriptions) {
      this.send({ type: 'subscribe', channel });
    }
  }

  requestMissedMessages() {
    if (this.lastEventId) {
      this.send({
        type: 'replay',
        since: this.lastEventId,
      });
    }
  }
}
```

### Missed Message Handling on the Server

```javascript
import Redis from 'ioredis';

const redis = new Redis();
const MESSAGE_TTL = 3600; // 1 hour
const MAX_REPLAY = 1000;

async function storeMessage(channel, message) {
  const id = `${Date.now()}-${crypto.randomUUID()}`;
  const entry = JSON.stringify({ id, channel, ...message, ts: Date.now() });

  await redis
    .pipeline()
    .zadd(`messages:${channel}`, Date.now(), entry)
    .zremrangebyrank(`messages:${channel}`, 0, -(MAX_REPLAY + 1))
    .expire(`messages:${channel}`, MESSAGE_TTL)
    .exec();

  return id;
}

async function getMessagesSince(channel, sinceId) {
  const members = await redis.zrangebyscore(`messages:${channel}`, '-inf', '+inf');
  const messages = members.map((m) => JSON.parse(m));

  const idx = messages.findIndex((m) => m.id === sinceId);
  if (idx === -1) {
    // Client is too far behind; send full snapshot instead
    return { type: 'snapshot', messages: messages.slice(-100) };
  }

  return { type: 'replay', messages: messages.slice(idx + 1) };
}
```

### Anti-Patterns

- **Fixed delay reconnection** — causes thundering herd when many clients reconnect simultaneously
- **No jitter** — even with exponential backoff, synchronized retries overload the server
- **Infinite queue without bounds** — cap `messageQueue` length and drop oldest messages when full
- **No maximum retry limit** — provide an escape hatch after sustained failures
- **Reconnecting on code 1000** — code 1000 means normal closure; do not reconnect

---

## 6. Scaling

### When to Use

- The application needs more than one WebSocket server process or node
- User counts exceed what a single server can handle (typically 50K-100K concurrent)
- You need high availability with zero-downtime deployments

### Redis Pub/Sub Adapter

When running multiple server instances, messages emitted on one server must reach clients connected to other servers. Redis pub/sub acts as the message bus.

```javascript
import { Server } from 'socket.io';
import { createAdapter } from '@socket.io/redis-adapter';
import { createClient } from 'redis';

async function createScalableServer(httpServer) {
  const pubClient = createClient({ url: process.env.REDIS_URL });
  const subClient = pubClient.duplicate();

  await Promise.all([pubClient.connect(), subClient.connect()]);

  const io = new Server(httpServer, {
    adapter: createAdapter(pubClient, subClient),
    transports: ['websocket'],
  });

  // Graceful shutdown: disconnect Redis clients
  process.on('SIGTERM', async () => {
    await pubClient.quit();
    await subClient.quit();
  });

  return io;
}
```

### Redis Streams for Ordered Delivery

When message ordering matters, use Redis Streams instead of pub/sub.

```javascript
import Redis from 'ioredis';

const redis = new Redis();

// Producer: add message to stream
async function publishOrdered(channel, message) {
  const id = await redis.xadd(
    `stream:${channel}`,
    'MAXLEN', '~', '10000', // Approximate cap at 10,000 entries
    '*', // Auto-generate ID
    'data', JSON.stringify(message)
  );
  return id;
}

// Consumer: read messages from stream
async function consumeStream(channel, lastId = '$') {
  while (true) {
    const results = await redis.xread(
      'BLOCK', 5000, // Block for 5 seconds
      'COUNT', 100,
      'STREAMS', `stream:${channel}`, lastId
    );

    if (!results) continue;

    for (const [, entries] of results) {
      for (const [id, fields] of entries) {
        const data = JSON.parse(fields[1]); // fields = ['data', '...']
        await handleMessage(data);
        lastId = id;
      }
    }
  }
}
```

### Sticky Sessions

WebSocket connections are stateful. When using Socket.io with HTTP long-polling fallback, multiple HTTP requests from the same client must reach the same server.

**Nginx configuration:**

```nginx
upstream websocket_servers {
    ip_hash;  # Sticky sessions by client IP
    server app1:3000;
    server app2:3000;
    server app3:3000;
}

server {
    listen 80;

    location /socket.io/ {
        proxy_pass http://websocket_servers;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_read_timeout 86400s;
        proxy_send_timeout 86400s;
    }
}
```

**Cookie-based sticky sessions (preferred over ip_hash):**

```nginx
upstream websocket_servers {
    server app1:3000;
    server app2:3000;
    server app3:3000;

    sticky cookie srv_id expires=1h domain=.example.com path=/;
}
```

### Horizontal Scaling Architecture

```
                    ┌─────────────┐
    Clients ────>   │  Load       │
                    │  Balancer   │
                    │  (nginx)    │
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
        ┌─────┴─────┐ ┌───┴───┐ ┌─────┴─────┐
        │  WS Node  │ │  WS   │ │  WS Node  │
        │  1        │ │ Node 2│ │  3        │
        └─────┬─────┘ └───┬───┘ └─────┬─────┘
              │            │            │
              └────────────┼────────────┘
                           │
                    ┌──────┴──────┐
                    │   Redis     │
                    │   Cluster   │
                    └─────────────┘
```

### Connection Count Monitoring

```javascript
import { register, Gauge, Counter } from 'prom-client';

const activeConnections = new Gauge({
  name: 'ws_active_connections',
  help: 'Number of active WebSocket connections',
  labelNames: ['namespace'],
});

const totalMessages = new Counter({
  name: 'ws_messages_total',
  help: 'Total WebSocket messages processed',
  labelNames: ['namespace', 'event', 'direction'],
});

io.on('connection', (socket) => {
  activeConnections.inc({ namespace: '/' });

  socket.use(([event], next) => {
    totalMessages.inc({ namespace: '/', event, direction: 'inbound' });
    next();
  });

  socket.onAny(() => {
    // Tracked per-event above
  });

  socket.on('disconnect', () => {
    activeConnections.dec({ namespace: '/' });
  });
});

// Expose metrics endpoint for Prometheus
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});
```

### Anti-Patterns

- **No pub/sub adapter** — emitting events only reaches clients on the local process
- **Using ip_hash with CDN** — all clients behind a CDN share one IP; use cookie-based affinity
- **No connection limits per node** — set `maxConnections` and return 503 when at capacity
- **Ignoring message ordering** — pub/sub does not guarantee order; use Redis Streams when ordering matters
- **Single Redis instance** — use Redis Cluster or Sentinel for high availability

---

## 7. Security

### When to Use

- Every WebSocket deployment exposed to the internet
- Applications handling sensitive data or user actions
- Systems subject to compliance requirements

### Origin Validation

```javascript
const wss = new WebSocketServer({
  noServer: true,
});

const ALLOWED_ORIGINS = new Set([
  'https://app.example.com',
  'https://staging.example.com',
]);

server.on('upgrade', (request, socket, head) => {
  const origin = request.headers.origin;

  if (!origin || !ALLOWED_ORIGINS.has(origin)) {
    socket.write('HTTP/1.1 403 Forbidden\r\n\r\n');
    socket.destroy();
    return;
  }

  wss.handleUpgrade(request, socket, head, (ws) => {
    wss.emit('connection', ws, request);
  });
});
```

### Rate Limiting

```javascript
class RateLimiter {
  constructor({ windowMs = 60000, maxRequests = 100 } = {}) {
    this.windowMs = windowMs;
    this.maxRequests = maxRequests;
    this.clients = new Map();
  }

  consume(clientId) {
    const now = Date.now();
    let record = this.clients.get(clientId);

    if (!record || now - record.windowStart > this.windowMs) {
      record = { windowStart: now, count: 0 };
      this.clients.set(clientId, record);
    }

    record.count++;

    if (record.count > this.maxRequests) {
      return { allowed: false, retryAfter: record.windowStart + this.windowMs - now };
    }

    return { allowed: true, remaining: this.maxRequests - record.count };
  }

  cleanup() {
    const now = Date.now();
    for (const [id, record] of this.clients) {
      if (now - record.windowStart > this.windowMs) {
        this.clients.delete(id);
      }
    }
  }
}

const limiter = new RateLimiter({ windowMs: 60000, maxRequests: 120 });

// Clean up stale entries periodically
setInterval(() => limiter.cleanup(), 60000);

ws.on('message', (raw) => {
  const result = limiter.consume(ws.user.id);
  if (!result.allowed) {
    ws.send(JSON.stringify({
      type: 'error',
      code: 'RATE_LIMITED',
      retryAfter: result.retryAfter,
    }));
    return;
  }
  // Process message...
});
```

### Message Validation

```javascript
import { z } from 'zod';

const MessageSchema = z.discriminatedUnion('type', [
  z.object({
    type: z.literal('chat:message'),
    roomId: z.string().uuid(),
    text: z.string().min(1).max(4096),
  }),
  z.object({
    type: z.literal('chat:typing'),
    roomId: z.string().uuid(),
    isTyping: z.boolean(),
  }),
  z.object({
    type: z.literal('presence:update'),
    status: z.enum(['online', 'away', 'dnd']),
  }),
]);

function handleMessage(ws, raw) {
  let parsed;
  try {
    parsed = JSON.parse(raw);
  } catch {
    ws.send(JSON.stringify({ type: 'error', code: 'INVALID_JSON' }));
    return;
  }

  const result = MessageSchema.safeParse(parsed);
  if (!result.success) {
    ws.send(JSON.stringify({
      type: 'error',
      code: 'VALIDATION_ERROR',
      details: result.error.flatten(),
    }));
    return;
  }

  routeMessage(ws, result.data);
}
```

### Token Refresh Over WebSocket

```javascript
// Client-side token refresh
class AuthenticatedSocket {
  constructor(url, getToken) {
    this.url = url;
    this.getToken = getToken;
    this.connect();
  }

  async connect() {
    const token = await this.getToken();
    this.ws = new WebSocket(`${this.url}?token=${token}`);

    this.ws.addEventListener('message', (event) => {
      const data = JSON.parse(event.data);

      if (data.type === 'auth:expiring') {
        this.refreshToken();
      }
    });
  }

  async refreshToken() {
    const newToken = await this.getToken(); // Fetches fresh token
    this.ws.send(JSON.stringify({
      type: 'auth:refresh',
      token: newToken,
    }));
  }
}

// Server-side token refresh handling
ws.on('message', (raw) => {
  const msg = JSON.parse(raw);

  if (msg.type === 'auth:refresh') {
    try {
      const user = jwt.verify(msg.token, process.env.JWT_SECRET);
      ws.user = user;
      ws.tokenExpiry = user.exp * 1000;
      ws.send(JSON.stringify({ type: 'auth:refreshed' }));
    } catch {
      ws.close(4001, 'Invalid token');
    }
    return;
  }

  // Check token expiry on every message
  if (Date.now() > ws.tokenExpiry) {
    ws.send(JSON.stringify({ type: 'auth:expiring' }));
  }
});
```

### WSS / TLS Configuration

Always use `wss://` in production. Terminate TLS at the load balancer or reverse proxy.

```nginx
server {
    listen 443 ssl http2;
    server_name ws.example.com;

    ssl_certificate /etc/letsencrypt/live/ws.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/ws.example.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
    ssl_prefer_server_ciphers off;

    location /ws {
        proxy_pass http://websocket_backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header X-Real-IP $remote_addr;
        proxy_read_timeout 3600s;
    }
}
```

### Anti-Patterns

- **No origin check** — any page can open a WebSocket to your server; always validate `Origin`
- **Long-lived tokens without refresh** — JWTs used at connection time can outlive their expiry; implement in-band refresh
- **Trusting client messages** — validate every message with a schema; never pass raw input to queries or commands
- **Plain `ws://` in production** — always use `wss://` for encrypted transport
- **No rate limiting** — a single client can flood the server; enforce per-connection and per-IP limits
- **Logging message content** — avoid logging user data; log event types and metadata only

---

## 8. Testing

### When to Use

- Before deploying any WebSocket feature to production
- When building integration tests for real-time features
- For capacity planning and performance benchmarking

### Unit Testing with Vitest

```javascript
import { describe, it, expect, beforeEach, afterEach, vi } from 'vitest';
import { WebSocketServer } from 'ws';
import { createServer } from 'http';

describe('WebSocket message handler', () => {
  let server;
  let wss;
  let port;

  beforeEach(async () => {
    server = createServer();
    wss = new WebSocketServer({ server });
    await new Promise((resolve) => {
      server.listen(0, () => {
        port = server.address().port;
        resolve();
      });
    });
  });

  afterEach(async () => {
    wss.clients.forEach((ws) => ws.terminate());
    await new Promise((resolve) => server.close(resolve));
  });

  it('echoes messages back to sender', async () => {
    wss.on('connection', (ws) => {
      ws.on('message', (data) => {
        ws.send(data.toString());
      });
    });

    const ws = new WebSocket(`ws://localhost:${port}`);
    await new Promise((resolve) => ws.on('open', resolve));

    const response = new Promise((resolve) => {
      ws.on('message', (data) => resolve(data.toString()));
    });

    ws.send('hello');
    expect(await response).toBe('hello');
    ws.close();
  });

  it('broadcasts to all clients except sender', async () => {
    wss.on('connection', (ws) => {
      ws.on('message', (data) => {
        wss.clients.forEach((client) => {
          if (client !== ws && client.readyState === 1) {
            client.send(data.toString());
          }
        });
      });
    });

    const ws1 = new WebSocket(`ws://localhost:${port}`);
    const ws2 = new WebSocket(`ws://localhost:${port}`);
    await Promise.all([
      new Promise((r) => ws1.on('open', r)),
      new Promise((r) => ws2.on('open', r)),
    ]);

    const ws2Message = new Promise((resolve) => {
      ws2.on('message', (data) => resolve(data.toString()));
    });

    ws1.send('broadcast test');
    expect(await ws2Message).toBe('broadcast test');

    ws1.close();
    ws2.close();
  });

  it('rejects connections without valid auth token', async () => {
    // Assuming auth middleware is in place
    const ws = new WebSocket(`ws://localhost:${port}`);

    const closeEvent = new Promise((resolve) => {
      ws.on('close', (code, reason) => resolve({ code, reason: reason.toString() }));
      ws.on('error', () => {}); // Suppress error
    });

    const result = await closeEvent;
    // Depending on implementation, may get 1006 (abnormal) or custom code
    expect(result.code).toBeDefined();
  });
});
```

### Testing Socket.io with Jest

```javascript
import { createServer } from 'http';
import { Server } from 'socket.io';
import { io as Client } from 'socket.io-client';

describe('Socket.io chat', () => {
  let httpServer, ioServer, clientSocket;

  beforeAll((done) => {
    httpServer = createServer();
    ioServer = new Server(httpServer);
    httpServer.listen(() => {
      const port = httpServer.address().port;
      clientSocket = Client(`http://localhost:${port}`);
      clientSocket.on('connect', done);
    });

    // Register handlers
    ioServer.on('connection', (socket) => {
      socket.on('chat:message', (msg, callback) => {
        socket.broadcast.emit('chat:message', {
          ...msg,
          id: 'msg-123',
          timestamp: Date.now(),
        });
        callback({ status: 'ok', id: 'msg-123' });
      });

      socket.on('room:join', (roomId) => {
        socket.join(roomId);
        socket.emit('room:joined', { roomId });
      });
    });
  });

  afterAll(() => {
    ioServer.close();
    clientSocket.disconnect();
  });

  it('sends message with acknowledgement', (done) => {
    clientSocket.emit(
      'chat:message',
      { text: 'Hello', roomId: 'room-1' },
      (response) => {
        expect(response.status).toBe('ok');
        expect(response.id).toBe('msg-123');
        done();
      }
    );
  });

  it('joins a room and receives confirmation', (done) => {
    clientSocket.on('room:joined', (data) => {
      expect(data.roomId).toBe('room-42');
      done();
    });
    clientSocket.emit('room:join', 'room-42');
  });
});
```

### Load Testing with Artillery

```yaml
# artillery-ws-config.yml
config:
  target: "wss://ws.example.com"
  phases:
    - duration: 60
      arrivalRate: 10
      name: "Warm up"
    - duration: 120
      arrivalRate: 50
      name: "Sustained load"
    - duration: 60
      arrivalRate: 100
      name: "Peak load"
  ws:
    rejectUnauthorized: false
  variables:
    channels:
      - "general"
      - "trading"
      - "notifications"

scenarios:
  - name: "Chat user"
    engine: ws
    flow:
      - send:
          payload: '{"type":"auth","token":"{{ $env.TEST_TOKEN }}"}'
      - think: 1
      - send:
          payload: '{"type":"subscribe","channel":"{{ channels }}"}'
      - think: 2
      - loop:
          - send:
              payload: '{"type":"chat:message","text":"Load test {{ $randomString(20) }}","channel":"{{ channels }}"}'
          - think: 3
        count: 10
      - send:
          payload: '{"type":"unsubscribe","channel":"{{ channels }}"}'
      - think: 1
```

Run: `artillery run artillery-ws-config.yml`

### Load Testing with k6

```javascript
// k6-ws-test.js
import ws from 'k6/ws';
import { check, sleep } from 'k6';
import { Counter, Trend } from 'k6/metrics';

const messagesSent = new Counter('ws_messages_sent');
const messagesReceived = new Counter('ws_messages_received');
const messageLatency = new Trend('ws_message_latency');

export const options = {
  stages: [
    { duration: '30s', target: 100 },
    { duration: '1m', target: 500 },
    { duration: '30s', target: 1000 },
    { duration: '1m', target: 1000 },
    { duration: '30s', target: 0 },
  ],
};

export default function () {
  const url = 'wss://ws.example.com/ws';

  const res = ws.connect(url, {}, function (socket) {
    socket.on('open', () => {
      socket.send(JSON.stringify({ type: 'auth', token: __ENV.TEST_TOKEN }));

      socket.setInterval(() => {
        const start = Date.now();
        socket.send(JSON.stringify({
          type: 'ping',
          ts: start,
        }));
        messagesSent.add(1);
      }, 2000);
    });

    socket.on('message', (data) => {
      messagesReceived.add(1);
      const msg = JSON.parse(data);
      if (msg.type === 'pong' && msg.ts) {
        messageLatency.add(Date.now() - msg.ts);
      }
    });

    socket.on('close', () => {});

    socket.setTimeout(() => {
      socket.close();
    }, 60000);
  });

  check(res, {
    'Connected successfully': (r) => r && r.status === 101,
  });

  sleep(1);
}
```

Run: `k6 run --env TEST_TOKEN=xyz k6-ws-test.js`

### Integration Testing: Connection Lifecycle

```javascript
import { describe, it, expect } from 'vitest';

describe('Connection lifecycle', () => {
  it('handles reconnection and state recovery', async () => {
    const ws1 = new WebSocket(`ws://localhost:${port}?token=${testToken}`);
    await waitForOpen(ws1);

    // Subscribe to a channel
    ws1.send(JSON.stringify({ type: 'subscribe', channel: 'updates' }));
    await waitForMessage(ws1, (m) => m.type === 'subscribed');

    // Simulate server-sent messages
    broadcastToChannel('updates', { type: 'data', value: 1 });
    broadcastToChannel('updates', { type: 'data', value: 2 });

    const msg1 = await waitForMessage(ws1, (m) => m.value === 1);
    const msg2 = await waitForMessage(ws1, (m) => m.value === 2);
    const lastId = msg2.id;

    // Simulate disconnect
    ws1.close();

    // Reconnect with last known event ID
    const ws2 = new WebSocket(
      `ws://localhost:${port}?token=${testToken}&lastEventId=${lastId}`
    );
    await waitForOpen(ws2);

    // Resubscribe
    ws2.send(JSON.stringify({ type: 'subscribe', channel: 'updates' }));

    // Messages sent during disconnect should be replayed
    broadcastToChannel('updates', { type: 'data', value: 3 });

    const msg3 = await waitForMessage(ws2, (m) => m.value === 3);
    expect(msg3.value).toBe(3);

    ws2.close();
  });
});

// Test helpers
function waitForOpen(ws) {
  return new Promise((resolve, reject) => {
    ws.on('open', resolve);
    ws.on('error', reject);
  });
}

function waitForMessage(ws, predicate, timeout = 5000) {
  return new Promise((resolve, reject) => {
    const timer = setTimeout(() => reject(new Error('Timeout')), timeout);
    const handler = (raw) => {
      const msg = JSON.parse(raw.toString());
      if (predicate(msg)) {
        clearTimeout(timer);
        ws.off('message', handler);
        resolve(msg);
      }
    };
    ws.on('message', handler);
  });
}
```

### Anti-Patterns

- **No timeout on test assertions** — WebSocket tests can hang indefinitely; always set assertion timeouts
- **Not cleaning up connections** — leaked WebSocket connections cause port exhaustion and flaky tests
- **Testing only the happy path** — test disconnection, reconnection, invalid messages, and rate limiting
- **Load testing against production** — always use a dedicated staging environment
- **Ignoring message ordering in tests** — use sequence numbers or timestamps to verify delivery order
