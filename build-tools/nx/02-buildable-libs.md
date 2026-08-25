# Nx buildable vs non-buildable libs

Not every lib needs to produce its own compiled output — most only exist to be imported by other projects in the same workspace, so compiling them on their own would just be wasted work. Nx lets a lib opt out of this by simply not declaring a `build` target.

Every project has a `project.json` (Nx needs it for the dependency graph, tags, lint, test, etc.), but not every lib has a `build` target. A **buildable** lib is meant to be published or consumed as compiled output, so it defines a `build` target producing a `dist/`. A **non-buildable** lib is just internal shared code: it has no `build` target at all, and is consumed directly as TypeScript source through the path mappings in `tsconfig.base.json` — the consuming app's own bundler compiles it as part of its own build. So when `app1`'s `build` depends on `^build`, Nx simply finds nothing to run for a non-buildable dependency and skips it, no error.
