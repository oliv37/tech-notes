# Tech Notes

Personal knowledge base of short, visual notes about technologies (frameworks, tools, languages). Goal: capture *why* a technology exists and *how* it works, not exhaustive documentation. See README.md for the reader-facing description.

## Structure

- One folder per technology, grouped under a category folder (`back-end`, `front-end`, `build-tools`, `databases`, `devops`, etc.), e.g. `build-tools/nx/`.
- Each technology folder contains a `README.md` (the note) and an `img/` folder for its images.
- Image files are named after the concept they illustrate, not the technology (the folder already gives that context): `use-case.jpeg`, `structure.jpeg`, `cache.jpeg`, not `nx_use_case.jpeg`.
- Naming convention uses hyphens for multi-word names (`front-end`, `build-tools`, `use-case.jpeg`).
- After adding a technology note, add a link to it in the root `README.md` under "Notes", grouped by category.

## Note template

Each technology `README.md` has two sections:

1. **The use case** — why the technology was created and when to reach for it. One image + a short paragraph.
2. **Key concepts** — the concepts worth knowing, broken into `###` subsections (e.g. Structure, Cache, Affected, Task pipeline for Nx; `satisfies`, Type narrowing, `as const` for TypeScript). Each subsection typically has its own image + a short paragraph, and can include a minimal usage example when relevant.

Keep paragraphs short (a few sentences) — this is a cheat sheet, not full documentation. Only add a subsection when there's a genuinely distinct concept to explain; don't force one for the sake of completeness (e.g. skip an image for a simple concept if it wouldn't add anything).

## Working conventions

- Write in English, without a first-person voice ("this repo helps..." not "I use this repo to...").
- Images are generated externally by the user with an AI image tool, then dropped into the `img/` folder — Claude does not generate them, but can write a descriptive prompt (matching the existing visual style: light blue gradient background, bold navy uppercase title, rounded panels, blue pill labels, arrows) for the user to run through their tool.
- Before writing explanatory text for an image, read it and check the text actually matches what's shown — call out any mismatch rather than describing what the text says the image should show.
- Verify technical claims against how the technology actually behaves in practice, not just what a diagram implies (e.g. a generated image can be wrong or misleading — check against real usage, like `ng test` not requiring a prior build).
