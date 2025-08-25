---
trigger: "model_decision"
description: "Comprehensive ruleset for JavaScript ES2023 (ECMAScript 2023/ES14) covering new features, best practices, performance considerations, and compatibility requirements"
globs: ["*.js", "*.mjs", "*.ts", "*.tsx", "*.jsx"]
---

# JavaScript ES2023 (ECMAScript 2023/ES14) Rules

## Browser Support and Compatibility

- **REQUIRED**: Verify ES2023 feature support before using in production environments
- **MODERN BROWSERS**: All major browsers support ES2023 as of mid-2023 (Chrome 110+, Edge 110+, Firefox 115+, Safari 16.4+, Opera 96+)
- **NODE.JS**: Requires Node.js 18.0+ for full ES2023 support
- **TRANSPILATION**: Use Babel or TypeScript for legacy browser support, but note that some features (top-level await, regex v flag) cannot be fully polyfilled
- **FEATURE DETECTION**: Always implement feature detection for critical ES2023 functionality in environments with mixed support

```javascript
// Feature detection example
if (Array.prototype.findLast) {
  // Use ES2023 method
  const result = array.findLast(predicate);
} else {
  // Fallback implementation
  const result = array.slice().reverse().find(predicate);
}
```

## New Array Methods - Immutable Operations

### Array.prototype.toReversed(), toSorted(), toSpliced()

- **IMMUTABLE PRINCIPLE**: Always use immutable array methods when you need to preserve the original array
- **MEMORY CONSIDERATION**: These methods create new arrays - be mindful of memory usage with large datasets
- **CONSISTENT PATTERN**: Prefer immutable methods for functional programming patterns and Redux-style state management

```javascript
// ✅ CORRECT: Using immutable methods
const originalArray = [1, 2, 3, 4, 5];
const reversed = originalArray.toReversed(); // [5, 4, 3, 2, 1]
const sorted = originalArray.toSorted((a, b) => b - a); // [5, 4, 3, 2, 1]
const spliced = originalArray.toSpliced(1, 2, 'new'); // [1, 'new', 4, 5]

// ❌ INCORRECT: Mutating when immutability is needed
originalArray.reverse(); // Mutates original array
originalArray.sort(); // Mutates original array
```

### Array.prototype.with()

- **INDEX REPLACEMENT**: Use `with()` for creating new arrays with single element replacements
- **BOUNDS CHECKING**: Method throws RangeError for out-of-bounds indices
- **NEGATIVE INDICES**: Supports negative indexing like bracket notation

```javascript
// ✅ CORRECT: Using with() for immutable updates
const colors = ['red', 'green', 'blue'];
const newColors = colors.with(1, 'yellow'); // ['red', 'yellow', 'blue']
const lastChanged = colors.with(-1, 'purple'); // ['red', 'green', 'purple']

// ✅ CORRECT: Error handling for invalid indices
try {
  const invalid = colors.with(10, 'orange'); // Throws RangeError
} catch (error) {
  console.error('Invalid index:', error.message);
}

// ❌ INCORRECT: Not handling potential RangeError
const risky = colors.with(userInput, 'new'); // May throw if userInput is invalid
```

### Array.prototype.findLast() and findLastIndex()

- **REVERSE SEARCH**: Use when you need the last matching element instead of the first
- **PERFORMANCE**: More efficient than reverse().find() for large arrays
- **RETURN VALUES**: findLast() returns undefined, findLastIndex() returns -1 when no match found

```javascript
// ✅ CORRECT: Using findLast for reverse search
const logs = [
  { level: 'info', message: 'App started' },
  { level: 'error', message: 'Database error' },
  { level: 'info', message: 'Request processed' },
  { level: 'error', message: 'API timeout' }
];

const lastError = logs.findLast(log => log.level === 'error');
const lastErrorIndex = logs.findLastIndex(log => log.level === 'error');

// ✅ CORRECT: Handling undefined/not found cases
if (lastError) {
  console.log('Last error:', lastError.message);
} else {
  console.log('No errors found');
}

// ❌ INCORRECT: Not checking for undefined
console.log(lastError.message); // May throw if no error found

// ❌ INCORRECT: Inefficient alternative
const inefficient = logs.slice().reverse().find(log => log.level === 'error');
```

## Top-level await

- **MODULE REQUIREMENT**: Only available in ES modules (type="module" or .mjs files)
- **BLOCKING BEHAVIOR**: Top-level await blocks module initialization - use sparingly
- **DEPENDENCY IMPACT**: Can delay dependent module loading
- **ERROR HANDLING**: Always wrap in try-catch for robust error handling

```javascript
// ✅ CORRECT: Top-level await in ES module
// config.mjs
try {
  const config = await fetch('/api/config').then(r => r.json());
  export default config;
} catch (error) {
  console.error('Failed to load config:', error);
  export default { /* fallback config */ };
}

// ✅ CORRECT: Non-blocking initialization
const configPromise = fetch('/api/config').then(r => r.json());
export { configPromise };

// ❌ INCORRECT: Blocking critical path
await new Promise(resolve => setTimeout(resolve, 5000)); // Delays all dependent modules

// ❌ INCORRECT: Using in CommonJS
// This will not work in .js files without type="module"
const data = await fetch('/api/data'); // SyntaxError
```

## Regular Expression v Flag

- **UNICODE SUPPORT**: Enhanced Unicode property escapes and set notation
- **SET NOTATION**: Supports mathematical set operations in character classes
- **COMPATIBILITY**: Not supported in older browsers - feature detect before use
- **ESCAPE SEQUENCES**: Different escaping rules compared to u flag

```javascript
// ✅ CORRECT: Using v flag for advanced Unicode patterns
const emojiRegex = /^\p{Emoji}+$/v;
const latinScript = /^\p{Script=Latin}+$/v;

// ✅ CORRECT: Set notation operations
const digitOrLetter = /^[\p{L}&&[\p{ASCII}]]+$/v; // Intersection
const notDigit = /^[^\p{N}]+$/v; // Complement

// ✅ CORRECT: Feature detection
function createRegexWithVFlag(pattern) {
  try {
    return new RegExp(pattern, 'v');
  } catch (error) {
    // Fallback for browsers without v flag support
    return new RegExp(pattern, 'u');
  }
}

// ❌ INCORRECT: Using without feature detection
const regex = /pattern/v; // May fail in older browsers

// ❌ INCORRECT: Mixing v and u flags
const invalid = /pattern/vu; // SyntaxError
```

## Hashbang Grammar

- **CLI SCRIPTS**: Only relevant for scripts intended for direct execution
- **POSITION REQUIREMENT**: Must be the very first line of the file
- **BROWSER BEHAVIOR**: Ignored by browsers, treated as comment
- **NODE.JS EXECUTION**: Enables direct script execution without explicit node command

```javascript
// ✅ CORRECT: Hashbang for CLI script
#!/usr/bin/env node
// build-script.mjs
console.log('Building application...');

// ✅ CORRECT: Conditional hashbang handling
#!/usr/bin/env node
if (typeof window === 'undefined') {
  // Node.js environment
  import('./cli-implementation.mjs');
} else {
  // Browser environment
  console.log('Running in browser');
}

// ❌ INCORRECT: Hashbang not on first line
// Some comment
#!/usr/bin/env node // Invalid position

// ❌ INCORRECT: Using in browser-only scripts
#!/usr/bin/env node
// This is unnecessary for browser scripts
```

## Symbols as WeakMap Keys

- **MEMORY MANAGEMENT**: Leverages WeakMap's garbage collection for symbols
- **PRIVATE METADATA**: Useful for associating private data with symbols
- **SYMBOL LIFECYCLE**: WeakMap entries are cleaned up when symbols are garbage collected
- **USE CASES**: Ideal for library internal state management

```javascript
// ✅ CORRECT: Using symbols as WeakMap keys
const symbolMetadata = new WeakMap();
const mySymbol = Symbol('metadata');
const metadata = { created: Date.now(), type: 'special' };

symbolMetadata.set(mySymbol, metadata);

function getSymbolInfo(sym) {
  return symbolMetadata.get(sym);
}

// ✅ CORRECT: Cleanup happens automatically
function createTemporarySymbol() {
  const tempSymbol = Symbol('temp');
  symbolMetadata.set(tempSymbol, { temporary: true });
  return tempSymbol;
  // When tempSymbol goes out of scope and is GC'd,
  // WeakMap entry is automatically removed
}

// ❌ INCORRECT: Using with non-symbol/non-object keys
const invalidWeakMap = new WeakMap();
invalidWeakMap.set('string', data); // TypeError

// ❌ INCORRECT: Assuming symbols are always enumerable
for (const key in symbolMetadata) {
  // This won't work - WeakMaps are not enumerable
}
```

## TypeScript Compatibility

- **LIB CONFIGURATION**: Add "ES2023" and "ES2023.Array" to tsconfig.json lib array
- **TARGET SETTING**: Set target to "ES2022" or higher for ES2023 features
- **VERSION REQUIREMENT**: Requires TypeScript 5.2+ for full ES2023 support
- **TYPE DEFINITIONS**: Some features may require @types updates

```json
// ✅ CORRECT: TypeScript configuration for ES2023
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2023", "ES2023.Array", "DOM"],
    "module": "ESNext",
    "moduleResolution": "bundler"
  }
}
```

```typescript
// ✅ CORRECT: Using ES2023 features in TypeScript
interface LogEntry {
  level: 'info' | 'error' | 'warn';
  message: string;
  timestamp: number;
}

const logs: LogEntry[] = [
  { level: 'info', message: 'Start', timestamp: 1 },
  { level: 'error', message: 'Error', timestamp: 2 }
];

// Type-safe usage
const lastError: LogEntry | undefined = logs.findLast(log => log.level === 'error');
const sortedLogs: LogEntry[] = logs.toSorted((a, b) => a.timestamp - b.timestamp);

// ❌ INCORRECT: Missing lib configuration
// Property 'findLast' does not exist on type 'LogEntry[]'
```

## Performance Considerations

### Memory Management

- **IMMUTABLE METHODS**: Create new arrays - monitor memory usage with large datasets
- **GARBAGE COLLECTION**: WeakMap with symbols enables automatic cleanup
- **ARRAY SIZE**: Consider memory implications when using toSorted/toReversed on large arrays

```javascript
// ✅ CORRECT: Memory-conscious large array handling
function processLargeArray(data) {
  if (data.length > 10000) {
    // For very large arrays, consider streaming or chunking
    console.warn('Large array detected, consider alternative approach');
  }
  
  return data.toSorted((a, b) => a.value - b.value);
}

// ✅ CORRECT: Reusing sorted results
const sortComparator = (a, b) => a.timestamp - b.timestamp;
const sorted = data.toSorted(sortComparator);
// Reuse 'sorted' instead of calling toSorted multiple times

// ❌ INCORRECT: Repeatedly creating sorted arrays
const result1 = data.toSorted(comparator);
const result2 = data.toSorted(comparator); // Wasteful duplication
```

### Regex Performance

- **COMPILATION CACHING**: Cache regex objects when using v flag repeatedly
- **COMPLEXITY MONITORING**: Complex Unicode patterns may impact performance

```javascript
// ✅ CORRECT: Caching compiled regex
const EMOJI_REGEX = /^\p{Emoji}+$/v;

function isEmoji(text) {
  return EMOJI_REGEX.test(text);
}

// ❌ INCORRECT: Recompiling regex repeatedly
function isEmoji(text) {
  return /^\p{Emoji}+$/v.test(text); // Compiles regex each time
}
```

## Security Considerations

- **TOP-LEVEL AWAIT**: Can be exploited for timing attacks - avoid sensitive operations
- **REGEX COMPLEXITY**: Complex Unicode patterns may enable ReDoS attacks
- **SYMBOL KEYS**: While providing some privacy, symbols are not truly private

```javascript
// ✅ CORRECT: Safe top-level await usage
const publicConfig = await fetch('/api/public-config').then(r => r.json());

// ❌ INCORRECT: Exposing sensitive data
const apiKeys = await fetch('/api/keys').then(r => r.json()); // Security risk

// ✅ CORRECT: ReDoS protection
function validateInput(input, timeout = 100) {
  const startTime = Date.now();
  const regex = /complex-pattern/v;
  
  if (Date.now() - startTime > timeout) {
    throw new Error('Regex timeout - potential ReDoS');
  }
  
  return regex.test(input);
}
```

## Known Issues and Mitigations

### TypeScript Integration

- **VSCode VERSION**: Some versions may show incorrect errors with ES2023 target
- **LIB MISMATCH**: Ensure TypeScript version matches ES2023 lib declarations
- **BUILD TOOLS**: Verify bundler/build tool supports ES2023 syntax

### Browser Compatibility

- **FEATURE DETECTION**: Always detect feature availability in production
- **POLYFILL LIMITATIONS**: Some features cannot be polyfilled (top-level await, regex v flag)
- **TRANSPILATION**: Use appropriate Babel presets for target environment

```javascript
// ✅ CORRECT: Comprehensive feature detection
const ES2023_SUPPORT = {
  findLast: Array.prototype.findLast !== undefined,
  toReversed: Array.prototype.toReversed !== undefined,
  regexVFlag: (() => {
    try {
      new RegExp('', 'v');
      return true;
    } catch {
      return false;
    }
  })(),
  topLevelAwait: false // Detected at module level
};

export { ES2023_SUPPORT };

// ✅ CORRECT: Progressive enhancement
function findLastElement(array, predicate) {
  if (ES2023_SUPPORT.findLast) {
    return array.findLast(predicate);
  }
  
  // Fallback implementation
  for (let i = array.length - 1; i >= 0; i--) {
    if (predicate(array[i], i, array)) {
      return array[i];
    }
  }
  return undefined;
}
```

## Migration Best Practices

### Gradual Adoption

- **INCREMENTAL UPDATES**: Introduce ES2023 features gradually
- **TEAM TRAINING**: Ensure team understanding of new features before adoption
- **TESTING STRATEGY**: Comprehensive testing across target environments

### Code Review Guidelines

- **IMMUTABILITY CHECKS**: Verify proper use of immutable array methods
- **PERFORMANCE REVIEW**: Assess memory implications of new methods
- **COMPATIBILITY VERIFICATION**: Confirm feature support in target environments

### Tooling Updates

- **ESLINT RULES**: Update ESLint configuration for ES2023 support
- **PRETTIER**: Ensure code formatter handles new syntax
- **BUNDLER CONFIG**: Update webpack/rollup for ES2023 compatibility

```javascript
// ✅ CORRECT: ESLint configuration update
// .eslintrc.js
module.exports = {
  parserOptions: {
    ecmaVersion: 2023,
    sourceType: 'module'
  },
  env: {
    es2023: true
  }
};
```

## Documentation Sources

- **Official Specification**: ECMAScript 2023 Language Specification (ECMA-262, 14th edition)
- **Version Information**: ES2023/ES14, released June 2023
- **Browser Support**: Can I Use ES2023 features
- **TypeScript Support**: TypeScript 5.2+ required for full compatibility