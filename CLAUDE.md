# Tech Notes

Personal knowledge base of short notes about technologies (frameworks, tools, languages). Goal: capture _why_ a technology exists and _how_ it works, not exhaustive documentation. See README.md for the reader-facing description.

## Structure

- One folder per technology, grouped under a category folder (`back-end`, `front-end`, `build-tools`, `databases`, `devops`, `languages`, etc.), e.g. `build-tools/nx/`.
- Each technology folder contains a `README.md` and its notes.
- The `README.md` gives a brief description of the technology and lists links to its notes.
- Notes are individual markdown files, numbered in reading/learning order: `01-fundamentals.md`, `02-plugins.md`, etc. Each note groups a handful of related concepts — broad enough to avoid ending up with dozens of tiny files, focused enough to stay readable. Split off a new note when a topic clearly stops being related to the current one, not for every single concept.
- The filename doesn't repeat the technology name (the folder already gives that context, and it keeps filenames short), but the note's title (`#`) does, e.g. `01-fundamentals.md` titled `# Nx fundamentals` — the title is the only anchor left when the note is opened on its own, outside its folder.
- A note has no fixed format: use whatever mix of prose, code, images, or Q&A best explains its topic.
- Images referenced by a note live in `img/<note-number>/`, e.g. `img/01/`, named after the concept they illustrate, not the technology (the folder already gives that context): `use-case.jpeg`, `structure.jpeg`, not `nx-use-case.jpeg`.
- Naming convention uses hyphens for multi-word names (`front-end`, `build-tools`, `01-fundamentals.md`, `use-case.jpeg`).
- After adding a technology, add a link to its folder under its category heading in the root `README.md` (create the heading if the category is new). After adding a note, add a link to it in the technology's `README.md`.

## Working conventions

- Write in English, without a first-person voice ("this repo helps..." not "I use this repo to...").
- Keep notes short — this is a cheat sheet, not full documentation.
- Images are generated externally by the user with an AI image tool, then dropped into the relevant `img/<note-number>/` folder — Claude does not generate them, but can write a descriptive prompt (matching the existing visual style: light blue gradient background, bold navy uppercase title, rounded panels, blue pill labels, arrows) for the user to run through their tool.
- Before writing explanatory text for an image, read it and check the text actually matches what's shown — call out any mismatch rather than describing what the text says the image should show.
- Verify technical claims against how the technology actually behaves in practice, not just what a diagram implies (e.g. a generated image can be wrong or misleading — check against real usage, like `ng test` not requiring a prior build).
