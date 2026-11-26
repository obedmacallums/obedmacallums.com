---
author: Obed Macallums
pubDatetime: 2026-11-26T21:07:43Z
title: "Agent Skills Meet MCP: Exposing Your Python Tools as Reusable Skills"
slug: agent-skills-mcp-python
featured: false
draft: false
tags:
  - mcp
  - agents
  - agent-skills
  - python
  - llm
  - fastapi
description: Learn how to package procedures as Agent Skills (SKILL.md plus supporting files) and serve them over MCP from Python next to the tools they describe. Covers the SKILL.md format, the skill:// resource convention from the MCP Skills extension (SEP-2640), manifests with digests, and how skills and MCP tools complement each other in an agent workflow.
---

An MCP server can give an agent **tools**: functions it can call, like "classify this text" or "query this database". A tool doesn't tell the agent _when_ to call it, in what order, or what to do with the result. That knowledge usually lives in a long system prompt or in someone's head. **Agent Skills** fill that gap. A skill is a folder of instructions, reference documents and scripts that the agent loads only when it needs them. Now that the MCP Skills extension is final, a server can ship its skills next to its tools. In this article we build a small Python MCP server that does both: it exposes a sentiment-classification tool backed by a FastAPI service, and it serves a `feedback-triage` skill that teaches the agent how to use that tool properly.

## Table of Contents

## Tools vs. Skills: Two Halves of an Agent

The two concepts are easy to mix up, so here is the distinction we'll use throughout the post:

| Aspect       | MCP tool                               | Agent Skill                                              |
| ------------ | -------------------------------------- | -------------------------------------------------------- |
| What it is   | A callable function with a JSON schema | A directory with a `SKILL.md` and optional files         |
| Answers      | "What can I do?"                       | "How should I do this task, step by step?"               |
| Executed by  | The server                             | The model reads it and follows it (scripts may run)      |
| Context cost | Schema always loaded                   | Name + description only, until the skill is activated    |
| Changes when | The API changes                        | The procedure, policy or domain knowledge changes        |
| Example      | `classify_sentiment(text)`             | "Triage customer feedback: classify, group, escalate..." |

A useful rule of thumb: **put anything deterministic in a tool and anything procedural in a skill**. Computing a sentiment score, querying a database or calling an API belong in tools. "Always classify every item before summarizing", "escalate anything negative that mentions billing" or "format the report like this template" belong in a skill.

Until recently the only standard place to put that procedural text was the MCP server's `instructions` field. It's a single string with a practical size limit, and the agent gets all of it at once. Skills give that text structure and let the agent load it in stages.

## The Agent Skills Format

Agent Skills is an open specification (originally from Anthropic, now maintained under the Agentic AI Foundation alongside MCP). A skill is a directory with at least one file:

```text
feedback-triage/
├── SKILL.md          # Required: metadata + instructions
├── references/       # Optional: documentation loaded on demand
│   └── RUBRIC.md
└── scripts/          # Optional: executable helpers
    └── summarize.py
```

`SKILL.md` starts with YAML frontmatter and is followed by Markdown instructions. Only two fields are required:

- `name`: 1–64 characters, lowercase letters, digits and hyphens only, no leading, trailing or consecutive hyphens, and it **must match the directory name**.
- `description`: 1–1024 characters, saying both what the skill does and when to use it.

Optional fields include `license`, `compatibility`, a string-to-string `metadata` map and the experimental `allowed-tools`.

The key idea is **progressive disclosure**. An agent loads skills in three stages:

1. **Metadata** (about 100 tokens): only `name` and `description` are loaded at startup, for every skill.
2. **Instructions**: the full `SKILL.md` body is loaded when the skill is activated. The spec recommends keeping it under 500 lines.
3. **Resources**: files in `references/`, `scripts/` or `assets/` are read only when the instructions call for them.

So you can offer dozens of skills to an agent and pay roughly one sentence of context for each, until one of them is actually needed.

## How Skills Travel over MCP

The MCP Skills extension (`io.modelcontextprotocol/skills`, specified in SEP-2640, now **Final**) doesn't invent a new primitive. It maps skills onto the existing **Resources** primitive:

- Every file in a skill directory is an MCP resource, conventionally under the `skill://` scheme: `skill://<skill-path>/<file-path>`.
- The skill's entry point is always `skill://<skill-path>/SKILL.md`, and the last segment of `<skill-path>` must equal the skill's `name`. A prefix is allowed for organization, e.g. `skill://acme/support/feedback-triage/SKILL.md`.
- The `SKILL.md` resource should use `mimeType: text/markdown`, with the resource `name` and `description` copied from the frontmatter.
- Relative references inside a skill (`references/RUBRIC.md`) resolve against the skill root, exactly as on a filesystem.

On top of that, a server that **declares** the extension must implement two methods:

- `skills/list`: enumerates skills, each with its verbatim `frontmatter` and a complete `resources` manifest of `{uri, digest, size}` entries (SHA-256 digests).
- `skills/get`: returns the same entry for a single skill URI, including skills that aren't listed.

There's also an optional `resources/directory/read` method, gated behind a `directoryRead` capability flag. The declaration looks like this:

```json
{
  "capabilities": {
    "resources": {},
    "extensions": {
      "io.modelcontextprotocol/skills": {
        "directoryRead": false
      }
    }
  }
}
```

Two details from the spec matter for server authors:

1. **The baseline is plain `resources/read`.** Hosts must be able to load a skill given only its URI, for example a URI the server mentions in its `instructions`. A server that just serves `skill://` resources is already useful to any host that can read resources.
2. **Reading is not activating.** A `resources/read` of a `SKILL.md` only returns bytes. The host decides when a skill becomes active. It verifies the digests and the frontmatter and asks the user for approval where needed. Skill content is treated as untrusted input.

In the rest of the post we'll build that baseline with the stable Python SDK, then compute the exact manifest that `skills/list` returns, so moving to the full extension is a small step.

## Project Setup

We'll reuse the sentiment API from [Serving ML Models in Production with FastAPI + ONNX](/posts/serving-ml-models-fastapi-onnx/). Any HTTP service works, but a real model makes the example concrete. Start it on port 8001 so it doesn't collide with the MCP server:

```bash
uvicorn main:app --host 127.0.0.1 --port 8001
```

The MCP project is a separate folder:

```text
feedback-mcp/
├── server.py
├── skills_loader.py
└── skills/
    └── feedback-triage/
        ├── SKILL.md
        ├── references/
        │   └── RUBRIC.md
        └── scripts/
            └── summarize.py
```

Install the official MCP Python SDK (v2), `httpx` for calling the API, and PyYAML for parsing frontmatter:

```bash
pip install "mcp>=2.3" httpx pyyaml
```

## Step 1: The Tool

The tool is deliberately thin. It forwards text to the FastAPI service and returns the result:

```python
# server.py (part 1)
import os

import httpx
from mcp.server import MCPServer

SENTIMENT_API = os.environ.get("SENTIMENT_API", "http://127.0.0.1:8001/predict")

mcp = MCPServer(
    "feedback",
    instructions=(
        "Tools for analyzing customer feedback. Before triaging a batch of "
        "feedback, read the skill at skill://feedback-triage/SKILL.md."
    ),
)


@mcp.tool()
async def classify_sentiment(text: str) -> dict:
    """Classify the sentiment of one piece of text.

    Returns a dict with 'label' (POSITIVE or NEGATIVE) and 'score' (0-1).
    """
    async with httpx.AsyncClient(timeout=10.0) as client:
        response = await client.post(SENTIMENT_API, json={"text": text})
        response.raise_for_status()
        return response.json()
```

The `instructions` string points to the skill URI. This is the "pointer from server instructions" discovery path described in the spec. It works with any host that can read resources, even one that doesn't implement `skills/list` yet. Also note that the tool's docstring says nothing about _when_ to use the tool or what to do with low-confidence scores. That belongs in the skill.

## Step 2: Writing the Skill

`SKILL.md` contains the procedure. Its frontmatter is what every agent sees up front, so the description has to say clearly when the skill applies:

```markdown
---
name: feedback-triage
description: Triage a batch of customer feedback into positive, negative and needs-review groups, and escalate urgent negative items. Use when the user shares reviews, support tickets, survey answers or app store comments and asks for a summary or prioritization.
license: MIT
metadata:
  version: "1.0"
---

# Feedback triage

Follow these steps in order.

1. Call the `classify_sentiment` tool once per feedback item. Never guess
   the sentiment yourself.
2. Group results with the thresholds in [the rubric](references/RUBRIC.md):
   - `score >= 0.85`: trust the label.
   - `score < 0.85`: put the item in **needs review**, whatever the label.
3. Read the escalation rules in the rubric and flag every negative item
   that matches one of them as **urgent**.
4. If there are more than 20 items, save the results as JSON lines
   (`{"text", "label", "score"}`) and run `scripts/summarize.py` on that
   file to get the counts, instead of counting by hand.
5. Reply with:
   - a one-line overall summary with counts per group;
   - the urgent items first, each quoted verbatim;
   - at most three themes you noticed in the negative items.

Do not paraphrase urgent items: quote them exactly.
```

The rubric is a separate file. The agent reads it only in step 2, and the business rules can change without touching the main instructions:

```markdown
# Triage rubric

## Confidence thresholds

| Score      | Treatment              |
| ---------- | ---------------------- |
| >= 0.85    | Accept the model label |
| below 0.85 | Needs human review     |

## Escalation rules

A NEGATIVE item is **urgent** if it mentions any of:

- billing, charges, refunds or double payments;
- data loss, deleted work or security concerns;
- legal action or regulatory complaints.
```

The helper script is plain, dependency-free Python, so it runs anywhere:

```python
# skills/feedback-triage/scripts/summarize.py
"""Summarize classified feedback stored as JSON lines.

Usage: python summarize.py results.jsonl
"""
import json
import sys
from collections import Counter

THRESHOLD = 0.85


def main(path: str) -> None:
    groups: Counter[str] = Counter()
    with open(path, encoding="utf-8") as fh:
        for line in fh:
            if not line.strip():
                continue
            item = json.loads(line)
            if item["score"] < THRESHOLD:
                groups["needs_review"] += 1
            else:
                groups[item["label"].lower()] += 1
    print(json.dumps(dict(groups), indent=2))


if __name__ == "__main__":
    if len(sys.argv) != 2:
        sys.exit("usage: python summarize.py results.jsonl")
    main(sys.argv[1])
```

Whether a host will actually _execute_ a script served over MCP is up to the host. The spec requires explicit per-skill user approval for host-side code execution. That's why step 4 is a convenience for large batches and not the core of the procedure.

## Step 3: Loading and Validating Skills

Next we need a loader. It reads a skill directory, checks the Agent Skills naming rules and the extension's size limits, and lists the files to serve. Keeping it in its own module means we can use it from the server and from CI:

```python
# skills_loader.py
import hashlib
import json
import re
import sys
from dataclasses import dataclass
from pathlib import Path

import yaml

NAME_RE = re.compile(r"^[a-z0-9]+(-[a-z0-9]+)*$")
MAX_FILES = 512                  # per-skill limit from the Skills extension
MAX_TOTAL_BYTES = 16 * 1024 * 1024  # 16 MiB per skill

MIME_TYPES = {
    ".md": "text/markdown",
    ".py": "text/x-python",
    ".json": "application/json",
    ".txt": "text/plain",
}


@dataclass
class SkillFile:
    uri: str
    path: Path
    relative: str
    mime_type: str


@dataclass
class Skill:
    skill_path: str
    frontmatter: dict
    files: list[SkillFile]

    @property
    def uri(self) -> str:
        return f"skill://{self.skill_path}/SKILL.md"


def parse_frontmatter(skill_md: Path) -> dict:
    text = skill_md.read_text(encoding="utf-8")
    if not text.startswith("---\n"):
        raise ValueError(f"{skill_md} must start with YAML frontmatter")
    _, raw, _ = text.split("---\n", 2)
    return yaml.safe_load(raw)


def load_skill(skill_dir: Path, prefix: str = "") -> Skill:
    meta = parse_frontmatter(skill_dir / "SKILL.md")
    name, description = meta.get("name", ""), meta.get("description", "")

    if not (1 <= len(name) <= 64 and NAME_RE.match(name)):
        raise ValueError(f"invalid skill name: {name!r}")
    if name != skill_dir.name:
        raise ValueError(f"name {name!r} must match directory {skill_dir.name!r}")
    if not (1 <= len(description) <= 1024):
        raise ValueError(f"{name}: description must be 1-1024 characters")

    skill_path = f"{prefix}/{name}" if prefix else name
    files = []
    for path in sorted(skill_dir.rglob("*")):
        if not path.is_file() or "__pycache__" in path.parts:
            continue
        relative = path.relative_to(skill_dir).as_posix()
        files.append(
            SkillFile(
                uri=f"skill://{skill_path}/{relative}",
                path=path,
                relative=relative,
                mime_type=MIME_TYPES.get(path.suffix, "text/plain"),
            )
        )

    total = sum(f.path.stat().st_size for f in files)
    if len(files) > MAX_FILES or total > MAX_TOTAL_BYTES:
        raise ValueError(f"{name}: exceeds 512 files or 16 MiB")

    return Skill(skill_path=skill_path, frontmatter=meta, files=files)


def skill_entry(skill: Skill) -> dict:
    """Build the entry that skills/list and skills/get return for this skill."""
    resources = []
    for f in skill.files:
        data = f.path.read_bytes()
        resources.append(
            {
                "uri": f.uri,
                "digest": "sha256:" + hashlib.sha256(data).hexdigest(),
                "size": len(data),
            }
        )
    return {"uri": skill.uri, "frontmatter": skill.frontmatter, "resources": resources}


if __name__ == "__main__":
    root = Path(sys.argv[1] if len(sys.argv) > 1 else "skills")
    entries = [skill_entry(load_skill(d)) for d in sorted(root.iterdir()) if d.is_dir()]
    print(json.dumps({"skills": entries}, indent=2, default=str))
```

`skill_entry()` builds exactly the shape the spec defines for `skills/list`: the `SKILL.md` URI, the frontmatter copied verbatim, and a **complete** manifest in which every file appears once with a `sha256:` digest over its raw bytes and its byte size. The digests are what allow a host to cache files and to tie a user's approval to a specific version of the skill. If any file changes, the manifest changes and the approval has to be given again.

Run it as a validation step locally or in CI (see [CI/CD with GitHub Actions](/posts/github-actions-cicd/) for wiring a step like this into a pipeline):

```bash
python skills_loader.py skills
```

The output looks like this (digests abbreviated; sizes depend on your files):

```json
{
  "skills": [
    {
      "uri": "skill://feedback-triage/SKILL.md",
      "frontmatter": {
        "name": "feedback-triage",
        "description": "Triage a batch of customer feedback into ...",
        "license": "MIT",
        "metadata": { "version": "1.0" }
      },
      "resources": [
        {
          "uri": "skill://feedback-triage/SKILL.md",
          "digest": "sha256:...",
          "size": 1184
        },
        {
          "uri": "skill://feedback-triage/references/RUBRIC.md",
          "digest": "sha256:...",
          "size": 412
        },
        {
          "uri": "skill://feedback-triage/scripts/summarize.py",
          "digest": "sha256:...",
          "size": 803
        }
      ]
    }
  ]
}
```

A malformed skill (wrong name, missing description, oversized folder) now fails the build instead of confusing an agent in production.

## Step 4: Serving the Skill as MCP Resources

With the loader in place, registering every file as a resource takes a few lines. We use the SDK's `@mcp.resource` decorator with an explicit `name`, `description` and `mime_type`, and we register one static resource per file:

```python
# server.py (part 2)
from pathlib import Path

from skills_loader import Skill, SkillFile, load_skill

SKILLS_DIR = Path(__file__).parent / "skills"


def register_file(skill: Skill, file: SkillFile) -> None:
    is_entry = file.relative == "SKILL.md"

    @mcp.resource(
        file.uri,
        name=skill.frontmatter["name"] if is_entry else file.relative,
        description=skill.frontmatter["description"] if is_entry else None,
        mime_type=file.mime_type,
    )
    def read_file() -> str:
        return file.path.read_text(encoding="utf-8")


def register_skills(root: Path) -> list[Skill]:
    skills = [load_skill(d) for d in sorted(root.iterdir()) if d.is_dir()]
    for skill in skills:
        for file in skill.files:
            register_file(skill, file)
    return skills


register_skills(SKILLS_DIR)

if __name__ == "__main__":
    mcp.run(transport="streamable-http", host="127.0.0.1", port=8000)
```

A few design choices are worth explaining:

- **Static resources, not a URI template.** A template such as `skill://{name}/{path}` would need to match paths with multiple segments, and SDK v2 is stricter about template matching and path safety. Registering each file explicitly also means the server can only serve files that are in the manifest, which is the same rule hosts enforce on their side.
- **The file is read on every request.** If you edit the rubric, the next `resources/read` returns the new content. In production, bake the skills into the container image (see [Understanding Docker Basics](/posts/docker-basics/)) so the served bytes match a released version.
- **Metadata comes from the frontmatter.** The extension says the `SKILL.md` resource's `name` and `description` should come from the frontmatter. That's why the loader parses it once and reuses it.

Start the server:

```bash
python server.py
```

## Step 5: Trying It with a Client

The SDK's `Client` can connect to the Streamable HTTP endpoint. The script below does by hand what an agent host does: it discovers the skill, reads `SKILL.md`, follows its relative reference to the rubric, and calls the tool:

```python
# try_client.py
import asyncio

from mcp import Client

FEEDBACK = [
    "Love the new dashboard, it's so much faster.",
    "I was charged twice this month and nobody answers my emails.",
    "It's fine I guess.",
]


async def main() -> None:
    async with Client("http://127.0.0.1:8000/mcp") as client:
        listing = await client.list_resources()
        for resource in listing.resources:
            print(resource.uri, "-", resource.name)

        skill = await client.read_resource("skill://feedback-triage/SKILL.md")
        print(skill.contents[0].text[:300], "...")

        # Relative references resolve against the skill root
        rubric = await client.read_resource(
            "skill://feedback-triage/references/RUBRIC.md"
        )
        print(rubric.contents[0].text)

        for text in FEEDBACK:
            result = await client.call_tool("classify_sentiment", {"text": text})
            print(result.structured_content, "<-", text)


asyncio.run(main())
```

```text
skill://feedback-triage/SKILL.md - feedback-triage
skill://feedback-triage/references/RUBRIC.md - references/RUBRIC.md
skill://feedback-triage/scripts/summarize.py - scripts/summarize.py
---
name: feedback-triage
...
```

In a real agent the model drives these calls. At startup it sees the server instructions and the skill's name and description. When the user pastes a batch of reviews, the description matches and the host loads `SKILL.md`. The model then calls `classify_sentiment` once per item, reads `RUBRIC.md` when it reaches step 2, and returns a report in the format the skill asks for. The tool is never misused, and the escalation policy is applied the same way every time, because it's written down in one versioned place.

## Moving to the Full Skills Extension

What we built is the baseline the spec requires every host to support: skills that are readable by URI and announced through `instructions`. A server that fully adopts the extension adds three things:

1. It declares `io.modelcontextprotocol/skills` in its capabilities, next to `resources`.
2. It answers `skills/list` with entries like the ones `skill_entry()` builds, plus the list-caching fields `ttlMs` and `cacheScope` in protocol version `2026-07-28` and later.
3. It answers `skills/get` for any skill URI it serves, and returns `-32602` (Invalid params) for unknown ones.

Every official SDK needed a reference implementation before the SEP could be accepted. The Python implementation was merged into the SDK's main branch in October 2026, after the 2.3.0 release, and it plugs into the `extensions=` parameter of `MCPServer`. Its API may still change before it ships in a stable release, so check the SDK's release notes before you depend on it. The skill folder and the manifest logic above stay the same either way. The SDK helper takes over the declaration and the two methods.

On the host side, Microsoft Agent Framework for Python, Claude Code and other clients already load filesystem skills in the same three stages. Support for MCP-served skills is listed in the MCP client support matrix.

## Design Guidelines

A few lessons that apply whatever SDK version you use:

- **Write the description for routing.** The agent decides whether to load a skill from its name and description alone. Name the task, the inputs and the trigger phrases ("reviews, support tickets, survey answers").
- **Keep `SKILL.md` short and push details down.** Procedures go in `SKILL.md`; tables, policies and long examples go in `references/`. The agent pays for a reference file only when it reads it.
- **Make tools boring and skills opinionated.** `classify_sentiment` knows nothing about thresholds or escalation, so another skill (say, "weekly NPS report") can reuse it with different rules.
- **Version skills with the server.** Shipping both from the same repository solves the "fragmented distribution" problem that motivated the extension: whoever installs the server gets the matching instructions.
- **Assume skills are untrusted on the host side.** As a server author, don't rely on `allowed-tools` or scripts running automatically. Hosts must ask the user before granting those, and some hosts won't load skills whose resources are `"dynamic"` (no digests). Static files with stable digests are the most portable choice.

## Conclusion

MCP tools and Agent Skills solve two different problems. Tools give an agent capabilities. Skills give it the procedure for using them, loaded in stages so the context window stays small. With the MCP Skills extension, both can come from the same server, versioned together. In this post we wrote a `feedback-triage` skill with a rubric and a helper script, validated it against the Agent Skills naming rules and the extension's limits, built the exact `skills/list` manifest with SHA-256 digests, and served every file as a `skill://` resource next to a FastAPI-backed tool using the stable Python SDK.

From here, natural next steps are to switch to the SDK's built-in skills extension once it reaches a stable release, to add more skills that reuse the same tools, and to test your skills by running an agent against a fixed set of inputs, as you would with any other part of an API.

## References

- [Agent Skills Specification](https://agentskills.io/specification)
- [SEP-2640: Skills Extension](https://modelcontextprotocol.io/seps/2640-skills-extension.md)
- [MCP Skills Extension Overview](https://modelcontextprotocol.io/extensions/skills/overview)
- [ext-skills repository](https://github.com/modelcontextprotocol/ext-skills)
- [MCP Python SDK](https://github.com/modelcontextprotocol/python-sdk)
- [Python SDK skills extension PR #3485](https://github.com/modelcontextprotocol/python-sdk/pull/3485)
- [Agent Skills for Python is now released (Microsoft Agent Framework)](https://devblogs.microsoft.com/agent-framework/agent-skills-for-python-is-now-released/)
- [Scaling AI agents with open standards: MCP and the AAIF (Google Open Source Blog)](https://opensource.googleblog.com/2026/11/scaling-ai-agents-open-standards-mcp-aaif.html)
