---
name: dockerize-project
description: "Inspect a software repository and create a secure, cache-efficient Docker local development setup. Use when asked to containerize a project, add a development Dockerfile, or generate Docker build and run instructions."
triggers:
  - "Dockerize a project for local development"
  - "Generate a project-specific Dockerfile and .dockerignore"
  - "Inspect a repository and provide Docker run instructions"
tools:
  - read_file
  - bash
---

# Dockerize Project

Inspect the target repository and create a local-development container setup tailored to its actual runtime, dependencies, commands, configuration, and ports. Do not modify application source files. Do not create a generic or guessed configuration when repository evidence can determine the correct values.

## Workflow

### 1. Discover the project

Inspect the repository root and relevant subdirectories. Read only the files needed to establish the application and its runtime. Check, as applicable:

- Manifests and lockfiles: `package.json` and npm/yarn/pnpm lockfiles, `pyproject.toml`, `requirements*.txt`, `Pipfile`, `go.mod`, `Cargo.toml`, `pom.xml`, `build.gradle*`, `composer.json`, `Gemfile`, `*.csproj`, `global.json`, `Dockerfile`, and workspace/monorepo manifests.
- Runtime and tool-version declarations: `.nvmrc`, `.node-version`, `.python-version`, `rust-toolchain.toml`, `go.mod`, Java toolchains, SDK configuration, and CI setup.
- Existing developer commands and service configuration: `Makefile`, `Procfile`, package scripts, framework config, server entry points, and existing container files.
- Environment examples and source/config references for environment variables, listening sockets, external services, and persistent paths. Inspect `.env.example` or equivalent without revealing or copying real secret values from local `.env` files.
- Ignore rules and generated/dependency directories to identify build output, caches, local dependencies, and persistent application data.

For monorepos, identify the requested application and its working directory. Reuse an existing Docker setup if it already supports the requested local workflow; do not overwrite it without a concrete reason. When several runtimes coexist, follow the app's dependency graph and its declared development command rather than selecting a language based on file counts alone.

### 2. Establish the execution contract

Derive and record:

- Language, framework, package manager, runtime/SDK version, and install command from manifests and lockfiles.
- A development entry-point command from an explicit dev script, `Makefile`, `Procfile`, or framework convention. Prefer a watch/reload command for local development. Do not silently substitute a production command.
- Container listening ports and protocols from application code, framework configuration, and existing deployment/development settings. Distinguish the port bound inside the container from the host port. Ensure the application binds to `0.0.0.0` (or the runtime's equivalent) so host port publishing works; make only the smallest justified configuration adjustment, and never edit application source to do so.
- Required environment variables, defaults, and whether a value is secret. Report names and safe defaults only. A variable is required only when code/config or an explicit example indicates it is required; do not infer secrets or invent credential values.
- Required external services and persistent data paths. A service is a blocker only if the app cannot start without it and the repository provides no local default or existing service configuration.

Do not ask about details already established by repository evidence. If a necessary command, runtime version, required external service, or port cannot be resolved and choosing incorrectly would make the setup unusable, ask up to three short, targeted questions and wait before generating files. Otherwise choose the best-supported convention, disclose the assumption, and proceed.

### 3. Generate the local development setup

Create or update only the target application's `Dockerfile` and `.dockerignore`. Do not add Compose, production/staging targets, or change application source unless explicitly requested. Preserve unrelated user edits and existing project conventions.

For the `Dockerfile`:

- Choose an official runtime/development image compatible with the detected version and required native build tools. Prefer a maintained standard or slim variant when the application needs compilers, shell tools, or debugging utilities; use Alpine only when compatibility is established. Use a specific runtime version supported by the project, not `latest`.
- Set the application working directory and install dependencies before copying frequently changing source. Copy the relevant manifest and lockfile first, use the repository's package manager in its lockfile-respecting/frozen mode, then copy source. Include only files needed for a correct build; ensure every `COPY` source exists relative to the build context.
- Use a development/watch command established during discovery. Preserve interactive output and signal handling where practical; do not daemonize the app inside the container.
- Declare `EXPOSE` for each detected container listening port. This documents ports but does not publish them.
- Run as an unprivileged user when the selected image supports it. Ensure application directories are writable by that user. Account for bind-mounted source ownership without weakening permissions or embedding host-specific UID assumptions that break Windows/macOS use.
- Do not copy secret files into the image, pass secrets as build arguments, or bake environment-specific values into layers. Do not introduce multi-stage production optimization into this local-only setup.

For `.dockerignore`:

- Exclude VCS data, local dependencies, virtual environments, caches, build/test output, logs, editor/OS metadata, and generated artifacts that are not needed to build or run the app.
- Exclude `.env` and secret/key/credential files. Keep a safe example file such as `.env.example` available to inspection and, only if needed, to the build context.
- Do not ignore manifests, lockfiles, source, or configuration required by Dockerfile `COPY` commands. Check repository-specific build inputs before adding broad patterns.

Use the repository's existing editor file tools to write the files. If a requested path already contains user changes, read and preserve them; do not replace unrelated content.

### 4. Verify the setup

After writing the files, perform the narrowest available checks:

1. Confirm every Dockerfile `COPY`/`ADD` input exists within the build context and required manifests are not ignored.
2. Confirm every detected container port has a matching `EXPOSE` and a host-to-container `-p` mapping in the run command. Do not map unrelated ports.
3. Confirm `.dockerignore` excludes `.env` and other discovered secrets, local dependencies, VCS data, and relevant build artifacts while retaining required inputs.
4. If Docker is installed and the project has all required local inputs, run `docker build` from the correct context. Report build failures accurately; do not claim a build was verified if Docker is unavailable or a required external service blocks it.
5. Check that the run command uses the same image tag, work directory, ports, environment file, and bind mounts as the generated files. Use `--env-file` only for a developer-provided local file and never create or print live credentials.

Do not install Docker or alter machine-wide configuration as part of this skill.

## Response Format

Start with a concise inspection summary table containing:

| Item | Finding |
|---|---|
| Detected language/framework | ... |
| Runtime version | ... |
| Entry-point command | ... |
| Exposed ports | ... |
| Required environment variables | ... |

List any unresolved, blocking questions before creating files. If there are none, state assumptions briefly, then summarize the generated `Dockerfile` and `.dockerignore` and provide concrete commands for the inspected repository:

- `docker build` with the correct build context and image tag.
- `docker run` with `--rm -it`, matching `-p` mappings, a source bind mount for iterative development, and the required working directory/command where needed.
- Include `--env-file` only when the project requires runtime variables and an example or documented file identifies their names. Tell the developer to supply their own local values; never output secret values.

Give syntax appropriate to the user's shell. For Windows PowerShell, use its current-directory path form for volume mounts; for POSIX shells, use `$PWD`. State that the guide targets local development, list verification actually completed, and explain any remaining assumptions or checks that could not run.