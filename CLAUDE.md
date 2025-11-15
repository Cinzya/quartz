# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Quartz is a static site generator for publishing digital gardens and notes as websites. It transforms Markdown files into a fully-featured website with advanced features like transclusion, wikilinks, graph views, and full-text search.

- Built with TypeScript, Preact, and the unified/remark/rehype ecosystem
- Plugin-based architecture for content transformation
- Supports incremental builds with watch mode
- SSG with client-side SPA routing for fast navigation

## Common Commands

### Build and Development
```bash
# Build the site (static HTML output to public/)
npx quartz build

# Build with live server and watch mode
npx quartz build --serve

# Build documentation site
npm run docs

# Build with custom concurrency
npx quartz build --concurrency=4
```

### Code Quality
```bash
# Type check (no emit)
npm run check

# Format code
npm run format

# Run tests
npm test

# Run specific test file
npx tsx --test quartz/util/path.test.ts
```

### CLI Commands
```bash
# Initialize new Quartz project
npx quartz create

# Update Quartz to latest version
npx quartz update

# Sync content to/from GitHub
npx quartz sync

# Restore content from cache
npx quartz restore
```

## Architecture

### Three-Phase Build Pipeline

The build process (`quartz/build.ts`) follows a strict three-phase architecture:

1. **PARSE** (`quartz/processors/parse.ts`)
   - Reads markdown files as VFiles
   - Applies transformer plugins in sequence:
     - `textTransform()` - raw text preprocessing
     - `markdownPlugins()` - remark plugins (MDAST manipulation)
     - `htmlPlugins()` - rehype plugins (HAST manipulation)
   - Uses worker threads for parallelization (chunks of 128 files)
   - Output: `ProcessedContent[]` = array of `[HtmlRoot, VFile]` tuples

2. **FILTER** (`quartz/processors/filter.ts`)
   - Sequentially applies filter plugins
   - Each `shouldPublish()` determines content inclusion
   - Example: RemoveDrafts filters `frontmatter.draft === true`

3. **EMIT** (`quartz/processors/emit.ts`)
   - Parallel execution of emitter plugins
   - Each emitter processes all filtered content
   - Returns file paths written or async generators for streaming
   - Supports `partialEmit()` for incremental builds

### Plugin System

**Three Plugin Types:**

1. **Transformers** (`quartz/plugins/transformers/`)
   - Transform markdown/HTML AST during parsing
   - Required: `name` field
   - Optional: `textTransform()`, `markdownPlugins()`, `htmlPlugins()`, `externalResources()`
   - Use unified/remark/rehype plugins for AST manipulation
   - Examples: GFM, ObsidianFlavoredMarkdown, Latex, SyntaxHighlighting, CrawlLinks

2. **Filters** (`quartz/plugins/filters/`)
   - Required: `name`, `shouldPublish(ctx, content): boolean`
   - Applied sequentially after parsing
   - Examples: RemoveDrafts, ExplicitPublish

3. **Emitters** (`quartz/plugins/emitters/`)
   - Required: `name`, `emit()`, `getQuartzComponents()`
   - Optional: `partialEmit()` for incremental builds
   - Generate output files (HTML pages, assets, indexes, etc.)
   - Examples: ContentPage, FolderPage, TagPage, Assets, ComponentResources

**Creating New Plugins:**
- Define plugin function that takes options and returns plugin instance
- Re-export in `quartz/plugins/{transformers,filters,emitters}/index.ts`
- Use `unified` ecosystem for AST transformations
- Declare TypeScript module augmentations for custom VFile data fields

### Component System

**Component Architecture:**
- Built with Preact for isomorphic rendering
- Components are TypeScript files (not JSX), typed as `QuartzComponent`
- Can declare `css`, `beforeDOMLoaded`, `afterDOMLoaded` resources
- Layout defined in `quartz.layout.ts` with slots: `head`, `header`, `beforeBody`, `pageBody`, `afterBody`, `left`, `right`, `footer`

**Component Locations:**
- `quartz/components/` - UI components
- `quartz/components/pages/` - Page-type components
- `quartz/components/scripts/` - Client-side JavaScript (inline and deferred)
- `quartz/components/styles/` - Component-specific styles

**Rendering:**
- Server-side: `renderPage()` in `quartz/components/renderPage.tsx`
- Supports transclusion (embed other page content)
- Detects circular transclusions
- Uses `preact-render-to-string` for SSR

### Content Representation

**Data Flow:**
```
File (string)
  → VFile (virtual file with metadata)
    → VFile.data extended with QuartzPluginData:
      - filePath: FilePath (absolute)
      - relativePath: FilePath (relative to content root)
      - slug: FullSlug (URL-safe identifier)
      - frontmatter: parsed YAML
      - htmlAst: HAST for transclusion
      - blocks: named sections for block-level transclusion
    → ProcessedContent = [HtmlRoot (HAST), VFile]
```

**Key Path Types** (branded strings in `quartz/util/path.ts`):
- `FilePath`: absolute, non-relative, with extension
- `FullSlug`: no leading/trailing slashes, no forbidden chars, no index suffix
- `SimpleSlug`: folder-like, no extension
- `RelativeURL`: starts with `.` or `..`

### Configuration Files

**quartz.config.ts:**
- Global configuration: page title, theme, analytics, locale, ignore patterns
- Plugin configuration: transformers, filters, emitters
- Theme: typography, colors (light/dark mode)

**quartz.layout.ts:**
- Component layouts for different page types
- `sharedPageComponents`: components across all pages (head, header, footer)
- `defaultContentPageLayout`: single note pages
- `defaultListPageLayout`: tag/folder listing pages

### Watch Mode & Incremental Builds

When `--watch` flag is used:
- File changes detected via chokidar
- Tracks changes in `ContentMap` (Map<FilePath, ParsedContent>)
- Only reparses changed markdown files
- Calls `partialEmit()` on emitters (falls back to full `emit()`)
- WebSocket sends reload signal to browser

### Worker Threading

- Worker thread pool for parsing (default: 1-4 workers based on file count)
- Concurrency heuristic: `clamp(fileCount / 128, 1, 4)`
- Chunks files (128 files/chunk) for worker distribution
- Uses `workerpool` library

## Key Directories

```
quartz/
  cli/                  # CLI handlers and argument parsing
  components/           # Preact UI components
  plugins/              # Plugin system (transformers, filters, emitters)
  processors/           # Core processing pipeline (parse, filter, emit)
  util/                 # Utilities (path, logging, themes, resources)
  i18n/                 # Internationalization (30+ locales)
  styles/               # Global SCSS styles
  static/               # Static assets

content/                # User content (markdown files)
docs/                   # Quartz documentation
public/                 # Build output
```

## Testing

- Uses Node.js built-in test runner (`node:test`)
- Run with: `npm test` or `npx tsx --test`
- Test files: `**/*.test.ts`
- Use `describe()` and `test()` from `node:test`
- Use `assert` from `node:assert`

## Important Development Notes

1. **TypeScript Configuration:**
   - JSX pragma: `preact` (`jsxImportSource: "preact"`)
   - Module system: ESNext with ES modules
   - Strict mode enabled
   - Target: ESNext

2. **Path Handling:**
   - Complex slug/path system with branded string types
   - Use utilities in `quartz/util/path.ts` for path transformations
   - Three link resolution strategies: absolute, shortest, relative

3. **AST Manipulation:**
   - Use `unist-util-visit` for traversing AST nodes
   - Use `mdast-util-find-and-replace` for text replacements
   - MDAST for Markdown, HAST for HTML

4. **Adding External Resources:**
   - CSS/JS resources declared via `externalResources()` in plugins
   - Can be inline content or external URLs
   - Load time: `beforeDOMReady` or `afterDOMReady`

5. **Content Processing:**
   - VFile stores metadata in `file.data`
   - Frontmatter parsed by FrontMatter transformer
   - Slugs generated from file paths with special character handling

6. **Client-Side Features:**
   - SPA routing (prevents full page reloads)
   - Full-text search (FlexSearch)
   - Interactive graph (D3)
   - Popover previews
   - Dark mode
   - Mermaid diagrams

## Common Patterns

### Creating a Transformer Plugin
```typescript
import { QuartzTransformerPlugin } from "../types"

export const MyPlugin: QuartzTransformerPlugin = () => {
  return {
    name: "MyPlugin",
    markdownPlugins() {
      return [myRemarkPlugin]
    },
    htmlPlugins() {
      return [myRehypePlugin]
    },
    externalResources() {
      return {
        css: [{ content: "path/to/style.css" }],
      }
    },
  }
}

// Augment VFile data if adding custom metadata
declare module "vfile" {
  interface DataMap {
    myCustomField: string
  }
}
```

### Creating an Emitter Plugin
```typescript
import { QuartzEmitterPlugin } from "../types"
import { write } from "./helpers"

export const MyEmitter: QuartzEmitterPlugin = () => {
  return {
    name: "MyEmitter",
    getQuartzComponents() {
      return [/* components used */]
    },
    async emit(ctx, content, resources) {
      const fps: FilePath[] = []
      for (const [tree, file] of content) {
        const fp = await write({
          ctx,
          slug: file.data.slug!,
          ext: ".html",
          content: "...",
        })
        fps.push(fp)
      }
      return fps
    },
  }
}
```

### Creating a Component
```typescript
import { QuartzComponent, QuartzComponentConstructor, QuartzComponentProps } from "./types"

export default (() => {
  const MyComponent: QuartzComponent = ({ fileData, cfg }: QuartzComponentProps) => {
    return <div>...</div>
  }

  MyComponent.css = `/* styles */`
  MyComponent.afterDOMLoaded = `/* client script */`

  return MyComponent
}) satisfies QuartzComponentConstructor
```
