---
trigger: ["model_decision"]
description: Comprehensive Socket.IO v4.x ruleset for real-time bidirectional communication with WebSocket and HTTP long-polling support
globs: ["**/*.js", "**/*.ts", "**/*.mjs", "**/*.cjs"]
version: "4.8.1"
last_updated: "2025-01-01"
---

# Socket.IO v4.x Rules

## Installation & Setup

- **ALWAYS** use Socket.IO v4.x for latest features and security updates
- **INSTALL** server and client packages: `npm install socket.io` and `npm install socket.io-client`
- **CONFIGURE** proper module imports based on your environment

```bash
# Server installation
npm install socket.io

# Client installation
npm install socket.io-client
```

## Configuration & Initialization

### Server Configuration

```javascript
const { Server } = require("socket.io");
const httpServer = require("http").createServer();

const io = new Server(httpServer, {
  cors: {
    origin: ["https://example.com"],
    methods: ["GET", "POST"],
    credentials: true
  },
  pingTimeout: 60000,
  pingInterval: 25000,
  connectTimeout: 45000,
  maxHttpBufferSize: 1e6,
  allowEIO3: false, // Disable Engine.IO v3 support for security
  transports: ["websocket", "polling"],
  upgradeTimeout: 10000,
  allowUpgrades: true
});
```

### Client Configuration

```javascript
import { io } from "socket.io-client";

const socket = io('https://example.com', {
  transports: ['websocket', 'polling'],
  upgrade: true,
  reconnection: true,
  reconnectionDelay: 1000,
  reconnectionDelayMax: 5000,
  maxReconnectionAttempts: 5,
  timeout: 20000,
  forceNew: false, // Reuse existing manager
  auth: {
    token: "your-auth-token"
  },
  query: {
    userId: "12345"
  }
});
```

## Core Concepts / API Usage

### Connection Handling

```javascript
// Server-side connection handling
io.on('connection', (socket) => {
  console.log('User connected:', socket.id);

  socket.on('disconnect', (reason) => {
    console.log('User disconnected:', socket.id, reason);
    // Cleanup user-specific resources
  });
});

// Client-side connection handling
socket.on('connect', () => {
  console.log('Connected:', socket.id);
});

socket.on('disconnect', (reason) => {
  if (reason === 'io server disconnect') {
    // Server initiated disconnect, manually reconnect
    socket.connect();
  }
  // else: client disconnect, auto-reconnect will handle
});

socket.on('connect_error', (error) => {
  console.error('Connection error:', error.message);
});
```

### Event Handling

```javascript
// Server-side event handling with validation
io.on('connection', (socket) => {
  socket.on('chat-message', (data) => {
    if (!data || typeof data.message !== 'string') {
      socket.emit('error', { message: 'Invalid message format' });
      return;
    }

    if (data.message.length > 500) {
      socket.emit('error', { message: 'Message too long' });
      return;
    }

    // Sanitize and process
    const sanitizedMessage = escapeHtml(data.message);
    io.emit('chat-message', {
      id: generateMessageId(),
      message: sanitizedMessage,
      user: socket.userId,
      timestamp: Date.now()
    });
  });
});
```

## Security & Permissions

### Authentication Middleware

```javascript
// Server-side authentication
io.use((socket, next) => {
  const token = socket.handshake.auth.token;

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    socket.userId = decoded.userId;
    socket.user = decoded;
    next();
  } catch (err) {
    next(new Error('Authentication failed'));
  }
});
```

### CORS Configuration

```javascript
const io = new Server(httpServer, {
  cors: {
    origin: (origin, callback) => {
      const allowedOrigins = process.env.ALLOWED_ORIGINS?.split(',') || [];
      if (!origin || allowedOrigins.includes(origin)) {
        callback(null, true);
      } else {
        callback(new Error('Not allowed by CORS'));
      }
    },
    methods: ['GET', 'POST'],
    credentials: true
  }
});
```

## Performance & Scalability

### Connection Limits and Resource Management

```javascript
const io = new Server(httpServer, {
  pingInterval: 10000,
  pingTimeout: 5000,
  maxHttpBufferSize: 1e6,
  transports: ['websocket'], // WebSocket only for performance
  allowUpgrades: false,
  compression: true,
  httpCompression: true
});

// Connection counting and limits
let connectionCount = 0;
const MAX_CONNECTIONS = 10000;

io.use((socket, next) => {
  if (connectionCount >= MAX_CONNECTIONS) {
    return next(new Error('Server at capacity'));
  }
  connectionCount++;
  next();
});

io.on('connection', (socket) => {
  socket.on('disconnect', () => {
    connectionCount--;
  });
});
```

## Error Handling & Troubleshooting

### Comprehensive Error Handling

```javascript
// Server-side error handling
io.on('connection', (socket) => {
  socket.on('error', (error) => {
    console.error('Socket error:', error);
    logger.error('Socket error', {
      socketId: socket.id,
      userId: socket.userId,
      error: error.message,
      stack: error.stack
    });
  });
});

// Global error handler
io.engine.on('connection_error', (err) => {
  console.error('Connection error:', {
    code: err.code,
    message: err.message,
    context: err.context,
    req: err.req && err.req.url
  });
});
```

## Testing

### Testing Socket.IO Applications

```javascript
const { createServer } = require("http");
const { Server } = require("socket.io");
const { io: Client } = require("socket.io-client");

describe("Socket.IO Server", () => {
  let io, serverSocket, clientSocket;

  beforeAll((done) => {
    const httpServer = createServer();
    io = new Server(httpServer);
    httpServer.listen(() => {
      const port = httpServer.address().port;
      clientSocket = new Client(`http://localhost:${port}`);
      io.on("connection", (socket) => {
        serverSocket = socket;
      });
      clientSocket.on("connect", done);
    });
  });

  afterAll(() => {
    io.close();
    clientSocket.close();
  });

  test("should communicate", (done) => {
    clientSocket.emit("echo", "hello");
    serverSocket.on("echo", (arg) => {
      expect(arg).toBe("hello");
      done();
    });
  });
});
```

## Deployment & Production Patterns

### Clustering with Redis

```javascript
import { createAdapter } from '@socket.io/redis-adapter';
import { Redis } from 'ioredis';

const pubClient = new Redis(process.env.REDIS_URL);
const subClient = pubClient.duplicate();

io.adapter(createAdapter(pubClient, subClient));
```

### Environment-Specific Configuration

```javascript
const config = {
  development: {
    cors: { origin: "*" },
    allowEIO3: true
  },
  production: {
    cors: {
      origin: process.env.ALLOWED_ORIGINS?.split(','),
      credentials: true
    },
    allowEIO3: false,
    cookie: {
      httpOnly: true,
      secure: true,
      sameSite: 'strict'
    }
  }
};

const io = new Server(httpServer, config[process.env.NODE_ENV]);
```

## Known Issues & Mitigations

- **Connection drops**: Implement proper reconnection logic with exponential backoff
- **Memory leaks**: Clean up event listeners and user-specific data on disconnect
- **CORS issues**: Validate origin configuration and handle preflight requests
- **Performance degradation**: Monitor connection count and implement connection limits
- **Authentication failures**: Validate JWT tokens and handle token refresh

## Version Compatibility Notes

- **Current version tested**: Socket.IO v4.8.1
- **Breaking changes**: Major API changes from v3.x - review migration guide
- **Engine.IO compatibility**: Ensure compatible Engine.IO versions
- **Client compatibility**: Match client and server versions for best compatibility

## References

- [Socket.IO Official Documentation](https://socket.io/docs/v4/)
- [Socket.IO Server API](https://socket.io/docs/v4/server-api/)
- [Socket.IO Client API](https://socket.io/docs/v4/client-api/)
- [Socket.IO Scaling Guide](https://socket.io/docs/v4/using-multiple-nodes/)