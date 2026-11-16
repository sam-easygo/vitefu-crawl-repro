# vitefu-crawl-repro

A small reproduction for https://github.com/bluwy/vitefu/pull/31.

`crawlFrameworkPkgs` reaches a package once for each chain of parents that leads to it, and it finds and reads the `package.json` again each time. With a few Svelte libraries that do not depend on each other, this is about 20 lookups. When Svelte libraries depend on each other in layers, the number of chains multiplies at each layer.

## What is here

A pnpm workspace with a Vite app that uses `@sveltejs/vite-plugin-svelte`, and 32 Svelte libraries in `packages/`.

- The libraries are in 8 layers of 4: `lib-0-0` to `lib-7-3`.
- The app depends on the 4 libraries in layer 0.
- Each library depends on the 4 libraries in the next layer.

So there are 4 chains to each library in layer 0, 16 to each in layer 1, and 65,536 to each in layer 7.

## Run it

```sh
pnpm install
time pnpm build
```

The build itself takes about 40 ms. The rest of the time is the crawl, before Vite starts the build.

To try the fix, remove the `#` from the two `patchedDependencies` lines in `pnpm-workspace.yaml`, then run the same two commands again. `patches/vitefu@1.1.3.patch` is the diff of the pull request.

## Results

`time pnpm build`, 5 runs each, Node 26 on a laptop.

| | Total time |
| --- | --- |
| `vitefu@1.1.3` | 4.2 to 6.1 s |
| With the fix | 0.70 to 0.77 s |

## The real project

The shape here is simpler than a real project, but the cause is the same. In the workspace where this was found, one crawl through `@sveltejs/vite-plugin-svelte` does this:

| Measurement | Value |
| --- | --- |
| Svelte packages the crawl detects | 111 |
| `package.json` files the crawl goes into | 127 |
| Times it goes into them | 19,743 |
| Deepest chain of Svelte packages | 13 |
| Packages with 2 or more Svelte parents | 43 |
| Packages with 5 or more Svelte parents | 18 |
| Most times one package is crawled | 2,369 |
| Dependency lookups | 131,261 for 848 different answers |
| Time | about 10 s on a laptop, about 34 s on a CI runner |
