---
title: Contribution
nav_order: 10
---

# Contribution

Contributions to MOSAIC are welcome. This page explains how to report issues, propose changes, and submit pull requests across the MOSAIC repositories.

---

## Repositories

| Repository | Language | What it contains |
|:---|:---|:---|
| [mosaic-core](https://github.com/ACSL-MOSAIC/mosaic-core) | C++17 | Robot-side WebRTC library |
| [MOSAIC-Server](https://github.com/ACSL-MOSAIC/MOSAIC-Server) | Java / TypeScript | Backend + frontend dashboard |
| [mosaic-ros2](https://github.com/ACSL-MOSAIC/mosaic-ros2) | C++ / Python | ROS2 connector packages |
| [mosaic-docs](https://github.com/ACSL-MOSAIC/mosaic-docs) | Markdown | This documentation site |

---

## Reporting Issues

Before opening an issue, search the existing issues on the relevant repository to avoid duplicates.

When filing a bug report, include:

- A clear description of the problem and the expected behavior
- Steps to reproduce the issue
- Your environment (OS, ROS2 distribution, compiler version, etc.)
- Relevant log output or error messages

For feature requests, describe the use case and why the existing API does not cover it.

---

## Submitting a Pull Request

1. **Fork** the repository and create a branch from `main`:

   ```bash
   git checkout -b feat/your-feature-name
   ```

2. **Make your changes.** Keep commits focused — one logical change per commit.

3. **Test your changes** before opening a PR:
   - mosaic-core: build with `-DBUILD_TESTS=ON` and run the test suite
   - MOSAIC-Server backend: run `./gradlew test`
   - MOSAIC-Server frontend: run `npm run lint && npm test`

4. **Open a Pull Request** against the `main` branch. Fill in the PR template with:
   - What problem this solves or what feature it adds
   - How it was tested
   - Any breaking changes

---

## Coding Style

### mosaic-core (C++)

- Follow the existing code style (Google C++ Style as a baseline).
- Use `snake_case` for variables and functions, `PascalCase` for classes.
- Keep public headers clean — implementation details go in `.cpp` or via PIMPL.
- Add unit tests under `test/` for new functionality.

### MOSAIC-Server backend (Java)

- Follow standard Spring Boot conventions.
- Reactive types (`Mono`, `Flux`) throughout — avoid blocking calls.
- New database schema changes go in a new Flyway migration file (`V{N}__description.sql`).

### MOSAIC-Server frontend (TypeScript / React)

- Run `npm run lint` and fix all warnings before committing.
- Follow the existing hook and context patterns when adding new store or channel types.
- Components go under `src/components/`, hooks under `src/hooks/`.

### mosaic-ros2 (C++ / Python)

- Follow ROS2 coding conventions and `ament_cmake` package structure.
- Package names use lowercase with hyphens (`mosaic-ros2-sensor`).

---

## Commit Messages

Use the conventional commit format:

```
<type>: <short summary>

<optional body>
```

Common types: `feat`, `fix`, `docs`, `refactor`, `test`, `chore`

Examples:

```
feat: add DataChannelByteReceivable base class
fix: stop capture thread before joining on disconnect
docs: add Config on Robot page
```

---

## Documentation

Documentation lives in the [mosaic-docs](https://github.com/ACSL-MOSAIC/mosaic-docs) repository and is built with Jekyll + Just the Docs.

To preview locally:

```bash
bundle install
bundle exec jekyll serve
```

The site is served at `http://localhost:4000`.

When adding a new page, set the `nav_order` and `parent` front matter fields to place it correctly in the sidebar.