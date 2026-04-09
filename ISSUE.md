## `<style>` tags in template silently prevent import organization

### Description

Import organization silently fails (returns the file unchanged) whenever the `.astro` file contains a `<style>` tag in its template section. No error is reported -- the file is simply returned unmodified.

This affects any `.astro` component that uses `<style>`, `<style lang="scss">`, `<style is:global>`, etc., which in practice is the majority of Astro components.

### Root cause

The plugin's `organizeImports` path (the `Ypt` function in the bundled source) extracts the frontmatter and wraps the remaining template in a synthetic TSX function:

```ts
// Simplified from the bundled source
const combined = frontmatter
  + `const __ASTRO_BOUNDARY__=0\nfunction __(){return(<>\n`
  + template
  + `</>)}`;
```

This combined string is then passed to TypeScript's language-service `organizeImports` API. A `<style>` tag containing CSS is not valid TSX, so TypeScript fails to parse the file and `organizeImports` returns it unchanged. The outer `try/catch` in `JR` doesn't fire because TS doesn't throw -- it simply returns no edits for an unparseable file.

### Reproduction

**Two files -- identical except one has a `<style>` tag:**

`src/no-style.astro` (imports are sorted correctly):
```astro
---
import Beta from "./components/Beta.astro";
import Alpha from "./components/Alpha.astro";
---
<Alpha />
<Beta />
```

`src/with-style.astro` (imports are NOT sorted):
```astro
---
import Beta from "./components/Beta.astro";
import Alpha from "./components/Alpha.astro";
---
<Alpha />
<Beta />
<style>
  div { color: red; }
</style>
```

`prettier.config.cjs`:
```js
/** @type {import("prettier").Config} */
const config = {
  plugins: [
    "prettier-plugin-astro",
    "prettier-plugin-astro-organize-imports",
  ],
};
module.exports = config;
```

**Run:**
```sh
npx prettier -w src/no-style.astro   # Alpha sorted before Beta
npx prettier -w src/with-style.astro  # Beta stays before Alpha (unchanged)
```

A full reproduction project is available at: https://stackblitz.com/github/adamchal/astro-organize-imports-style-bug

### Expected behavior

Imports should be organized regardless of whether a `<style>` tag is present in the template.

### Suggested fix

Strip `<style>` (and possibly `<script>`) blocks from the template portion before constructing the synthetic TSX string that gets passed to TypeScript. The `Rge` function already uses `@astrojs/compiler/sync` to parse the AST and locate `<script>` elements -- the same approach could be used to remove `<style>` elements before the content reaches `Ypt`.

### Environment

- `prettier-plugin-astro-organize-imports`: 0.4.12
- `prettier-plugin-astro`: 0.14.1
- `prettier`: 3.8.1
- `astro`: 5.x
- Node: 22.x
