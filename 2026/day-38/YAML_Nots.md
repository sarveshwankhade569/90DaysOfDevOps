# YAML Notes — The Basics

## What is YAML?

YAML (YAML Ain't Markup Language) is a human-readable format used for configuration. You'll run into it constantly in DevOps:

- GitHub Actions
- GitLab CI/CD
- Docker Compose
- Kubernetes
- Ansible
- Helm

**The one rule that matters most:** YAML uses indentation to define structure — and it must be **spaces, never tabs**. A tab character will break the file.

## Key-value pairs

```yaml
name: Sarvesh
role: DevOps Engineer
experience_years: 0
learning: true
```

Simple `key: value` pairs. One thing worth noticing:
```yaml
learning: true      # a boolean
learning: "true"    # a string
```
Without quotes, `true`/`false` are booleans. With quotes, they're just text. This distinction matters a lot once a tool (like Kubernetes) is actually reading the value and expecting a real boolean.

## Lists

Two ways to write the same list:

**Block style** (most common, easiest to read):
```yaml
tools:
  - Linux
  - Git
  - Docker
```

**Inline/flow style** (compact, fine for short lists):
```yaml
hobbies: [learning, coding, reading]
```

## Nested objects

Indentation is what tells YAML "this belongs to that":
```yaml
server:
  name: web-server
  ip: 192.168.1.10
  port: 80

database:
  host: db.example.com
  name: devops_db
  credentials:
    user: devops
    password: secret123
```
`host` and `name` belong to `database` because they're indented under it. `user` and `password` belong to `credentials` for the same reason — one level deeper.

⚠️ **Never mix tabs and spaces.** This looks fine to the eye but breaks:
```yaml
database:
	host: db.example.com
```
YAML parsers will throw something like `found character '\t' that cannot start any token`. Always use spaces.

## Multi-line strings

Two operators for writing text across multiple lines:

**`|` — preserve line breaks** (great for shell scripts, configs):
```yaml
startup_script: |
  #!/bin/bash
  echo "Starting application"
  systemctl start nginx
```
This keeps every line break exactly as written — useful for scripts or ConfigMaps where line structure matters.

**`>` — fold lines into one** (great for long paragraphs):
```yaml
description: >
  This is a long description
  that is written across
  multiple YAML lines.
```
This becomes one continuous sentence: *"This is a long description that is written across multiple YAML lines."*

| Operator | Meaning | Good for |
|---|---|---|
| `\|` | Preserve line breaks | Scripts, configs |
| `>` | Fold into one line | Long paragraphs/descriptions |

## Validating YAML

Install a linter:
```bash
sudo apt update
sudo apt install yamllint -y
```

Check a file:
```bash
yamllint server.yaml
```

If your indentation is off — say, one key is indented further than it should be — `yamllint` will flag it immediately. This is worth doing before feeding a YAML file into any real tool, since a small indentation mistake can silently produce a completely different structure instead of an obvious crash.

## A common mistake: inconsistent list indentation

**Broken:**
```yaml
tools:
- docker
  - kubernetes
```
**Correct:**
```yaml
tools:
  - docker
  - kubernetes
```
Both list items belong to `tools`, so they need to sit at the *same* indentation level. Think of it as a small tree:
```
tools
├── docker
└── kubernetes
```
Siblings need to line up.

## Cheat sheet

```yaml
# key-value
name: Sarvesh

# list (block style)
tools:
  - Linux
  - Docker

# list (inline style)
tools: [Linux, Docker]

# nested object
server:
  name: web
  port: 80

# boolean vs string
learning: true      # boolean
learning: "true"    # string

# multi-line, preserve breaks
script: |
  echo "Hello"
  echo "World"

# multi-line, fold into one
description: >
  This is a long
  description.
```

```bash
yamllint file.yaml   # validate
cat file.yaml        # view
```

## The 5 rules to remember

1. YAML is indentation-sensitive — structure comes from spacing.
2. Use spaces, never tabs.
3. Items at the same level need the same indentation.
4. `|` preserves line breaks; `>` folds them into one line.
5. Always validate with `yamllint` before using a file in CI/CD.

Note: never commit real passwords or secrets in a YAML file — use environment variables or a secrets manager instead. Any example with plaintext credentials here is for practice only.
