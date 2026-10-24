# Security policy

## Supported versions

Watchcat has not had a release yet. Until version 1.0, only the latest commit on `main` receives security fixes.

| Version | Supported |
|---|---|
| `main` (latest) | Yes |
| Anything older | No |

## Reporting a vulnerability

**Please do not report security vulnerabilities through public issues, discussions or pull requests.**

Report them privately through GitHub's private vulnerability reporting:

1. Go to the repository's **Security** tab.
2. Select **Report a vulnerability**.
3. Describe the issue, the affected component, steps to reproduce, and the potential impact.

Please include, where possible:

- the type of issue (for example authentication bypass, injection, secret exposure, sanction forgery);
- the affected file paths, commit or configuration;
- a minimal proof of concept;
- whether the issue could cause **wrongful automated sanctions**, leak **server keys or tokens**, or expose **player data**.

Do not include real players' personal data, real server keys or real tokens in your report. Redact them.

## What to expect

- We aim to acknowledge reports within **7 days**.
- We will keep you informed while we investigate and fix the issue.
- Once a fix is released, we will publish a security advisory. We are happy to credit you under the name or pseudonym of your choice, or not at all.

Please give us reasonable time to fix the issue before any public disclosure.

## Scope

In scope:

- code in this repository;
- default configuration and deployment files shipped here;
- anything that allows an attacker to trigger, forge or suppress sanctions, impersonate staff, or obtain credentials or player data from a Watchcat instance.

Out of scope:

- vulnerabilities in the **ER:LC API itself**: report these to Police Roleplay Community as described in the [ER:LC API acceptable use policy](https://apidocs.erlc.gg/policies/aup.md);
- vulnerabilities in Roblox, Discord or other third-party services;
- misconfiguration of a specific self-hosted instance by its operator.

## Leaked credentials

If you find an ER:LC server key, API key or Discord token committed to this repository or posted in an issue, report it privately using the process above. If it is your own key, **rotate it immediately**: regenerate the ER:LC server key in your private server settings and reset the Discord token in the Discord Developer Portal.
