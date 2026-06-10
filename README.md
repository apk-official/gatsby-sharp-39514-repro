# gatsby-sharp-39514-repro

Reproduction for [gatsbyjs/gatsby#39514](https://github.com/gatsbyjs/gatsby/issues/39514): sharp installation fails intermittently on Node 22 production builds.

## Root cause

Gatsby pins `sharp@^0.32.6` across four packages (gatsby-plugin-sharp, gatsby-transformer-sharp, gatsby-sharp, gatsby-plugin-manifest). sharp 0.32 downloads its libvips binary from GitHub in a postinstall script. When that download fails in CI, sharp falls back to compiling libvips from source, which fails on missing system headers (`glib-object.h: No such file or directory`). sharp 0.33+ removed the postinstall download entirely and ships prebuilt binaries as regular npm packages (`@img/*`), so upgrading sharp fixes this class of failure. This repo shows what breaks in Gatsby when you do that upgrade.

## Environment

- Node 22 (see `.nvmrc`)
- npm
- macOS arm64 (the engine error mentions `darwin-arm64`, on Linux CI it would be `linux-x64`)

## Repro steps

Each commit in this repo is one step. Build output for each step is saved in `logs/`.

| Step | Commit | What happens | Log |
|------|--------|--------------|-----|
| 1 | scaffold gatsby-starter-default | Fresh `gatsby new` on Node 22 | - |
| 2 | render images via StaticImage | Baseline: `gatsby build` passes on sharp 0.32.6 | `logs/01-build-sharp-0.32.6-passes.txt` |
| 3 | override sharp to 0.34.5 | `gatsby build` fails at "Validating Rendering Engines" with `Generated engines use disallowed import "@img/sharp-darwin-arm64/sharp.node"` | `logs/02-build-sharp-0.34.5-engine-validation-fails.txt` |
| 4 | remove DSG/SSR pages | `gatsby build` passes on sharp 0.34.5, all image processing works | `logs/03-build-sharp-0.34.5-no-engines-passes.txt` |

To run it yourself at any step:

```bash
nvm use
rm -rf node_modules package-lock.json
npm install
npx gatsby clean
npx gatsby build
```

## What this shows

1. Image processing itself works fine on sharp 0.34. Step 4 builds and generates all image variants correctly.
2. The breakage is in Gatsby's engine bundling. `packages/gatsby/src/schema/graphql-engine/bundle-webpack.ts` is built around old sharp's vendor layout and also calls `sharp.vendor.installed`, an API removed in sharp 0.33. The validation in `packages/gatsby/src/utils/validate-engines/child.ts` rejects the new `@img/*` binary imports.
3. Not visible in this repro but found by inspection: `sharp.simd()` was removed in sharp 0.33 and is still called in `packages/gatsby-sharp/src/index.ts` and `packages/gatsby-plugin-manifest/src/safe-sharp.js`. Both calls would crash on the new version.

Also worth noting: sharp 0.33+ requires Node ^18.17 || ^20.3 || >=21, slightly narrower than Gatsby's current `>=18.0.0` engines field.