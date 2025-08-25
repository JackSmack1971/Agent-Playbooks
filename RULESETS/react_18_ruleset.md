---
trigger: [glob, model_decision]
description: Comprehensive React 18.x rules covering new concurrent features, performance optimization, best practices, migration patterns, and common issue resolution
globs: ["**/*.{jsx,tsx}", "**/package.json", "**/*.js", "**/*.ts"]
---

# React 18.x Rules

## Setup and Migration

### Root API Migration
- **REQUIRED**: Replace `ReactDOM.render` with `createRoot` for React 18 features
- Migrate legacy render callbacks to `useEffect` hooks
- Use `hydrateRoot` for SSR applications instead of `ReactDOM.hydrate`

```javascript
// ❌ Avoid - Legacy API
import { render } from 'react-dom';
render(<App />, container);

// ✅ Correct - React 18 API
import { createRoot } from 'react-dom/client';
const root = createRoot(container);
root.render(<App />);
```

### Package Dependencies
- Update React and ReactDOM to 18.x simultaneously
- Verify third-party library compatibility (React Router v6+, popular UI libraries)
- Check for React 18 breaking changes in dependencies before upgrading

## Concurrent Features

### useTransition Hook
- Use `useTransition` to mark non-urgent state updates
- Wrap expensive state updates in `startTransition` to maintain responsiveness
- Monitor `isPending` state to show loading indicators during transitions

```javascript
// ✅ Correct - Non-blocking UI updates
import { useTransition, useState } from 'react';

function SearchResults() {
  const [isPending, startTransition] = useTransition();
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);

  const handleSearch = (newQuery) => {
    setQuery(newQuery); // Urgent update
    startTransition(() => {
      setResults(expensiveSearch(newQuery)); // Non-urgent update
    });
  };

  return (
    <div>
      <input onChange={(e) => handleSearch(e.target.value)} />
      {isPending && <Spinner />}
      <Results data={results} />
    </div>
  );
}
```

### useDeferredValue Hook
- Use `useDeferredValue` to defer expensive renders
- Particularly effective for search inputs and large lists
- Does not create fixed delays - adapts to React's scheduling

```javascript
// ✅ Correct - Deferred expensive rendering
import { useDeferredValue, useMemo } from 'react';

function ProductList({ query }) {
  const deferredQuery = useDeferredValue(query);
  
  const filteredProducts = useMemo(() => 
    products.filter(p => p.name.includes(deferredQuery)),
    [deferredQuery]
  );

  return <div>{filteredProducts.map(product => <Product key={product.id} {...product} />)}</div>;
}
```

### useSyncExternalStore Hook
- Use for integrating external stores with concurrent rendering
- Required for custom state management libraries
- Prevents tearing during concurrent updates

```javascript
// ✅ Correct - External store integration
import { useSyncExternalStore } from 'react';

function useOnlineStatus() {
  return useSyncExternalStore(
    (callback) => {
      window.addEventListener('online', callback);
      window.addEventListener('offline', callback);
      return () => {
        window.removeEventListener('online', callback);
        window.removeEventListener('offline', callback);
      };
    },
    () => navigator.onLine,
    () => true // Server snapshot
  );
}
```

### useId Hook
- Use `useId` for generating unique IDs for accessibility
- Essential for SSR to avoid hydration mismatches
- Do not use for React keys in lists

```javascript
// ✅ Correct - Unique ID generation
function Form() {
  const passwordHintId = useId();
  
  return (
    <>
      <input type="password" aria-describedby={passwordHintId} />
      <div id={passwordHintId}>Password must be at least 8 characters</div>
    </>
  );
}
```

## Performance Optimization

### Memoization Strategy
- Use `React.memo` selectively for expensive components
- Implement custom comparison functions for complex props
- Prefer `useCallback` for function props passed to memoized components
- Use `useMemo` for expensive calculations, not simple object creation

```javascript
// ✅ Correct - Strategic memoization
const ExpensiveComponent = React.memo(({ data, onUpdate }) => {
  const processedData = useMemo(() => 
    data.filter(item => item.active).sort((a, b) => b.priority - a.priority),
    [data]
  );

  const handleUpdate = useCallback((id) => {
    onUpdate(id);
  }, [onUpdate]);

  return <List data={processedData} onItemUpdate={handleUpdate} />;
}, (prevProps, nextProps) => {
  // Custom comparison for complex scenarios
  return prevProps.data === nextProps.data && prevProps.onUpdate === nextProps.onUpdate;
});
```

### Virtualization for Large Lists
- Implement virtualization for lists with 100+ items
- Use libraries like `react-window` or `react-virtualized`
- Always provide stable `key` props for list items

```javascript
// ✅ Correct - Virtualized large lists
import { FixedSizeList } from 'react-window';

const VirtualizedList = ({ items }) => (
  <FixedSizeList
    height={600}
    itemCount={items.length}
    itemSize={50}
    itemData={items}
  >
    {({ index, style, data }) => (
      <div style={style} key={data[index].id}>
        {data[index].name}
      </div>
    )}
  </FixedSizeList>
);
```

### Code Splitting and Lazy Loading
- Use `React.lazy` for route-based code splitting
- Implement `Suspense` boundaries for loading states
- Split large component trees into separate bundles

```javascript
// ✅ Correct - Lazy loading with Suspense
import { lazy, Suspense } from 'react';

const Dashboard = lazy(() => import('./Dashboard'));
const Settings = lazy(() => import('./Settings'));

function App() {
  return (
    <Suspense fallback={<LoadingSpinner />}>
      <Routes>
        <Route path="/dashboard" element={<Dashboard />} />
        <Route path="/settings" element={<Settings />} />
      </Routes>
    </Suspense>
  );
}
```

## Automatic Batching

### Understanding Batching Behavior
- React 18 automatically batches all state updates (promises, timeouts, native events)
- Multiple `setState` calls in the same event are batched into single re-render
- Use `flushSync` to force synchronous updates when needed (rare cases)

```javascript
// ✅ Understanding - All these are now batched
function handleClick() {
  setCount(c => c + 1);
  setFlag(f => !f);
  // Single re-render in React 18 (was multiple in React 17)
}

setTimeout(() => {
  setCount(c => c + 1);
  setFlag(f => !f);
  // Single re-render in React 18 (was multiple in React 17)
}, 1000);
```

## Effect and Lifecycle Patterns

### Effect Cleanup and Race Condition Prevention
- Always implement cleanup functions for async operations in `useEffect`
- Use cancellation tokens to prevent setting state on unmounted components
- Make effects idempotent for Strict Mode compatibility

```javascript
// ✅ Correct - Robust effect with cleanup
useEffect(() => {
  let isCancelled = false;
  
  const fetchData = async () => {
    try {
      const result = await api.getData();
      if (!isCancelled) {
        setData(result);
      }
    } catch (error) {
      if (!isCancelled) {
        setError(error);
      }
    }
  };

  fetchData();

  return () => {
    isCancelled = true;
  };
}, [dependency]);
```

### Separation of Concerns
- Use separate `useEffect` hooks for different concerns
- Group related side effects together
- Avoid combining unrelated logic in single effects

```javascript
// ✅ Correct - Separated concerns
function ChatRoom({ roomId }) {
  useEffect(() => {
    // Connection management
    const connection = createConnection(roomId);
    connection.connect();
    return () => connection.disconnect();
  }, [roomId]);

  useEffect(() => {
    // Analytics tracking
    analytics.track('room_visit', { roomId });
  }, [roomId]);
}
```

## Context and State Management

### Context Optimization
- Split contexts by update frequency
- Use multiple small contexts instead of one large context
- Wrap context consumers in `memo` when appropriate

```javascript
// ✅ Correct - Split contexts
const ThemeContext = createContext();
const UserContext = createContext();

// ❌ Avoid - Single large context
const AppContext = createContext(); // Contains theme, user, settings, etc.
```

### State Collocation
- Keep state as close to where it's used as possible
- Lift state up only when multiple components need it
- Use `useReducer` for complex state logic

## Server-Side Rendering (SSR)

### Streaming SSR
- Use `renderToPipeableStream` for Node.js environments
- Use `renderToReadableStream` for edge runtimes
- Implement proper error boundaries for streaming

```javascript
// ✅ Correct - Streaming SSR setup
import { renderToPipeableStream } from 'react-dom/server';

app.get('/', (req, res) => {
  const { pipe } = renderToPipeableStream(<App />, {
    onShellReady() {
      res.statusCode = 200;
      res.setHeader('Content-type', 'text/html');
      pipe(res);
    },
    onError(error) {
      res.statusCode = 500;
      console.error(error);
    }
  });
});
```

## TypeScript Integration

### Improved Type Inference
- Leverage React 18's improved TypeScript support
- Use contextual typing for `useReducer` without explicit type arguments
- Define proper types for ref objects and event handlers

```typescript
// ✅ Correct - Improved TypeScript patterns
interface State {
  count: number;
}

type Action = 
  | { type: 'increment' }
  | { type: 'decrement' }
  | { type: 'reset' };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'increment':
      return { count: state.count + 1 };
    case 'decrement':
      return { count: state.count - 1 };
    case 'reset':
      return { count: 0 };
  }
}

// React 18 - No need for explicit type arguments
const [state, dispatch] = useReducer(reducer, { count: 0 });
```

## Known Issues and Mitigations

### Strict Mode Double Effects
- Expect effects to run twice in development (Strict Mode)
- Design effects to be idempotent
- Use cleanup functions to prevent double subscriptions

```javascript
// ✅ Correct - Idempotent effect design
useEffect(() => {
  const subscription = api.subscribe(callback);
  return () => subscription.unsubscribe(); // Prevents double subscriptions
}, []);
```

### Third-Party Library Compatibility
- Test React Router v6+ compatibility
- Verify UI library support for React 18
- Check for breaking changes in state management libraries

### Hydration Mismatches
- Ensure server and client render identical markup
- Use `suppressHydrationWarning` sparingly for unavoidable mismatches
- Implement proper loading states for dynamic content

```javascript
// ✅ Correct - Handling unavoidable mismatches
function ClientOnlyComponent() {
  const [mounted, setMounted] = useState(false);
  
  useEffect(() => {
    setMounted(true);
  }, []);

  if (!mounted) return null;
  
  return <div>{new Date().toLocaleString()}</div>;
}
```

## Error Boundaries and Suspense

### Error Boundary Implementation
- Implement error boundaries for concurrent features
- Use `onRecoverableError` option in `createRoot`
- Handle both synchronous and asynchronous errors

```javascript
// ✅ Correct - Error boundary with recovery
class ErrorBoundary extends Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false };
  }

  static getDerivedStateFromError(error) {
    return { hasError: true };
  }

  componentDidCatch(error, errorInfo) {
    console.error('Error caught by boundary:', error, errorInfo);
  }

  render() {
    if (this.state.hasError) {
      return <div>Something went wrong.</div>;
    }
    return this.props.children;
  }
}
```

### Suspense Best Practices
- Wrap async components in `Suspense`
- Provide meaningful fallback components
- Use multiple Suspense boundaries for granular loading states

```javascript
// ✅ Correct - Granular Suspense boundaries
function App() {
  return (
    <div>
      <Header />
      <Suspense fallback={<NavSkeleton />}>
        <Navigation />
      </Suspense>
      <Suspense fallback={<ContentSkeleton />}>
        <MainContent />
      </Suspense>
    </div>
  );
}
```

## Testing Considerations

### Test Environment Setup
- Configure testing environments for React 18 features
- Mock concurrent features appropriately
- Test error boundaries and Suspense fallbacks

### Performance Testing
- Use React DevTools Profiler for render analysis
- Monitor Web Vitals in production
- Test with CPU throttling enabled

## Security Considerations

### XSS Prevention
- Continue using React's built-in XSS protection
- Validate and sanitize external data
- Use `dangerouslySetInnerHTML` with extreme caution

### Dependency Management
- Keep React and dependencies updated
- Monitor security advisories for React ecosystem
- Use tools like `npm audit` regularly

## Version-Specific Notes

**React 18.3.1 (Latest Stable)**
- Includes all concurrent features in stable form
- Improved TypeScript definitions
- Better error messages and debugging experience
- Full backwards compatibility with React 17 patterns (except root API)

**Migration Timeline**
- Low risk: Basic apps with standard patterns
- Medium risk: Apps with complex state management or third-party dependencies
- High risk: Apps with custom renderers or advanced SSR setups