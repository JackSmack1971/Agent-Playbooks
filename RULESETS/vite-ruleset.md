---
trigger: ["glob"]
description: Comprehensive Vite build tool rules covering configuration, optimization, and best practices for v6.0+ and latest standards
globs: ["vite.config.*", "**/*.html", "**/package.json", "**/.env*"]
version: "6.0.0"
last_updated: "2025-01-01"
---

# Vite Rules

## Installation & Setup

- **USE** consistent project scaffolding with `npm create vite@latest`
- **SPECIFY** exact template with `--template` flag for non-interactive setup
- **CONFIGURE** root correctly when entry point is not in current directory
- **USE** absolute paths for reliability to avoid relative path dependencies
- **CONFIGURE** base path for subdirectory deployment

```bash
# Create new Vite project
npm create vite@latest my-app -- --template vue-ts

# Install dependencies
npm install

# Start development server
npm run dev
```

## Configuration & Initialization

### Basic Configuration

```javascript
// vite.config.js - Basic setup
import { defineConfig } from 'vite'

export default defineConfig({
  root: './src',
  base: '/my-app/', // for subdirectory deployment
  publicDir: '../public'
})
```

### Development Server Configuration

```javascript
// Development server configuration
export default defineConfig({
  server: {
    port: 3000,
    host: true, // expose to network
    cors: true,
    proxy: {
      '/api': {
        target: 'http://localhost:5000',
        changeOrigin: true,
        rewrite: (path) => path.replace(/^\/api/, '')
      }
    }
  }
})
```

## Core Concepts / API Usage

### Build Optimization

- **TARGET** modern browsers only with `build.target: 'esnext'` if legacy support not required
- **DISABLE** sourcemaps in production with `build.sourcemap: false` for faster builds
- **USE** esbuild for minification with `build.minify: 'esbuild'` for fastest performance
- **REMOVE** console logs in production with terser configuration
- **OPTIMIZE** chunk splitting with `build.rollupOptions.output.manualChunks`

```javascript
// Production optimization configuration
export default defineConfig({
  build: {
    target: 'esnext',
    sourcemap: false,
    minify: 'esbuild',
    assetsInlineLimit: 8192,
    rollupOptions: {
      output: {
        manualChunks(id) {
          if (id.includes('node_modules')) {
            return 'vendor'
          }
        }
      }
    }
  }
})
```

### Asset Handling

- **USE** proper import syntax for assets with explicit query parameters
- **CONFIGURE** asset inline limits based on performance requirements
- **USE** URL imports for dynamic asset loading with `new URL('./asset.png', import.meta.url)`
- **PLACE** static assets in `public/` directory for direct access
- **LEVERAGE** asset query parameters: `?raw`, `?url`, `?inline`

```javascript
// Asset handling examples
import logoUrl from './logo.svg?url'          // Gets URL to asset
import logoRaw from './logo.svg?raw'          // Gets raw content
import logoInline from './logo.svg?inline'    // Inlines as data URL

// Dynamic imports
const getAsset = (name) => new URL(`./assets/${name}`, import.meta.url)
```

## Security & Permissions

- **VALIDATE** file access using `server.fs.allow` to restrict file system access
- **SECURE** environment variables by never exposing secrets through VITE_ prefixed variables
- **CONFIGURE** CORS properly for production environments
- **HANDLE** user uploads safely with validation and sanitization
- **USE** secure headers for production deployment

```javascript
// Security configuration
export default defineConfig({
  server: {
    fs: {
      allow: ['..', '/allowed/path'],
      deny: ['.env', '.env.*']
    }
  }
})
```

## Performance & Scalability

### Environment Variables

- **USE** VITE_ prefix for client-side variables only
- **VALIDATE** required environment variables at build time
- **LEVERAGE** different configs per environment (`.env`, `.env.local`, `.env.production`)
- **SECURE** sensitive variables by avoiding VITE_ prefix for secrets
- **ACCESS** environment variables through `import.meta.env` object

```javascript
// Environment handling
if (!import.meta.env.VITE_API_URL) {
  throw new Error('VITE_API_URL is required')
}

// Type-safe environment variables (TypeScript)
interface ImportMetaEnv {
  readonly VITE_API_URL: string
  readonly VITE_APP_TITLE: string
}
```

### Plugin Configuration

- **AUDIT** plugin usage regularly to remove unused plugins
- **APPLY** plugins conditionally based on environment
- **ORDER** plugins correctly for proper dependencies
- **CONFIGURE** framework-specific plugin options
- **AVOID** plugin conflicts by ensuring no duplicate functionality

```javascript
// Conditional plugin loading
export default defineConfig(({ command, mode }) => ({
  plugins: [
    vue(),
    command === 'build' && legacy({
      targets: ['defaults', 'not IE 11']
    }),
    mode === 'development' && inspect()
  ].filter(Boolean)
}))
```

## Error Handling & Troubleshooting

### Known Issues and Mitigations

- **File descriptor limits on Linux**: Increase `ulimit -Sn` and inotify limits for large projects
- **SSL certificate caching issues**: Use trusted certificates to avoid Chrome caching problems
- **Request header size limits**: Use `--max-http-header-size` for large headers (431 errors)
- **Module externalization warnings**: Don't use Node.js modules in browser code
- **Case sensitivity issues**: Ensure consistent file naming across operating systems
- **Import resolution failures**: Verify file existence and correct path/extension usage

```bash
# Linux file descriptor fixes
ulimit -Sn 10000
sudo sysctl fs.inotify.max_queued_events=16384

# macOS SSL certificate fix
security add-trusted-cert -d -r trustRoot -k ~/Library/Keychains/login.keychain-db cert.cer
```

### Performance Monitoring

- **USE** profiling with `vite --profile --open` to identify bottlenecks
- **MONITOR** bundle size with analyzer plugins
- **OPTIMIZE** development startup with `server.warmup` configuration
- **HANDLE** file descriptor limits on Linux systems for large projects
- **DEBUG** import resolution with `vite --debug` flag

```javascript
// Performance optimization
export default defineConfig({
  server: {
    warmup: {
      clientFiles: ['./src/components/*.vue', './src/views/*.vue']
    }
  }
})
```

## Testing

### Testing Vite Applications

```javascript
// Example test setup with Vitest
import { describe, it, expect } from 'vitest'
import { render, screen } from '@testing-library/react'

describe('Vite App', () => {
  it('renders correctly', () => {
    render(<App />)
    expect(screen.getByText('Hello Vite!')).toBeInTheDocument()
  })
})
```

### Build Testing

- **TEST** builds in target deployment environments
- **VALIDATE** asset loading and path resolution
- **CHECK** bundle size and chunk splitting
- **VERIFY** environment variable handling
- **TEST** different build configurations

## Deployment & Production Patterns

### Production Configuration

```javascript
// Production deployment configuration
export default defineConfig({
  base: '/my-app/',
  build: {
    outDir: 'dist',
    assetsDir: 'assets',
    manifest: true, // for SSR
    rollupOptions: {
      input: {
        main: path.resolve(__dirname, 'index.html'),
        admin: path.resolve(__dirname, 'admin.html')
      }
    }
  }
})
```

### Framework-Specific Patterns

- **React**: Use `@vitejs/plugin-react` for React support with SWC/Babel options
- **Vue**: Use `@vitejs/plugin-vue` with proper SFC configuration
- **Svelte**: Use `@sveltejs/vite-plugin-svelte` with correct preprocessing
- **Solid**: Use `vite-plugin-solid` for SolidJS projects
- **Preact**: Use `@preact/preset-vite` for Preact optimization

## Version Compatibility Notes

- **Node.js support**: Use Node.js 18+, 20+, or 22+ (Node.js 21 deprecated in Vite 6.0+)
- **Update dependencies regularly**: Keep Vite and plugins updated for security and performance
- **Check breaking changes**: Review migration guides when updating major versions
- **Use compatible plugin versions**: Ensure plugins support your Vite version
- **Test across environments**: Validate builds work in target deployment environments

## References

- [Vite Official Documentation](https://vitejs.dev/)
- [Vite Configuration Reference](https://vitejs.dev/config/)
- [Vite Plugin API](https://vitejs.dev/guide/api-plugin.html)
- [Framework Integration Guides](https://vitejs.dev/guide/)
- [Migration Guides](https://vitejs.dev/guide/migration.html)