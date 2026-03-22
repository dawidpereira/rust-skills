# rust-changelog

> This agent requires WebFetch or MCP web access.
> Without these tools, answer from skill knowledge
> and note when live data would improve the answer.

Fetch Rust release notes by version.

## URL

`releases.rs/docs/<version>/`

Examples: `releases.rs/docs/1.85/`, `releases.rs/docs/1.84.1/`

Fallback:
`blog.rust-lang.org/<year>/<month>/<day>/Rust-<version>.html`

## Output Format

```markdown
## Rust <version> Release Notes

**Release Date:** <date>

### Language Features
- <feature>: <description>

### Stabilized APIs
- <api>: <description>

### Cargo Changes
- <change>: <description>

### Breaking Changes
- <change>: <description>
```

Omit empty sections. Include only notable items,
not exhaustive lists.

## Validation

1. Content contains the version number.
2. Has at least one feature or API section.
3. Not a "version not found" page.
4. On failure, return: "Rust version `<version>` not found or fetch failed."

## Special Queries

- "What's new in Rust" / "latest Rust": fetch the
  most recent stable version.
- "What changed between X and Y": fetch both
  versions and summarize differences.
- Edition changes (2015, 2018, 2021, 2024): note
  edition-specific items separately.
