---
trigger: glob
description: Comprehensive Vite build tool rules covering configuration, optimization, and best practices for v6.0+ and latest standards (2024-2025)
globs: ["vite.config.*", "**/*.html", "**/package.json", "**/.env*"]
---

# Vite Rules

## Project Setup and Configuration

- **Use consistent project scaffolding**: Always use `npm create vite@latest` or equivalent with modern package managers (pnpm, yarn, bun)
- **Specify exact template**: Use `--template` flag for non-interactive setup: `npm create vite@latest my-app -- --template vue-ts`
- **Configure root correctly**: Set `root` option in config when entry point is not in current directory
- **Use absolute paths for reliability**: Always use absolute paths in file operations and avoid relative paths that depend on current working directory
- **Configure base path for deployment**: Set `base` option when deploying to subdirectories: `base: '/my-app/'`

```javascript
// vite.config.js - Basic setup
import { defineConfig } from 'vite'

export default defineConfig({
  root: './src',
  base: '/my-app/', // for subdirectory deployment
  publicDir: '../public'
})
```

## Development Server Configuration

- **Use specific port configuration**: Set `server.port` to avoid conflicts and ensure consistency across team
- **Configure host for external access**: Use `server.host: true` or `server.host: '0.0.0.0'` for network access
- **Enable CORS when needed**: Set `server.cors: true` for cross-origin requests during development
- **Configure proxy for API calls**: Use `server.proxy` to avoid CORS issues with backend services
- **Set up SSL for HTTPS development**: Use `server.https` with proper certificates when required

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

## Build Optimization

- **Target modern browsers only**: Set `build.target: 'esnext'` if legacy support not required
- **Disable sourcemaps in production**: Set `build.sourcemap: false` for faster builds and smaller artifacts
- **Use esbuild for minification**: Keep default `build.minify: 'esbuild'` for fastest performance
- **Remove console logs in production**: Configure terser to drop console statements
- **Optimize chunk splitting**: Use `build.rollupOptions.output.manualChunks` for better caching

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

## Asset Handling

- **Use proper import syntax for assets**: Import assets with explicit query parameters when needed
- **Configure asset inline limits**: Set `build.assetsInlineLimit` based on performance requirements
- **Use URL imports for asset paths**: Use `new URL('./asset.png', import.meta.url)` for dynamic asset loading
- **Handle static assets correctly**: Place static assets in `public/` directory for direct access
- **Use asset query parameters**: Leverage `?raw`, `?url`, `?inline` suffixes for specific asset handling

```javascript
// Asset handling examples
import logoUrl from './logo.svg?url'          // Gets URL to asset
import logoRaw from './logo.svg?raw'          // Gets raw content
import logoInline from './logo.svg?inline'    // Inlines as data URL

// Dynamic imports
const getAsset = (name) => new URL(`./assets/${name}`, import.meta.url)
```

## Environment Variables and Configuration

- **Use VITE_ prefix for client-side variables**: Only variables with `VITE_` prefix are exposed to client
- **Validate environment variables**: Check required env vars at build time
- **Use different configs per environment**: Leverage `.env`, `.env.local`, `.env.production` files
- **Secure sensitive variables**: Never expose API keys or secrets with `VITE_` prefix
- **Use import.meta.env correctly**: Access environment variables through `import.meta.env` object

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

## Plugin Configuration

- **Audit plugin usage regularly**: Remove unused plugins to improve build performance
- **Apply plugins conditionally**: Use plugins only in relevant environments
- **Order plugins correctly**: Place plugins in proper order for dependencies
- **Configure plugin options properly**: Use framework-specific plugin configurations
- **Avoid plugin conflicts**: Ensure plugins don't duplicate functionality

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

## Import and Module Handling

- **Use path aliases sparingly**: Avoid complex aliasing that slows resolution
- **Avoid barrel files**: Don't use index.js files that re-export many modules
- **Use dynamic imports for code splitting**: Implement route-based and component-based splitting
- **Handle JSON imports correctly**: Use import assertions for JSON when required
- **Optimize dependency pre-bundling**: Configure `optimizeDeps` for better performance

```javascript
// Path aliases configuration
export default defineConfig({
  resolve: {
    alias: {
      '@': path.resolve(__dirname, 'src'),
      '@components': path.resolve(__dirname, 'src/components')
    }
  },
  optimizeDeps: {
    include: ['lodash-es', 'date-fns'],
    exclude: ['@my-lib']
  }
})
```

## Worker and Web Worker Support

- **Use proper worker import syntax**: Use `?worker` suffix for worker imports
- **Configure worker handling**: Set up proper worker configuration for different types
- **Handle worker assets correctly**: Ensure workers can access required assets
- **Use shared workers appropriately**: Use `?sharedworker` for shared worker instances
- **Handle worker module types**: Configure module vs classic worker types correctly

```javascript
// Worker import patterns
import MyWorker from './worker.js?worker'
import MySharedWorker from './shared-worker.js?sharedworker'

// Worker with URL constructor
const worker = new Worker(new URL('./worker.js', import.meta.url))
```

## CSS and Styling

- **Use CSS modules correctly**: Configure CSS modules with proper naming patterns
- **Install CSS preprocessors**: Add sass, less, or stylus dependencies when needed
- **Configure PostCSS properly**: Set up PostCSS plugins in correct order
- **Handle CSS imports efficiently**: Use proper CSS import strategies for performance
- **Configure CSS code splitting**: Enable/disable CSS code splitting based on needs

```javascript
// CSS configuration
export default defineConfig({
  css: {
    modules: {
      localsConvention: 'camelCase',
      generateScopedName: '[name]__[local]___[hash:base64:5]'
    },
    preprocessorOptions: {
      scss: {
        additionalData: `@import "@/styles/variables.scss";`
      }
    }
  }
})
```

## Performance Monitoring and Debugging

- **Use profiling for performance issues**: Run `vite --profile --open` to identify bottlenecks
- **Monitor bundle size**: Use bundle analyzer plugins to track size changes
- **Optimize development startup**: Configure `server.warmup` for faster development starts
- **Handle file descriptor limits**: Increase limits on Linux systems for large projects
- **Debug import resolution**: Use `vite --debug` flag for import resolution issues

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

## Known Issues and Mitigations

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

## Security Considerations

- **Validate file access**: Use `server.fs.allow` to restrict file system access
- **Secure environment variables**: Never expose secrets through VITE_ prefixed variables
- **Configure CORS properly**: Set appropriate CORS policies for production
- **Handle user uploads safely**: Validate and sanitize any user-provided file paths
- **Use secure headers**: Configure security headers for production deployment

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

## Framework-Specific Patterns

- **React**: Use `@vitejs/plugin-react` for React support with SWC/Babel options
- **Vue**: Use `@vitejs/plugin-vue` with proper SFC configuration
- **Svelte**: Use `@sveltejs/vite-plugin-svelte` with correct preprocessing
- **Solid**: Use `vite-plugin-solid` for SolidJS projects
- **Preact**: Use `@preact/preset-vite` for Preact optimization

## Deployment and Production

- **Generate manifest for backend integration**: Enable `build.manifest` for server-side rendering
- **Configure public path correctly**: Set proper `base` for CDN or subdirectory deployment
- **Handle static assets**: Use proper asset referencing for different deployment scenarios
- **Configure build output**: Set appropriate `build.outDir` and `build.assetsDir`
- **Enable gzip/brotli**: Configure server compression for better performance

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

## Version Compatibility

- **Node.js support**: Use Node.js 18+, 20+, or 22+ (Node.js 21 deprecated in Vite 6.0+)
- **Update dependencies regularly**: Keep Vite and plugins updated for security and performance
- **Check breaking changes**: Review migration guides when updating major versions
- **Use compatible plugin versions**: Ensure plugins support your Vite version
- **Test across environments**: Validate builds work in target deployment environments