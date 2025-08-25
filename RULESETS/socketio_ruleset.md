---
trigger: model_decision
description: Comprehensive Socket.IO v4.x ruleset for real-time bidirectional communication with WebSocket and HTTP long-polling support
globs: ["**/*.js", "**/*.ts", "**/*.mjs", "**/*.cjs"]
---

# Socket.IO v4.x Rules

## Server Initialization and Configuration

### Use Proper Module Import Syntax
- **ES Modules**: `import { Server } from "socket.io";`
- **CommonJS**: `const { Server } = require("socket.io");`
- **TypeScript**: Import with proper types: `import { Server, Socket } from "socket.io";`
- Never use default exports for Socket.IO v4.x+ (breaking change from v3.x)

### Initialize Server Instance Correctly
- **With HTTP Server**: `const io = new Server(httpServer, options);`
- **Standalone**: `const io = new Server(3000, options);` or `const io = new Server(options); io.listen(3000);`
- **With Express**: Always create HTTP server first: `const server = createServer(app); const io = new Server(server);`
- **With Fastify**: Use `fastify-socket.io` plugin and wait for server.ready() before accessing `server.io`

### Essential Server Configuration Options
```javascript
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
  transports: ["websocket", "polling"], // Prefer WebSocket
  upgradeTimeout: 10000,
  allowUpgrades: true
});
```

### Performance Optimization Server Settings
```javascript
const io = new Server(httpServer, {
  pingInterval: 10000,    // Reduce for faster disconnect detection
  pingTimeout: 5000,      // Shorter timeout for production
  maxHttpBufferSize: 1e6, // Limit payload size to prevent DoS
  compression: true,      // Enable compression
  httpCompression: true   // Enable HTTP compression
});
```

## Client Setup and Connection

### Browser Client Installation
- **CDN**: `<script src="https://cdn.socket.io/4.8.1/socket.io.min.js" integrity="..." crossorigin="anonymous"></script>`
- **NPM**: `npm install socket.io-client`
- **ES Modules**: `import { io } from "socket.io-client";`
- **CommonJS**: `const { io } = require("socket.io-client");`

### Client Connection Best Practices
```javascript
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

### Handle Connection States Properly
```javascript
socket.on('connect', () => {
  console.log('Connected:', socket.id);
  socket.connected; // true
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
  // Handle specific error types
  if (error.message === 'xhr poll error') {
    // Network connectivity issue
  }
});
```

## Event Handling and Emission

### Event Naming Conventions
- Use kebab-case for event names: `'user-joined'`, `'message-sent'`
- Avoid Socket.IO reserved events: `connect`, `disconnect`, `error`, `newListener`, `removeListener`
- Use descriptive, action-oriented names: `'chat-message'` not `'message'`

### Safe Event Emission Patterns
```javascript
// Server-side emission with error handling
io.emit('global-update', data);
socket.emit('personal-message', data);
socket.to(roomId).emit('room-update', data);
socket.broadcast.emit('user-action', data);

// With acknowledgment
socket.emit('request-data', payload, (response) => {
  if (response.error) {
    console.error('Request failed:', response.error);
  }
});

// Promise-based acknowledgment (v4.6.0+)
try {
  const response = await socket.emitWithAck('async-request', data);
  console.log('Response:', response);
} catch (error) {
  console.error('Request failed:', error);
}
```

### Event Listener Best Practices
```javascript
// Server-side event handling
io.on('connection', (socket) => {
  console.log('User connected:', socket.id);

  // Always validate incoming data
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

  socket.on('disconnect', (reason) => {
    console.log('User disconnected:', socket.id, reason);
    // Cleanup user-specific resources
  });
});
```

## Namespaces and Rooms

### Namespace Usage Patterns
```javascript
// Create namespaces for logical separation
const adminNamespace = io.of('/admin');
const userNamespace = io.of('/user');

// Dynamic namespaces with regex
io.of(/^\/dynamic-\d+$/).on('connection', (socket) => {
  const namespace = socket.nsp.name; // e.g., '/dynamic-101'
});

// Namespace-specific middleware
adminNamespace.use((socket, next) => {
  if (isAdmin(socket.request.user)) {
    next();
  } else {
    next(new Error('Unauthorized'));
  }
});
```

### Room Management Best Practices
```javascript
// Join/leave rooms safely
socket.on('join-room', (roomId) => {
  if (!isValidRoomId(roomId)) {
    socket.emit('error', { message: 'Invalid room ID' });
    return;
  }
  
  socket.join(roomId);
  socket.emit('joined-room', { roomId });
  socket.to(roomId).emit('user-joined', { 
    userId: socket.userId, 
    roomId 
  });
});

// Broadcast to room with sender exclusion
socket.on('room-message', (data) => {
  socket.to(data.roomId).emit('room-message', {
    message: data.message,
    sender: socket.userId,
    timestamp: Date.now()
  });
});

// Auto-cleanup rooms on disconnect
socket.on('disconnect', () => {
  // Socket.IO automatically removes from all rooms
  // Manual cleanup if needed:
  socket.rooms.forEach(room => {
    socket.to(room).emit('user-left', { userId: socket.userId });
  });
});
```

## Middleware and Authentication

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

// Rate limiting middleware
io.use(rateLimit({
  windowMs: 60 * 1000, // 1 minute
  max: 100, // 100 requests per minute
  message: 'Too many requests'
}));
```

### Connection Validation
```javascript
// Validate connection parameters
io.use((socket, next) => {
  const { version, clientId } = socket.handshake.query;
  
  if (!version || !isValidVersion(version)) {
    return next(new Error('Invalid client version'));
  }
  
  if (!clientId || !isValidClientId(clientId)) {
    return next(new Error('Invalid client ID'));
  }
  
  next();
});
```

## Error Handling and Debugging

### Comprehensive Error Handling
```javascript
// Server-side error handling
io.on('connection', (socket) => {
  socket.on('error', (error) => {
    console.error('Socket error:', error);
    // Log error details for debugging
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

// Client-side error handling
socket.on('connect_error', (error) => {
  console.error('Connection failed:', error.message);
  
  if (error.message === 'Authentication failed') {
    // Redirect to login
    window.location.href = '/login';
  } else if (error.message === 'xhr poll error') {
    // Network issue, show offline message
    showOfflineNotification();
  }
});
```

### Debug Mode Configuration
```javascript
// Enable debug logs (development only)
localStorage.debug = 'socket.io-client:socket';

// Server-side debugging
const debug = require('debug')('socket.io:server');
debug('Server starting on port %d', port);
```

## Performance Optimization

### Connection Limits and Resource Management
```javascript
// Optimize server for high concurrency
const io = new Server(httpServer, {
  pingInterval: 10000,
  pingTimeout: 5000,
  maxHttpBufferSize: 1e6,
  transports: ['websocket'], // WebSocket only for performance
  allowUpgrades: false,
  compression: true,
  perMessageDeflate: {
    threshold: 1024,
    concurrencyLimit: 10,
    memLevel: 7
  }
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

### Message Optimization
```javascript
// Efficient message broadcasting
const throttle = require('lodash.throttle');

// Throttle high-frequency events
const broadcastUpdate = throttle((data) => {
  io.emit('live-update', data);
}, 100); // Max once per 100ms

// Batch multiple updates
const pendingUpdates = [];
const flushUpdates = debounce(() => {
  if (pendingUpdates.length > 0) {
    io.emit('batch-update', pendingUpdates);
    pendingUpdates.length = 0;
  }
}, 50);
```

### Memory Management
```javascript
// Clean up event listeners
socket.on('disconnect', () => {
  socket.removeAllListeners();
  // Clear any user-specific data
  delete userSessions[socket.userId];
});

// Implement connection pooling for database
const pool = new Pool({
  connectionLimit: 10,
  acquireTimeout: 60000,
  timeout: 60000
});
```

## Production Deployment and Scaling

### Clustering with Adapters
```javascript
// Redis Adapter for horizontal scaling
import { createAdapter } from '@socket.io/redis-adapter';
import { Redis } from 'ioredis';

const pubClient = new Redis(process.env.REDIS_URL);
const subClient = pubClient.duplicate();

io.adapter(createAdapter(pubClient, subClient));

// Cluster Adapter for Node.js clustering
import { createAdapter, setupPrimary } from '@socket.io/cluster-adapter';

if (cluster.isPrimary) {
  setupPrimary();
  // Fork workers
  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }
} else {
  io.adapter(createAdapter());
}
```

### Load Balancer Configuration
```javascript
// Sticky sessions with socket.io-sticky
const { setupMaster, setupWorker } = require('@socket.io/sticky');

if (cluster.isMaster) {
  setupMaster(httpServer, {
    loadBalancingMethod: 'least-connection'
  });
} else {
  setupWorker(io);
}
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

## Security Best Practices

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

### Input Validation and Sanitization
```javascript
const joi = require('joi');

const messageSchema = joi.object({
  message: joi.string().max(500).required(),
  roomId: joi.string().uuid().required()
});

socket.on('send-message', (data) => {
  const { error, value } = messageSchema.validate(data);
  if (error) {
    socket.emit('error', { message: 'Invalid input' });
    return;
  }
  
  // Sanitize HTML
  const cleanMessage = DOMPurify.sanitize(value.message);
  // Process validated and sanitized data
});
```

### Rate Limiting
```javascript
const rateLimit = new Map();

socket.use((packet, next) => {
  const userId = socket.userId;
  const now = Date.now();
  
  if (!rateLimit.has(userId)) {
    rateLimit.set(userId, { count: 1, resetTime: now + 60000 });
    return next();
  }
  
  const userLimit = rateLimit.get(userId);
  if (now > userLimit.resetTime) {
    userLimit.count = 1;
    userLimit.resetTime = now + 60000;
    return next();
  }
  
  if (userLimit.count >= 100) { // 100 messages per minute
    return next(new Error('Rate limit exceeded'));
  }
  
  userLimit.count++;
  next();
});
```

## Testing Best Practices

### Test Environment Setup
```javascript
// Vitest/Jest test setup
import { beforeAll, afterAll, describe, it, expect } from 'vitest';
import { createServer } from 'node:http';
import { Server } from 'socket.io';
import { io as Client } from 'socket.io-client';

let io, serverSocket, clientSocket, httpServer;

beforeAll(() => {
  return new Promise((resolve) => {
    httpServer = createServer();
    io = new Server(httpServer);
    httpServer.listen(() => {
      const port = httpServer.address().port;
      clientSocket = Client(`http://localhost:${port}`);
      io.on('connection', (socket) => {
        serverSocket = socket;
      });
      clientSocket.on('connect', resolve);
    });
  });
});

afterAll(() => {
  io.close();
  clientSocket.disconnect();
  httpServer.close();
});
```

### Test Event Handling
```javascript
it('should handle message exchange', (done) => {
  clientSocket.emit('test-message', 'hello');
  
  serverSocket.on('test-message', (data) => {
    expect(data).toBe('hello');
    serverSocket.emit('response', 'world');
  });
  
  clientSocket.on('response', (data) => {
    expect(data).toBe('world');
    done();
  });
});

it('should handle acknowledgments', async () => {
  serverSocket.on('ping', (callback) => {
    callback('pong');
  });
  
  const response = await clientSocket.emitWithAck('ping');
  expect(response).toBe('pong');
});
```

## Known Issues and Mitigations

### Common Connection Issues
- **IPv6 localhost issues**: Use `127.0.0.1` instead of `localhost` to force IPv4
- **CORS errors**: Ensure proper CORS configuration for cross-origin requests
- **Sticky sessions**: Required for load balancing with multiple server instances
- **Firewall blocking WebSockets**: Ensure both ports 80/443 and custom ports are open

### Transport Fallback Issues
```javascript
// Ensure proper transport configuration
const socket = io('https://example.com', {
  transports: ['websocket', 'polling'],
  upgrade: true,
  rememberUpgrade: true,
  transports: ['websocket'] // WebSocket only after successful upgrade
});
```

### Memory Leaks Prevention
```javascript
// Clean up event listeners
socket.on('disconnect', () => {
  socket.removeAllListeners();
  clearInterval(socket.heartbeat);
  delete socket.customData;
});

// Avoid duplicate listeners
socket.off('event-name').on('event-name', handler);
```

### Performance Issues
- **Avoid sending large payloads**: Keep messages under 1MB, use chunking for larger data
- **Implement message queuing**: For high-frequency events, use batching or throttling
- **Monitor connection counts**: Implement connection limits to prevent resource exhaustion
- **Use compression**: Enable compression for better bandwidth usage

### Version Compatibility
- **Socket.IO v4.x requires Engine.IO v6**: Ensure compatible versions
- **Node.js version**: Requires Node.js 10+ (16+ recommended)
- **Client-server version matching**: Use same major version for client and server

## Optional Performance Dependencies

### WebSocket Performance Optimizations
```bash
# Install optional binary add-ons for better performance
npm install --save-optional bufferutil utf-8-validate

# Alternative WebSocket engines
npm install eiows  # High-performance WebSocket server
npm install uWebSockets.js@uNetworking/uWebSockets.js#v20.4.0
```

### Configure Alternative WebSocket Engine
```javascript
const io = new Server(httpServer, {
  wsEngine: require('eiows').Server,
  // or
  wsEngine: require('uws').Server
});
```

## Database Integration Patterns

### Connection State Recovery
```javascript
const io = new Server(httpServer, {
  connectionStateRecovery: {
    maxDisconnectionDuration: 2 * 60 * 1000, // 2 minutes
    skipMiddlewares: true,
  }
});

// Store messages for recovery
socket.on('chat-message', async (msg, clientOffset) => {
  try {
    const result = await db.run(
      'INSERT INTO messages (content, client_offset) VALUES (?, ?)',
      msg, clientOffset
    );
    io.emit('chat-message', msg, result.lastID);
  } catch (e) {
    // Handle duplicate clientOffset (message already processed)
    if (e.code !== 'SQLITE_CONSTRAINT_UNIQUE') {
      throw e;
    }
  }
});
```

### Message Persistence
```javascript
// PostgreSQL message storage
socket.on('chat-message', async (data) => {
  const message = await db.query(
    'INSERT INTO messages (content, user_id, room_id, created_at) VALUES ($1, $2, $3, NOW()) RETURNING *',
    [data.content, socket.userId, data.roomId]
  );
  
  io.to(data.roomId).emit('chat-message', {
    id: message.rows[0].id,
    content: data.content,
    userId: socket.userId,
    roomId: data.roomId,
    timestamp: message.rows[0].created_at
  });
});
```

This ruleset covers Socket.IO v4.x best practices, common issues, and production-ready patterns for building scalable real-time applications.