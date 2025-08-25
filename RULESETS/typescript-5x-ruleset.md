---
trigger: glob
description: Comprehensive TypeScript 5.x rules for AI agents, covering configuration, syntax, best practices, and common pitfalls based on official documentation (v5.9.2) and recent release notes
globs: 
  - "**/*.ts"
  - "**/*.tsx" 
  - "**/tsconfig*.json"
  - "**/package.json"
---

# TypeScript 5.x Comprehensive Rules

## Project Configuration

### Modern tsconfig.json Setup
- Use `"module": "nodenext"` for Node.js projects to support modern module resolution
- Set `"target": "ES2022"` or higher for modern JavaScript features
- Enable `"strict": true` for comprehensive type checking
- Use `"moduleResolution": "bundler"` for modern bundler compatibility
- Set `"skipLibCheck": true` for faster builds and to avoid third-party declaration file errors
- Configure `"lib"` array to match your target environment (e.g., `["ES2023", "DOM"]`)

```json
{
  "compilerOptions": {
    "target": "ES2023",
    "module": "nodenext", 
    "moduleResolution": "nodenext",
    "lib": ["ES2023", "DOM"],
    "strict": true,
    "skipLibCheck": true,
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true,
    "forceConsistentCasingInFileNames": true,
    "declaration": true,
    "outDir": "./dist",
    "rootDir": "./src"
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "**/*.test.ts"]
}
```

### Node.js Version Targeting
- **Node.js 24**: `"target": "ES2024"`, `"lib": ["ES2024"]`, `"module": "nodenext"`
- **Node.js 22**: `"target": "ES2023"`, `"lib": ["ES2023"]`, `"module": "nodenext"`  
- **Node.js 20**: `"target": "ES2023"`, `"lib": ["ES2023"]`, `"module": "nodenext"`
- **Node.js 18**: `"target": "ES2022"`, `"lib": ["ES2022"]`, `"module": "node16"`
- **Node.js 16**: `"target": "ES2021"`, `"lib": ["ES2021"]`, `"module": "node16"`

### Project Structure Optimization
- Use `"include"` and `"exclude" patterns instead of explicit `"files"` arrays for better performance
- Avoid mixing source files from different projects in the same directory
- Set `"maxNodeModuleJsDepth": 2` or higher if consuming complex JavaScript packages
- Use `"declarationDir"` separate from `"outDir"` for cleaner build outputs

## TypeScript 5.x Language Features

### Decorators (TypeScript 5.0+)
- Use the new standard decorators syntax (not experimental decorators)
- Apply decorators from bottom to top order
- Use `ClassMethodDecoratorContext` for method decorators with proper typing
- Implement decorator factories for parameterized decorators

```typescript
function logged<T extends (...args: any[]) => any>(
  originalMethod: T,
  context: ClassMethodDecoratorContext
): T {
  const methodName = String(context.name);
  
  function replacementMethod(this: any, ...args: any[]) {
    console.log(`Calling ${methodName}`);
    const result = originalMethod.call(this, ...args);
    console.log(`Called ${methodName}`);
    return result;
  }
  
  return replacementMethod as T;
}

class MyClass {
  @logged
  greet(name: string) {
    console.log(`Hello, ${name}!`);
  }
}
```

### const Type Parameters (TypeScript 5.0+)
- Use `const` type parameters for better type inference with literal types
- Apply to functions that should preserve exact literal types

```typescript
function identity<const T>(value: T): T {
  return value;
}

// Type is ["a", "b", "c"] not string[]
const result = identity(["a", "b", "c"] as const);
```

### Satisfies Operator Usage (TypeScript 5.0+)
- Use `satisfies` to validate types without type assertion
- Preserve exact object types while ensuring type compatibility
- Available in JSDoc with `@satisfies` tag for JavaScript files

```typescript
type Colors = "red" | "green" | "blue";

const palette = {
  red: "#ff0000",
  green: "#00ff00", 
  blue: "#0000ff",
  // typo would be caught:
  // "reed": "#ff0000" // Error!
} satisfies Record<Colors, string>;

// palette.red is still string, not Colors
```

### Resource Management with `using` Declarations (TypeScript 5.2+)
- Use `using` declarations for automatic resource cleanup
- Implement `Symbol.dispose` or `Symbol.asyncDispose` in resource classes
- Automatically disposes resources at scope exit

```typescript
class DatabaseConnection {
  [Symbol.dispose]() {
    console.log("Closing database connection");
    // cleanup logic
  }
}

function processData() {
  using db = new DatabaseConnection();
  // db is automatically disposed when scope exits
  return db.query("SELECT * FROM users");
}
```

### Import Attributes (TypeScript 5.3+)
- Use `with { type: 'json' }` instead of deprecated `assert` syntax
- Support for JSON, CSS, and other non-JS assets
- Works with both static and dynamic imports

```typescript
// Static imports
import config from './config.json' with { type: 'json' };
import styles from './styles.css' with { type: 'css' };

// Dynamic imports
const data = await import('./data.json', { with: { type: 'json' } });
```

### Type Predicates Inference (TypeScript 5.5+)
- Automatic type predicate inference in filter callbacks
- Eliminates need for explicit type guards in many cases
- Works with array methods and custom filter functions

```typescript
const items: (string | number)[] = ["a", 1, "b", 2];

// TypeScript 5.5+ automatically infers type predicate
const strings = items.filter((item): item is string => typeof item === "string");
// strings is now string[], no explicit type guard needed

// Works with reusable functions
function isString(value: unknown): value is string {
  return typeof value === "string";
}

const onlyStrings = items.filter(isString); // Inferred as string[]
```

### Iterator Helpers (TypeScript 5.6+)
- Use new iterator methods: `map()`, `filter()`, `take()`, `drop()`, `forEach()`
- Extend from `Iterator<T>` for custom iterators
- Adapt existing iterables with `Iterator.from()`

```typescript
function* numbers() {
  let i = 1;
  while (true) yield i++;
}

const evenNumbers = numbers()
  .filter(x => x % 2 === 0)
  .take(5)
  .map(x => x * 2);

for (const num of evenNumbers) {
  console.log(num); // 4, 8, 12, 16, 20
}
```

## Module System and Imports

### Module Resolution Strategy
- Use `"moduleResolution": "bundler"` for modern bundlers (Webpack, Vite, etc.)
- Use `"moduleResolution": "nodenext"` for Node.js environments
- Avoid deprecated `"moduleResolution": "node"` in new projects

### Import/Export Best Practices
- Use type-only imports with `import type` for better tree-shaking
- Leverage `export type * as namespace` syntax for re-exports
- Use `verbatimModuleSyntax: true` for stricter module handling

```typescript
// Type-only imports
import type { User, Config } from './types';
import { validateUser } from './validation';

// Type-only re-exports
export type * as Models from './models';
export { validateUser };
```

### ESM/CommonJS Interoperability
- Configure package.json `"type": "module"` for ESM projects
- Use conditional exports for dual-package support
- Handle Node.js 22+ `require()` of ESM modules

```json
{
  "type": "module",
  "exports": {
    ".": {
      "import": "./dist/index.js",
      "require": "./dist/index.cjs",
      "types": "./dist/index.d.ts"
    }
  }
}
```

## Performance and Build Optimization

### Compiler Performance
- Enable `"skipLibCheck": true` to skip checking all declaration files
- Use `"incremental": true` and `"tsBuildInfoFile"` for faster subsequent builds
- Set `"assumeChangesOnlyAffectDirectDependencies": true` for large projects
- Use project references for monorepos with `"references"` array

### Advanced Compiler Flags (TypeScript 5.x)
- `--moduleDetection auto|legacy|force`: Control how files are detected as modules vs scripts
- `--noConstantBinaryExpression`: Catch always-truthy/falsy expressions (TypeScript 5.6+)
- `--stopOnBuildErrors`: Halt multi-project builds on first error in build mode
- `--exactOptionalPropertyTypes`: Stricter optional property handling
- `--noUncheckedIndexedAccess`: Add `undefined` to index access types for safety
- `--verbatimModuleSyntax`: Preserve import/export syntax without transformation

```json
{
  "compilerOptions": {
    "moduleDetection": "auto",
    "noConstantBinaryExpression": true,
    "exactOptionalPropertyTypes": true,
    "noUncheckedIndexedAccess": true,
    "verbatimModuleSyntax": true,
    "noImplicitOverride": true
  }
}
```

## Type System Best Practices

### Strict Mode Configuration
- Enable all strict flags: `"strict": true` includes:
  - `"noImplicitAny": true`
  - `"strictNullChecks": true` 
  - `"strictFunctionTypes": true`
  - `"strictBindCallApply": true`
  - `"strictPropertyInitialization": true`
  - `"noImplicitReturns": true`
  - `"noImplicitThis": true`
  - `"alwaysStrict": true`

### Advanced Type Patterns
- Use template literal types for string manipulation
- Leverage conditional types with `infer` keyword
- Apply mapped types for object transformations
- Utilize utility types: `Pick`, `Omit`, `Partial`, `Required`, `Record`, `NoInfer` (5.4+)
- Use `NoInfer<T>` to prevent unwanted type inference in specific positions

```typescript
// Template literal types
type EventName<T extends string> = `on${Capitalize<T>}`;

// Conditional types with infer
type ReturnType<T> = T extends (...args: any[]) => infer R ? R : never;

// NoInfer utility (TypeScript 5.4+)
function createStreetMap<T extends string>(
  locations: T[],
  description: NoInfer<T> // Prevents inference from this parameter
): Map<T, string> {
  return new Map(locations.map(loc => [loc, description]));
}

// Mapped types
type Readonly<T> = {
  readonly [P in keyof T]: T[P];
};
```

### Union and Intersection Types
- Use discriminated unions for type safety
- Apply intersection types for composition
- Leverage type guards for narrowing

```typescript
// Discriminated union
type Shape = 
  | { kind: 'circle'; radius: number }
  | { kind: 'rectangle'; width: number; height: number };

function getArea(shape: Shape): number {
  switch (shape.kind) {
    case 'circle':
      return Math.PI * shape.radius ** 2;
    case 'rectangle':
      return shape.width * shape.height;
  }
}
```

## Known Issues and Mitigations

### Common Compiler Errors
- **TS2550**: Update `"lib"` array to include required ECMAScript features
- **TS2583**: Add missing library to `"lib"` (e.g., ES2015+ features)
- **TS1295**: Adjust `"verbatimModuleSyntax"` or package.json `"type"` field
- **TS2354**: Install `tslib` for helper functions when using `"importHelpers": true`

### Library Configuration Issues
- Missing ES2015+ features: Add `"es2015"` or later to `"lib"` array
- BigInt usage: Require `"lib": ["es2020"]` or later
- Iterator methods: Need `"lib": ["esnext"]` for latest features
- Async/await: Require `"lib": ["es2017"]` minimum

### Module Resolution Problems
- Use `"moduleResolution": "bundler"` for modern tooling
- Set `"allowSyntheticDefaultImports": true` for better interop
- Configure `"esModuleInterop": true` for CommonJS compatibility
- Add `"resolveJsonModule": true` for JSON imports

## Migration and Breaking Changes

### TypeScript 5.0+ Breaking Changes
- **Removed in TypeScript 5.0+**:
  - `"importsNotUsedAsValues"`: Use `"verbatimModuleSyntax"` instead
  - `"preserveValueImports"`: Use `"verbatimModuleSyntax"` instead  
  - Some legacy `--target` values: Upgrade to modern ES targets
  - Minimum Node.js requirement raised to v12.20

### TypeScript 5.3+ Import Syntax Migration
- **Import Assertions → Import Attributes**: Replace `assert { type: 'json' }` with `with { type: 'json' }`
- Update build tools and loaders to support new syntax
- Use `resolution-mode` in import types for better module resolution control

```typescript
// Old (deprecated)
import data from './data.json' assert { type: 'json' };

// New (TypeScript 5.3+)
import data from './data.json' with { type: 'json' };
```

### Decorator Migration (TypeScript 5.0+)
- Replace experimental decorators with standard decorators
- Update to use `ClassMethodDecoratorContext` and related context objects
- Migrate metadata usage to new decorator metadata API

```typescript
// Old experimental decorators
function oldDecorator(target: any, key: string) {
  // legacy implementation
}

// New standard decorators (TypeScript 5.0+)
function newDecorator<T extends (...args: any[]) => any>(
  originalMethod: T,
  context: ClassMethodDecoratorContext
): T {
  // modern implementation with context
  return originalMethod;
}
```

### Enum Behavior Changes (TypeScript 5.0+)
- Mixed string/number enums now properly error
- Out-of-domain literal assignments to enums are now caught
- Review enums for consistent typing patterns

### Migration Strategy
- Use `"ignoreDeprecations": "5.0"` to silence warnings during transition
- Upgrade incrementally: 4.x → 5.0 → 5.1 → etc.
- Test thoroughly with `--strict` mode enabled
- Update CI/CD pipelines to use compatible Node.js versions

## Integration with Build Tools

### Popular Framework Configuration

#### Next.js
```json
{
  "compilerOptions": {
    "target": "es5",
    "lib": ["dom", "dom.iterable", "es6"],
    "allowJs": true,
    "skipLibCheck": true,
    "strict": true,
    "forceConsistentCasingInFileNames": true,
    "noEmit": true,
    "esModuleInterop": true,
    "module": "esnext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "preserve",
    "incremental": true,
    "plugins": [{"name": "next"}],
    "paths": {"@/*": ["./src/*"]}
  }
}
```

#### Vite/Vitest
```json
{
  "compilerOptions": {
    "target": "ES2020",
    "useDefineForClassFields": true,
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "skipLibCheck": true,
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "jsx": "react-jsx",
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true
  }
}
```

### Node.js with TypeScript 5.8+
- Use `--erasableSyntaxOnly` flag for Node.js `--experimental-strip-types` compatibility
- Enable `--libReplacement false` to disable lib replacement lookup if not used
- Configure for dual ESM/CommonJS support in package.json exports

## Development Workflow

### Editor Configuration
- Use TypeScript 5.6+ for improved error reporting and diagnostics
- Enable region-based diagnostics for faster feedback
- Configure auto-imports with proper exclude patterns
- Set up commit characters for better autocomplete

### Debugging and Analysis  
- Use `--noEmit` for type-checking only builds
- Enable `--diagnostics` for build performance analysis
- Use `--listFiles` to debug file inclusion
- Configure `--generateTrace` for advanced performance profiling

### Testing Integration
- Configure separate tsconfig for tests: `tsconfig.test.json`
- Use `"types": ["node", "jest"]` or similar for test frameworks
- Enable `"allowJs": true` for gradual migration of test files
- Set up path mapping for test utilities

## Security and Best Practices

### Type Safety
- Always use `"strict": true` in production code
- Enable `"noUncheckedIndexedAccess": true` for safer array/object access
- Use `"exactOptionalPropertyTypes": true` for stricter optional properties
- Configure `"noImplicitOverride": true` for explicit method overrides

### Code Quality
- Enable `"noUnusedLocals": true` and `"noUnusedParameters": true`
- Use `"noFallthroughCasesInSwitch": true` for safer switch statements
- Configure `"noImplicitReturns": true` for function completeness
- Set `"allowUnreachableCode": false` to catch dead code

### Performance Monitoring
- Use `"generateCpuProfile": true` for build performance analysis
- Monitor memory usage with `--diagnostics`
- Profile type checking with `--generateTrace`
- Optimize with `"assumeChangesOnlyAffectDirectDependencies": true`

### Version-Specific Highlights

### TypeScript 5.9 (Latest - 2024)
- Enhanced type inference for complex generic scenarios
- Improved Node.js compatibility and ESM/CommonJS interoperability
- Better build performance optimizations for large projects
- Preparation features for TypeScript 6.0/7.0 transition
- Enhanced error reporting and diagnostics

### TypeScript 5.8 (February 2025)
- `--erasableSyntaxOnly` flag for Node.js `--experimental-strip-types` compatibility  
- `--libReplacement` flag for performance optimization when not using lib replacements
- Improved path normalization performance (significant speedup for large projects)
- Better CommonJS/ESM interoperability support
- Enhanced branch checking in return expressions

### TypeScript 5.7 (Late 2024)
- Incremental support for recursive conditional types
- Improved project finder heuristics in VS Code integration
- Refinement of path mapping behaviors for complex project structures
- Enhanced auto-import functionality and organization

### TypeScript 5.6 (September 2024)
- Iterator helper methods support (`map`, `filter`, `take`, `drop`, etc.)
- Improved error reporting with region-based diagnostics for faster feedback
- `--stopOnBuildErrors` flag for stricter build mode behavior
- Better truthy/falsy expression checking with `--noConstantBinaryExpression`
- Enhanced auto-completion with commit character support

### TypeScript 5.5 (June 2024)
- Automatic type predicate inference in filter callbacks
- Control-flow narrowing for constant-indexed property access
- JSDoc `@import` tag support for better JavaScript integration
- `${configDir}` template variable for flexible path configuration

### TypeScript 5.4 (March 2024)
- `NoInfer<T>` utility type for controlling generic inference
- Preserved closure narrowing for better type flow in callbacks
- Standard library updates for `Object.groupBy` and `Map.groupBy`
- Improved `require()` support in bundler resolution mode

### TypeScript 5.3 (November 2023)
- Import attributes syntax (`with { type: 'json' }`)
- `resolution-mode` support in import types and reference directives
- Advanced type narrowing with `switch(true)` and boolean comparisons
- Enhanced `instanceof` narrowing via `Symbol.hasInstance`

### TypeScript 5.2 (August 2023)
- `using` declarations for explicit resource management
- Decorator metadata support for runtime type information
- Named and anonymous tuple elements flexibility
- Enhanced tuple label preservation through spreads

### TypeScript 5.1 (June 2023)
- Implicit `undefined` return types for cleaner function signatures
- Unrelated getter/setter types with explicit annotations
- Decoupled JSX element checking for async server components
- Namespaced JSX attributes and linked cursor editing

### TypeScript 5.0 (March 2023)
- Standard ECMAScript decorators implementation
- `const` type parameters for literal type preservation
- `--moduleResolution bundler` for modern build tool compatibility
- `--verbatimModuleSyntax` for ESM preservation
- Significant package size reduction (26.4 MB smaller)