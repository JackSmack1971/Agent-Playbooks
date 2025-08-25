---
trigger: ["model_decision"]
description: Comprehensive ruleset for Docusaurus static site generator focusing on configuration, project structure, best practices, and common pitfall prevention
globs: ["**/docusaurus.config.{js,ts,mjs}", "**/sidebars.{js,ts,json}", "**/blog/**/*.{md,mdx}", "**/docs/**/*.{md,mdx}", "**/src/pages/**/*.{js,jsx,ts,tsx,md,mdx}", "**/src/components/**/*.{js,jsx,ts,tsx}", "**/src/theme/**/*.{js,jsx,ts,tsx}"]
version: "3.8.1"
last_updated: "2025-01-01"
---

# Docusaurus Rules

## Installation & Setup

- **MUST** use Node.js 18+ for Docusaurus v3+ compatibility
- **ALWAYS** initialize projects with `npx create-docusaurus@latest` for latest templates
- **SHOULD** use TypeScript configuration for better development experience

```bash
# Create new Docusaurus site
npx create-docusaurus@latest my-website classic

# Install dependencies
cd my-website && npm install

# Start development server
npm run start
```

## Configuration & Initialization

- Always use the ES module export syntax for modern Docusaurus configurations: `export default { ... }` instead of `module.exports = { ... }`
- Include essential configuration fields: `title`, `tagline`, `url`, `baseUrl`, `organizationName`, and `projectName`
- Use `@docusaurus/preset-classic` for standard documentation sites unless specific plugins are needed
- Set `url` to the full domain without trailing slash (e.g., `https://mysite.com`) and `baseUrl` with trailing slash (e.g., `/my-project/`)
- Use `trailingSlash: false` for GitHub Pages compatibility, `trailingSlash: true` for other hosts that add them automatically

```javascript
export default {
  title: 'My Site',
  tagline: 'Documentation site',
  url: 'https://mysite.com',
  baseUrl: '/',
  organizationName: 'my-org',
  projectName: 'my-project',
  trailingSlash: false,
  presets: [
    [
      '@docusaurus/preset-classic',
      {
        docs: {
          sidebarPath: './sidebars.js',
        },
        theme: {
          customCss: ['./src/css/custom.css'],
        },
      },
    ],
  ],
};
```

## Core Concepts / API Usage

- Follow the standard Docusaurus directory structure:
  ```
  my-website/
  ├── blog/                    # Blog posts (.md/.mdx files)
  ├── docs/                    # Documentation (.md/.mdx files)
  ├── src/
  │   ├── css/
  │   │   └── custom.css      # Global custom styles
  │   ├── components/         # Reusable React components
  │   ├── pages/              # Custom pages
  │   └── theme/              # Swizzled theme components
  ├── static/                 # Static assets (images, files)
  ├── docusaurus.config.js    # Main configuration
  ├── sidebars.js            # Sidebar configuration
  └── package.json
  ```
- Place documentation files in the `docs/` directory with `.md` or `.mdx` extensions
- Store blog posts in the `blog/` directory with date prefixes: `YYYY-MM-DD-post-name.md`
- Put static assets in the `static/` directory for direct serving
- Use the `src/components/` directory for reusable React components
- Place custom pages in `src/pages/` directory

## Plugin and Theme Configuration

- Use full package names for plugins and themes: `@docusaurus/plugin-content-blog` instead of shortcuts
- Configure multi-instance plugins with unique `id` values when using multiple blogs or docs
- Use the array syntax `[pluginName, options]` when passing configuration options
- Always specify required fields like `routeBasePath` and `path` for content plugins

```javascript
export default {
  plugins: [
    [
      '@docusaurus/plugin-content-blog',
      {
        id: 'second-blog',
        routeBasePath: 'engineering-blog',
        path: './engineering-blog',
      },
    ],
  ],
  themes: ['@docusaurus/theme-classic'],
};
```

## Security & Permissions

- Never commit sensitive data to `docusaurus.config.js`
- Use `customFields` to pass environment variables to client-side safely
- Validate environment variables with proper fallbacks
- Use HTTPS URLs for production `url` configuration
- Implement Content Security Policy headers if hosting allows

## Performance & Scalability

- Enable Webpack optimizations carefully - avoid `experimental_faster: true` in production until stable
- Optimize images before placing in `static/` directory
- Use code splitting for large React components with `React.lazy()`
- Minimize custom CSS and use CSS modules when possible
- Configure `onBrokenLinks: 'throw'` to catch broken internal links during build

## Error Handling & Troubleshooting

- Use the recommended ESLint configuration for Docusaurus projects
- Extend `plugin:@docusaurus/recommended` in your `.eslintrc.json`:

```json
{
  "extends": ["plugin:@docusaurus/recommended"]
}
```

## Testing

- Run `npm run build` locally before deploying to catch build errors
- Use `npm run serve` to test production builds locally
- Implement visual regression testing for theme customizations
- Check accessibility compliance, especially for custom components
- Test mobile responsiveness and performance on various devices
- Validate HTML output and check for broken links regularly

## Deployment & Production Patterns

- Set correct `url` and `baseUrl` for your hosting environment
- Configure `organizationName` and `projectName` for GitHub Pages deployment
- Use environment variables for sensitive configuration: `process.env.CUSTOM_FIELD`
- Set `deploymentBranch` explicitly if not using default `gh-pages`
- Configure `customFields` for passing environment variables to client-side

```javascript
export default {
  url: 'https://myusername.github.io',
  baseUrl: '/my-project/',
  organizationName: 'myusername',
  projectName: 'my-project',
  deploymentBranch: 'gh-pages',
  customFields: {
    teamEmail: process.env.TEAM_EMAIL,
  },
};
```

## MDX and Content Best Practices

- Use MDX format for enhanced content with React components
- Import React components at the top of MDX files: `import MyComponent from '@site/src/components/MyComponent';`
- Use JSX syntax properly in MDX - components must be capitalized
- Escape curly braces that aren't JSX expressions: `\{notJSX\}`
- Use proper frontmatter YAML syntax with `---` delimiters
- Include `id`, `title`, and `sidebar_position` in document frontmatter for better organization

```markdown
---
id: my-doc
title: My Document
sidebar_position: 1
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# My Document

<Tabs>
  <TabItem value="js" label="JavaScript">
    ```js
    console.log('Hello');
    ```
  </TabItem>
</Tabs>
```

## Sidebar Configuration

- Define sidebars in `sidebars.js` using the object structure: `{ tutorialSidebar: [...] }`
- Use autogenerated sidebars when possible: `{ type: 'autogenerated', dirName: '.' }`
- Specify custom sidebar items with `type`, `id`, and `label` properties
- Use `sidebar_position` frontmatter to control order in autogenerated sidebars
- Create category items with `type: 'category'` for nested navigation

```javascript
module.exports = {
  tutorialSidebar: [
    'intro',
    {
      type: 'category',
      label: 'Tutorial',
      items: ['tutorial-basics/create-a-page'],
    },
    {
      type: 'autogenerated',
      dirName: 'guides',
    },
  ],
};
```

## Versioning and Internationalization

- Use semantic versioning for docs: `npm run docusaurus docs:version 1.0.0`
- Store versioned docs in `versioned_docs/version-X.Y.Z/` directories
- Create locale-specific content in `i18n/[locale]/` directories
- Use `docusaurus write-translations` command to extract translatable strings
- Configure i18n in `docusaurus.config.js` with `defaultLocale` and `locales` arrays

```javascript
export default {
  i18n: {
    defaultLocale: 'en',
    locales: ['en', 'fr', 'es'],
  },
};
```

## Known Issues & Mitigations

- **Node.js ESM Import Issues**: With Node.js ≥20.19, verify plugin imports work correctly. Use dynamic imports for ESM-only packages:
```javascript
export default async function createConfigAsync() {
  const lib = await import('esm-only-lib');
  return {
    title: 'My Site',
    // rest of config
  };
}
```

- **MDX v3 Migration**: Run `npx docusaurus-mdx-checker` before upgrading to catch compatibility issues
- **Dynamic Tab Titles**: Override tab title behavior in custom CSS if automatic titles are problematic:
```css
/* Hide dynamic title updates */
head title {
  /* Custom title handling */
}
```

- **Dark Mode CSS Overrides**: Some CSS custom properties require `!important` in dark mode
```css
[data-theme='dark'] {
  --custom-property: value !important;
}
```

- **Build Performance**: If dev server is slow (>3 seconds), check for large assets in `static/` and optimize plugins
- **Hot Reload Issues**: Restart dev server if changes don't propagate; known issue with file watching

## Blog Configuration Best Practices

- Use date prefixes in blog filenames: `2024-01-01-post-name.md`
- Configure global authors in `blog/authors.yml` for consistency
- Set `routeBasePath: '/'` for blog-only sites and disable docs with `docs: false`
- Use `truncateMarker: '<!-- truncate -->'` for custom blog post previews
- Configure RSS feeds with proper site metadata

```yaml
# blog/authors.yml
john:
  name: John Doe
  title: Developer
  url: https://github.com/johndoe
  image_url: https://github.com/johndoe.png
  socials:
    x: johndoe
    github: johndoe
```

## Advanced Configuration Patterns

- Use async config functions for dynamic imports
- Implement custom webpack configurations cautiously
- Create custom plugins following the official plugin API
- Use swizzling sparingly and document customizations
- Implement proper TypeScript support with `@docusaurus/module-type-aliases`

```javascript
// Advanced async configuration
export default async function createConfigAsync() {
  const remarkPlugin = await import('remark-plugin');

  return {
    title: 'My Site',
    presets: [
      [
        '@docusaurus/preset-classic',
        {
          docs: {
            remarkPlugins: [remarkPlugin.default],
          },
        },
      ],
    ],
  };
}
```

## Version Compatibility Notes

- **Current version tested**: Docusaurus 3.8.1
- **Node.js compatibility**: Requires Node.js 18+
- **React compatibility**: Uses React 18+ features
- **Migration considerations**: Major breaking changes in v3.x series

## References

- [Docusaurus Official Documentation](https://docusaurus.io/)
- [Docusaurus GitHub Repository](https://github.com/facebook/docusaurus)
- [Community Resources](https://docusaurus.io/community)
- [Migration Guide](https://docusaurus.io/docs/migration)