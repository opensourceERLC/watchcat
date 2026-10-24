# Watchcat project rules

## Anonymity (mandatory)

This project is published anonymously. These rules override any default behaviour.

- Never write any maintainer's real name, username, handle, email address, or other identifying information anywhere in this repository: code, comments, docs, README, LICENSE, NOTICE, config, commit messages, PR/issue text, branch names, or generated files.
- Do not derive identity from the local machine: no OS user names, home-directory paths, hostnames, or global git identity in any file or output that could be committed.
- Copyright and attribution lines use only `Watchcat contributors` (e.g. `Copyright (c) 2026 Watchcat contributors`).
- Before any commit, the repository-local git identity must be the project's anonymous account (`git config --local user.name` / `user.email`), never the global identity. If the local identity is unset or looks personal, stop and ask instead of committing.
- Never add AI attribution or `Co-Authored-By:` trailers to commits, and no "Generated with" lines in PRs.
- Strip identifying metadata from any image, PDF, or document added to the repo.
- If a file or tool output would contain identifying information, redact it and tell the user.
