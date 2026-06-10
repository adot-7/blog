## Quartz v5 Customization Guide

Quartz v5 is designed around a simple idea: content is transformed by plugins, plugins expose layout slots, and the site is assembled from a small number of reusable page-frame pieces. If you learn where those three layers meet, almost every customization becomes straightforward.

This guide is anchored to the actual codebase, not just the docs. For this workspace, the most important files are:

- `/home/adot/projects/akashparashar.dev/quartz/quartz.config.yaml`
- `/home/adot/projects/akashparashar.dev/quartz/quartz.ts`
- `/home/adot/projects/akashparashar.dev/quartz/quartz/cfg.ts`
- `/home/adot/projects/akashparashar.dev/quartz/quartz/plugins/loader/config-loader.ts`
- `/home/adot/projects/akashparashar.dev/quartz/quartz/styles/base.scss`
- `/home/adot/projects/akashparashar.dev/quartz/quartz/styles/variables.scss`

## 1. The mental model

Quartz v5 has three layers you should keep separate:

1. Site-wide configuration in `quartz.config.yaml`.
2. Plugin behavior and placement, also in `quartz.config.yaml`.
3. Rendering and responsive behavior in `quartz/` source files and SCSS.

In practice:

- `quartz.config.yaml` tells Quartz what to enable, what order to run plugins in, what options they get, and where their UI components should appear.
- `quartz/plugins/loader/config-loader.ts` reads that YAML, installs or resolves plugins, instantiates components, and builds the final page layout.
- `quartz/styles/base.scss` and `quartz/styles/variables.scss` decide how the assembled layout responds to desktop, tablet, and mobile widths.
- `quartz.ts` is the escape hatch for advanced programmatic overrides when YAML is not expressive enough.

If you remember one thing: YAML changes behavior by configuration; SCSS changes behavior by presentation; `quartz.ts` changes behavior by code.

## 2. How Quartz v5 boots and renders

The top-level `quartz.ts` file is minimal:

```ts
import { loadQuartzConfig, loadQuartzLayout } from "./quartz/plugins/loader/config-loader"

const config = await loadQuartzConfig()
export default config
export const layout = await loadQuartzLayout()
```

That tells you the real entry point is the loader. The loader:

- reads `quartz.config.yaml`
- installs or resolves plugins
- validates plugin dependencies and order
- instantiates component plugins
- builds the final page layout
- injects built-in structural pieces like `Head` and, if enabled, `Footer`

This is why Quartz feels highly customizable without becoming a hand-written React app. The site is still code, but most of the “what goes where” decisions are data-driven.

## 3. What you can customize in YAML

`quartz.config.yaml` is the primary place you should edit first.

### Global configuration

The `configuration` section controls site-wide settings such as:

- `pageTitle`
- `pageTitleSuffix`
- `enableSPA`
- `enablePopovers`
- `analytics`
- `baseUrl`
- `locale`
- `theme`

In your config, the most visible settings are the theme tokens:

```yaml
configuration:
	theme:
		typography:
			header: Schibsted Grotesk
			body: Atkinson Hyperlegible
			code: IBM Plex Mono
		colors:
			lightMode:
				light: "#faf8f8"
				dark: "#2b2b2b"
				secondary: "#284b63"
			darkMode:
				light: "#161618"
				dark: "#ebebec"
				secondary: "#7b97aa"
```

That is the right place to change the site’s overall visual identity.

### Plugin behavior

Each plugin entry can control:

- whether it is enabled
- its execution order
- its options
- its layout position
- whether it is hidden on mobile or desktop
- whether it is conditionally rendered

Example from your config:

```yaml
- source: github:quartz-community/graph
	enabled: true
	layout:
		position: right
		priority: 10
```

This means the graph is a component plugin placed in the right sidebar with a fairly high priority.

### Layout placement

Quartz v5 does not use a “drag and drop” editor. Instead, layout is declared in YAML.

The key positions are:

- `left`
- `right`
- `beforeBody`
- `afterBody`

Some plugins also support `display`:

- `all`
- `mobile-only`
- `desktop-only`

This is crucial for mobile behavior. If a component should disappear on mobile, the cleanest solution is usually `display: desktop-only`, not CSS hacks.

### Page frames

Quartz supports different page frames such as:

- `default`
- `full-width`
- `minimal`

Frames control the page shell, not the content itself. If a page type needs a different overall structure, you change its `template` in `layout.byPageType`.

## 4. Your current config, in plain English

Your current `quartz.config.yaml` already shows a pretty full Quartz build. The important parts are:

- the TOC plugin is enabled and placed on the right
- the graph plugin is enabled on the right
- explorer is enabled on the left
- search and dark mode are grouped into a toolbar on the left
- footer is enabled
- a custom theme is already defined

That means your site is not using Quartz in a minimal “content only” mode. It is using most of the classic sidebar components.

This matters because sidebars compete for space. If you move one component around, the mobile behavior changes based on the underlying grid and breakpoints, not just on the plugin itself.

## 5. Theme customization

If you want to change the overall look of the site, start in `configuration.theme`.

### Change typography

Quartz supports separate fonts for:

- headers
- body text
- code

If you want a different feel, change those values first. This is the lowest-risk visual customization.

### Change color palette

The theme colors map directly to CSS variables used across the site:

- `light` for page background
- `lightgray` for borders
- `gray` for graph links and heavier borders
- `darkgray` for body text
- `dark` for headers and icons
- `secondary` for links and graph accents
- `tertiary` for hover states and visited graph nodes
- `highlight` for background highlighting
- `textHighlight` for marked text

If the site feels too soft, too bright, or too “Quartz-default,” these are the values to change.

### What not to do

- Do not start by overriding random CSS selectors if the theme tokens can express the change.
- Do not change core layout because you want a color tweak.
- Do not edit plugin code just to recolor a component.

Use theme tokens first; use CSS only when the token system cannot express the look you want.

## 6. Graph customization

The graph is a community plugin, not hard-coded site chrome.

Your graph plugin entry is the correct place to customize it:

```yaml
- source: github:quartz-community/graph
	enabled: true
	options:
		localGraph:
			drag: true
			zoom: true
			depth: 1
			scale: 1.1
			repelForce: 0.5
			centerForce: 0.3
			linkDistance: 30
			fontSize: 0.6
			opacityScale: 1
			removeTags: []
			showTags: true
			enableRadial: false
		globalGraph:
			drag: true
			zoom: true
			depth: -1
			scale: 0.9
			repelForce: 0.5
			centerForce: 0.3
			linkDistance: 30
			fontSize: 0.6
			opacityScale: 1
			removeTags: []
			showTags: true
			focusOnHover: true
			enableRadial: true
```

The main levers are:

- `depth`: how much of the graph to show
- `scale`: overall sizing
- `repelForce` and `centerForce`: layout feel
- `linkDistance`: spacing between nodes
- `fontSize`: label readability
- `showTags` and `removeTags`: content filtering
- `enableRadial`: graph layout style

### To make the graph look different

Adjust the graph options first. If you need a deeper stylistic change, then move into custom CSS on the graph component classes. But for 90 percent of graph changes, the plugin options are enough.

### To remove the graph

Either remove the `graph` plugin entry or set `enabled: false`.

## 7. Footer customization

The footer is also just a plugin.

Your config includes:

```yaml
- source: github:quartz-community/footer
	enabled: true
	options:
		links:
			GitHub: https://github.com/adot-7
			X: https://x.com/itsakaashhh
```

### To remove the Quartz footer

Disable that plugin entry or remove it entirely. That is the clean way.

### To change the footer text or links

Edit the `links` map. That is what the plugin is built for.

### What not to do

- Do not hunt for footer text in the theme files first.
- Do not modify the loader unless you need a completely custom page shell.

## 8. The mobile TOC issue you described

This is the part that usually confuses people.

Quartz v5 does not ship with a default hamburger drawer for the sidebars. Instead, the page uses grid areas and sidebar wrappers that change at the mobile breakpoint.

The important responsive behavior is in `quartz/styles/base.scss`:

- the right sidebar is laid out as a column on desktop
- on non-desktop widths, the right sidebar becomes a row-like strip and the `.toc` element is hidden
- the left sidebar behaves differently on mobile and remains visible as a horizontal row

That means the TOC behaves differently depending on where you place it.

### Why your TOC expands on mobile when moved left

If you move the TOC from `right` to `left`, it is no longer affected by the mobile rule that hides `.toc` inside the right sidebar.

So on mobile:

- right sidebar TOC gets hidden by the stylesheet
- left sidebar TOC can still render and take up space

That is why it can feel like the TOC is “taking over” the mobile layout.

### The clean fix

If you do not want the TOC on mobile, do one of these:

1. Keep it in the right sidebar and leave the current responsive behavior alone.
2. Wrap the TOC in `desktop-only` so it never appears on mobile.
3. Add a custom CSS rule that hides the specific TOC container on mobile widths.

The best option is usually `display: desktop-only` in the plugin layout declaration.

Example:

```yaml
- source: github:quartz-community/table-of-contents
	enabled: true
	layout:
		position: right
		priority: 30
		display: desktop-only
```

If you want the TOC to stay hidden on mobile regardless of whether it is on the left or right, this is the safest path.

### If you want a hamburger menu

Quartz does not provide one as the default sidebar behavior. That would require a custom component or custom layout logic.

## 9. Explorer, search, and sidebar competition

Explorer, search, backlinks, graph, and TOC all compete for the same sidebar real estate.

That means the order and grouping matter.

In your config:

- explorer is on the left
- search is on the left and grouped into `toolbar`
- dark mode is also in that toolbar group
- graph and backlinks are on the right

If the sidebar becomes crowded on mobile, check whether a component should be:

- moved to `beforeBody`
- grouped into a toolbar
- marked `desktop-only`
- removed entirely from mobile

## 10. How `quartz.ts` fits in

Use `quartz.ts` when YAML is not enough.

Typical reasons:

- you need custom JavaScript callbacks
- you need programmatic layout overrides
- you need to force a different page frame for a page type
- you want to wrap components with custom logic

Example pattern:

```ts
import { loadQuartzConfig, loadQuartzLayout } from "./quartz/plugins/loader/config-loader"

const config = await loadQuartzConfig()
export default config
export const layout = await loadQuartzLayout({
  defaults: {
    // global layout overrides
  },
  byPageType: {
    content: {
      // content page overrides
    },
  },
})
```

### When to use `quartz.ts`

Use it for advanced composition.

### When not to use `quartz.ts`

Do not use it just to rename a title, change a color, or hide a plugin that already supports `enabled: false`.

## 11. What to change, and what not to change

### Change in `quartz.config.yaml`

- theme colors and typography
- plugin enabled/disabled state
- plugin order
- plugin options
- layout position
- display mode
- page-type layout overrides

### Change in SCSS only if needed

- custom responsive behavior not covered by YAML
- component-specific mobile hiding that YAML cannot express
- visual polish that theme tokens do not cover

### Change in `quartz.ts` only if needed

- custom component wrapping
- custom conditional logic
- advanced layout assembly
- nontrivial overrides that need code

### Avoid changing

- plugin internals unless you are maintaining the plugin itself
- loader internals unless you need a Quartz-wide behavior change
- CSS just to fix something that YAML can already express

## 12. Practical recipes

### Make the site feel more like your own

1. Change `pageTitle` and `pageTitleSuffix`.
2. Tune the theme typography.
3. Adjust the light/dark palette.
4. Replace or remove plugins you do not use.

### Make the graph less dominant

1. Reduce graph depth.
2. Lower opacity or scale.
3. Move it lower in the sidebar.
4. Hide it on mobile if necessary.

### Remove Quartz branding/footer

1. Disable the footer plugin.
2. Remove any footer links you do not want.
3. Check for plugin-provided text elsewhere, such as RSS or metadata, if you want a fully branded site.

### Fix mobile TOC behavior

1. Keep TOC on the right if you want Quartz’s built-in mobile hiding behavior.
2. If TOC must move left, add `desktop-only`.
3. If the TOC still appears, override the relevant mobile sidebar selector in SCSS.

## 13. Safe defaults

If you want to customize Quartz without breaking it, use this order:

1. Edit plugin options.
2. Move plugin layout positions.
3. Use `display: desktop-only` or `mobile-only`.
4. Change theme tokens.
5. Use `quartz.ts`.
6. Use custom SCSS last.

That order minimizes surprises.

## 14. The short version

Quartz v5 is customizable because the site is assembled from declarative plugin entries and a small set of layout frames. For most customizations, edit `quartz.config.yaml`. Use theme settings for look-and-feel, plugin options for behavior, layout positions for placement, and `display` wrappers for responsive visibility. Use `quartz.ts` and SCSS only when YAML stops being enough.

For your specific mobile TOC case, the key point is that the default mobile hiding rule applies to the right sidebar. If you move the TOC left, it can still render on mobile unless you explicitly hide it.
