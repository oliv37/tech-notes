# Tech Notes

Short technical notes covering the essentials for a Spring Boot / Angular back-end developer: Java, Spring Boot, Hibernate, Angular, TypeScript, RxJS, HTTP, security, Kafka, architecture. Goal: a quick refresher, e.g. to answer interview questions, not in-depth documentation.

## Structure

- One file per technology, named in uppercase, in a folder per group: `notes/back-end/<family>/` (`java`, including the servlet API, `spring`, `jdbc`, `hibernate`, `test`, `kafka`, `elasticsearch`), `notes/front-end/<family>/` (`angular`, `typescript`, `rxjs`), `notes/cross-cutting/`. E.g. `notes/back-end/java/JAVA.md`.
- After adding a file, link it in `README.md` under its group and family. Links between notes are relative (`../spring/SPRING.md`).
- Each file is a list of `##` sections, one per topic or question (`## map vs flatMap`, `## Hibernate N+1 problem`). No `#` title.
- Sections go from the simplest topic to the hardest; forward references use "see below".
- `_old/` holds previous notes kept for reference; don't edit them.

## Writing a section

- Short: the essentials needed to explain the topic in an interview, not every detail. A small detail gets a few lines or bullets.
- Start each section with a `> **In short**: ...` blockquote: the 2-3 sentence answer to give in an interview. The details follow below it.
- Then what it is / the key difference, then a minimal code example when it helps, then pitfalls or when to use it.
- Prefer concrete examples (`Author` / `Book`, `User`) and comparison tables for "X vs Y" topics.
- Verify technical claims against how the technology actually behaves, including the version a feature appeared in and whether it's final or still in preview.

## Working conventions

- Write in English, without a first-person voice.
- Commits follow Conventional Commits, always with type `docs`, scoped to the technology when the change is specific to one, e.g. `docs(java): ...` — no scope for repo-wide changes (README, CLAUDE.md).
