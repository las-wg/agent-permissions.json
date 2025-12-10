# agent-permissions.json

A robots.txt-inspired standard for how agents may interact with webpages. Made with ❤️ by the [Lightweight Agent Standards Working Group](https://las-wg.org/).

Note: This standard is still WIP! If you're interested in contributing, join our [Discord](https://discord.gg/wmRSNHsRAh)!

---

# Introduction

Websites need a machine-readable, non-binding way to declare how automated agents (bots, crawlers, assistants) may interact with them. This **JSON-only Agent Permissions File** lets sites express fine-grained resource-level rules and higher-level action guidelines in a single, versioned document.

This document does not cover agent-to-agent interactions or content usage policies (see [IETF AIPref](https://datatracker.ietf.org/wg/aipref/about/) instead).

The key words **“MUST”**, **“MUST NOT”**, **“SHOULD”**, and **“SHOULD NOT”** are to be interpreted as described in RFC 2119.

This file is advisory and non-binding: agents MAY choose stricter behaviors than those granted here, but MUST NOT exceed the permissions granted if they claim to honor this format.

# File Discovery

Consumers **MAY** locate and fetch the JSON policy via a link tag, e.g.:

```html
<link rel="agent-permissions" href="https://example.com/path/to/agent-permissions.json">
```

In case of no `<link>` tag, the default path is `/.well-known/agent-permissions.json`. Servers **SHOULD** serve this file with `Content-Type: application/json`.

# Rationale

There are two forms of permissions:

* **Resource permissions**: how you can interact with HTML elements (clicking, following links, filling forms, controlling media, uploading files…).
* **Action permissions**: what you can do on the web page (sending DMs, booking a flight, posting content…).

The former are described using `resource_rules`, which is a list containing, for each element:

* A `verb` (e.g., `"click_element"` or `"follow_link"`)
* A CSS selector that restricts what the rule applies to
* Whether the action is `allowed` or forbidden
* (Optionally) `modifiers`, e.g. a limit on how many actions can be performed at the same time or whether it requires explicit user consent

The top-level field `strict` prescribes the default behavior for an action that is not explicitly allowed or forbidden (`true`: defaults to forbidden, `false`: defaults to allowed).

The latter are described using `action_guidelines`, which is a list containing, for each element:

* An RFC-2119 style `directive` (`"MUST"`, `"MUST NOT"`, `"SHOULD"`, `"SHOULD NOT"`)
* A natural language `description` of the action (e.g. “Book a flight”)
* A natural language description of `exceptions`

Implementation-wise, agents can follow `resource_rules` at the interface level (e.g. if the agent is using a headless browser to perform an action, the code connecting the agent to the browser can act as an enforcement layer), while `action_guidelines` are provided as prompt or configuration to the agent.

Some higher-level behaviors (for example, “fill this form”) are expected to be **composite** operations built from primitive verbs (e.g. multiple `set_input_value` calls followed by `submit_form`).

Finally, the field `api` provides information on what actions can be performed by calling HTTP APIs (e.g. REST) instead of interacting with the website, as the former is often more efficient and reliable than the latter.

# Verbs

The `verb` in each `resource_rules` entry MUST be one of the following values. Future versions MAY extend this list; unknown verbs MUST be treated as disallowed.

* **`all`**
  Wildcard for all verbs.

* **`read_content`**
  Read the human-visible content of elements on the page (main body text, headings, labels, etc.).

* **`read_metadata`**
  Read non-body metadata associated with the page (e.g. `<title>`, meta tags, structured data, OpenGraph, JSON-LD).

* **`follow_link`**
  Navigate by activating a link element (e.g. `<a>`), causing a page load or navigation.

* **`click_element`**
  Synthesize a user click on the target element (buttons, links, controls, etc.), triggering any associated handlers.

* **`scroll_page`**
  Scroll the page viewport (or scrollable container) by a specified amount or direction.

* **`set_input_value`**
  Set the value or state of an input control. For text/number fields, sets the text/value; for checkboxes, sets checked/unchecked; for radio buttons, selects this option; for selects/dropdowns, chooses the specified option(s).

* **`submit_form`**
  Submit a form associated with the target element (e.g. the nearest enclosing `<form>`), triggering normal browser form submission behavior.

* **`execute_script`**
  Execute a script in the page context (e.g. JavaScript), subject to any additional sandboxing or policy constraints.

* **`play_media`**
  Start or resume playback of a media element (audio or video).

* **`pause_media`**
  Pause playback of a media element without resetting its position.

* **`mute_media`**
  Mute the audio of a media element.

* **`unmute_media`**
  Unmute the audio of a media element.

* **`upload_file`**
  Attach or provide a file to an upload-capable control (e.g. `<input type="file">`) or equivalent UI.

* **`download_file`**
  Initiate a file download via the target element (e.g. clicking a download link or button).

* **`copy_to_clipboard`**
  Copy specified text or content from the page to the agent’s clipboard abstraction.

# JSON Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "Agent Permissions File",
  "type": "object",
  "required": ["metadata", "strict"],
  "properties": {
    "metadata": {
      "type": "object",
      "required": ["schema_version", "last_updated"],
      "properties": {
        "schema_version": {
          "type": "string",
          "pattern": "^\\d+\\.\\d+\\.\\d+$",
          "description": "Semantic version, e.g. '1.0.0'"
        },
        "last_updated": {
          "type": "string",
          "format": "date-time",
          "description": "ISO-8601 timestamp of this file’s publication"
        },
        "author": {
          "type": "string",
          "description": "Optional human-readable identifier for the publisher"
        }
      },
      "additionalProperties": false
    },
    "strict": {
      "type": "boolean",
      "description": "true = default behavior is to forbid the action, false = default behavior is to allow"
    },
    "resource_rules": {
      "type": "array",
      "description": "Structured, machine-enforceable rules for low-level interactions",
      "items": {
        "type": "object",
        "required": ["verb", "selector", "allowed"],
        "properties": {
          "verb": {
            "type": "string",
            "enum": [
              "all",
              "read_content",
              "read_metadata",
              "follow_link",
              "click_element",
              "scroll_page",
              "set_input_value",
              "submit_form",
              "execute_script",
              "play_media",
              "pause_media",
              "mute_media",
              "unmute_media",
              "upload_file",
              "download_file",
              "copy_to_clipboard"
            ],
            "description": "Type of interaction; MUST be one of the defined verbs"
          },
          "selector": {
            "type": "string",
            "description": "CSS selector for the rule."
          },
          "allowed": {
            "type": "boolean",
            "description": "true = permit; false = forbid"
          },
          "modifiers": {
            "type": "object",
            "description": "Optional constraints on frequency, timing, human oversight, etc.",
            "properties": {
              "burst": {
                "type": "integer",
                "minimum": 1,
                "description": "Max concurrent actions for the agent instance"
              },
              "rate_limit": {
                "type": "object",
                "description": "Rate limit for the agent instance",
                "properties": {
                  "max_requests": {
                    "type": "integer",
                    "minimum": 1,
                    "description": "Maximum number of allowed actions in the window"
                  },
                  "window_seconds": {
                    "type": "integer",
                    "minimum": 1,
                    "description": "Length of the time window (in seconds)"
                  }
                },
                "additionalProperties": false
              },
              "time_window": {
                "type": "string",
                "pattern": "^[0-2]\\d:[0-5]\\d-[0-2]\\d:[0-5]\\d UTC$",
                "description": "Clock-time range, e.g. '08:00-20:00 UTC'"
              },
              "human_in_the_loop": {
                "type": "boolean",
                "description": "true = requires explicit human confirmation"
              }
            },
            "additionalProperties": false
          }
        },
        "additionalProperties": false
      }
    },
    "action_guidelines": {
      "type": "array",
      "description": "Semi-structured RFC 2119 directives for higher-level behaviors",
      "items": {
        "type": "object",
        "required": ["directive", "description"],
        "properties": {
          "directive": {
            "type": "string",
            "enum": ["MUST", "MUST NOT", "SHOULD", "SHOULD NOT"],
            "description": "Normative strength"
          },
          "description": {
            "type": "string",
            "description": "Human-readable explanation of the permitted or forbidden action"
          },
          "exceptions": {
            "type": "string",
            "description": "Optional explanation of exceptions"
          }
        },
        "additionalProperties": false
      }
    },
    "api": {
      "type": "array",
      "description": "Endpoints that can be used as an alternative to interacting with the webpage",
      "items": {
        "type": "object",
        "properties": {
          "type": {
            "type": "string",
            "enum": ["openapi", "mcp", "a2a"],
            "description": "Type of reference"
          },
          "endpoint": {
            "type": "string",
            "description": "URL to the endpoint"
          },
          "docs": {
            "type": "string",
            "description": "URL to the endpoint documentation"
          },
          "description": {
            "type": "string",
            "description": "Description of the endpoint in the context of the web page"
          }
        },
        "required": ["type", "endpoint", "description"],
        "additionalProperties": false
      }
    }
  },
  "additionalProperties": false
}
```

---

# Semantics & Enforcement

1. **Fetch & cache.**
   Fetch the file and cache it, observing any HTTP caching headers.

2. **Validate.**
   Parse according to the schema; on validation failure, consumers **SHOULD** treat the file as absent and **MUST NOT** grant additional permissions based on it (i.e., disallow all interactions beyond their default safety policies).

3. **Apply `resource_rules`.**
   Apply `resource_rules` following priority rules:
   1. Specific verbs have higher priority over `all`
   2. For the same verb, follow CSS priority rules

4. **Honor modifiers.**
   Enforce `modifiers` such as `burst`, `rate_limit`, `time_window`, and `human_in_the_loop` when deciding whether to execute an action.

5. **Interpret `action_guidelines`.**
   Log or otherwise surface violations of `action_guidelines`:

   * `MUST NOT` → error log
   * `SHOULD NOT` → warning
   * `SHOULD` → info

We expect the implementation of `resource_rules` to happen at the browser interface level (e.g. the agent is blocked from performing unintended interactions) and the implementation of `action_guidelines` to happen at the agent level (e.g. by feeding these rules to the LLM or equivalent decision component).

---

# Example

```json
{
  "metadata": {
    "schema_version": "1.0.0",
    "last_updated": "2025-06-01T12:00:00Z",
    "author": "Example Corp"
  },
  "strict": false,
  "resource_rules": [
    {
      "verb": "read_content",
      "selector": "*",
      "allowed": true
    },
    {
      "verb": "follow_link",
      "selector": ".private-area",
      "allowed": false
    },
    {
      "verb": "click_element",
      "selector": "#buy",
      "allowed": true,
      "modifiers": {
        "burst": 5,
        "time_window": "08:00-20:00 UTC"
      }
    },
    {
      "verb": "set_input_value",
      "selector": "form#checkout input[name='email']",
      "allowed": true,
      "modifiers": {
        "human_in_the_loop": true
      }
    },
    {
      "verb": "play_media",
      "selector": "video.hero",
      "allowed": false
    }
  ],
  "action_guidelines": [
    {
      "directive": "MUST NOT",
      "description": "Send unsolicited direct messages to end users.",
      "exceptions": "MAY message site administrators."
    },
    {
      "directive": "SHOULD",
      "description": "Add \"_bot\" to the username when registering an account."
    }
  ]
}
```
