# headless-wordpress-architect
AI skill for designing and reviewing scalable headless WordPress architectures, I built it from production experience operating high-traffic publishing platforms. Covers content delivery, webhooks, caching, SEO, Core Web Vitals, failure recovery, and frontend architecture across static, SSR, and multi-site systems.

The agent first classifies a request as **review**, **design**, or **implementation**, then applies only the guidance that matches measured scale. Architecture choices (REST vs GraphQL, static vs SSR, cache TTL, shared content service) are adaptable. Safety requirements (auth, secrets, unpublished content, idempotent webhooks, HTML sanitization) are not.

## Install

Clone or copy this repo into your agent skills directory.

**Cursor**

```bash
git clone https://github.com/Christonit/headless-wordpress-architect.git \
  ~/.cursor/skills/headless-wordpress-architect
```

**Claude Code**

```bash
git clone https://github.com/Christonit/headless-wordpress-architect.git \
  ~/.claude/skills/headless-wordpress-architect
```

## What it covers

Most headless WordPress advice assumes one right answer. In practice the right answer changes with how big the site is, how often editors publish, and what you are optimizing for: freshness, Core Web Vitals, infrastructure cost, or the editorial workflow. This skill makes the agent ask those questions first and then commit to an architecture, instead of defaulting to whatever pattern it saw most often in training data.


| If the site looks like this | What I reach for |
| --- | --- |
| Small, static, changes about once a day | Build the pages at build time. One CMS request per page is plenty, and WordPress never sees production traffic. |
| Medium, publishing regularly, builds still finish comfortably | REST at request time, with shared dictionaries for menus, taxonomies, authors, and SEO snapshots. Incremental builds are fine while they still hit the freshness target. |
| Large, high concurrency, or several sister sites | Stop rebuilding. A webhook normalizes the post, persists slim JSON, and the frontend serves it by slug. If more than one site repeats that work, move it into a shared content service. |

The goal is always the simplest setup that meets the current numbers, plus a clear signal for when to move to the next one, usually build time or publishing frequency crossing a budget you agreed on up front.

Also in the skill:

- Detail documents vs listing indexes vs shared dictionaries
- REST vs GraphQL with measurement, not dogma
- Webhook idempotency, ordering, and recovery
- Deferred third-party embeds (players, charts, voice)
- SEO snapshots so plugin updates do not break frontend metadata

## Files

| File | Role |
| --- | --- |
| [`SKILL.md`](./SKILL.md) | Agent contract, scale table, architecture guidance |
| [`reference.md`](./reference.md) | Evidence protocol, safety requirements, JSON/webhook contracts |

## Related writing

- [How to improve page load times and Web Core Vitals in Nuxt websites](https://dev.to/christonit/how-to-improve-page-load-times-and-web-core-vitals-in-nuxt-websites-j3l)
- [AWS, JavaScript, WordPress, and AI content automation](https://dev.to/christonit/aws-javascript-wordpress-fun-content-automation-strategies-using-artificial-intelligence-1gl1)

Treat those as historical case studies. The skill supersedes older claims (for example WordPress REST *can* query by slug).

## Author

[Christopher Santana](https://chsantana.com) — Full Stack Engineer. This skill is featured on the [portfolio](https://chsantana.com/projects).
```
