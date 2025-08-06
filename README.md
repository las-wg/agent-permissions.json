# agent-permissions.json

## Introduction

Websites need a machine-readable, non-binding way to declare how automated agents (bots, crawlers, assistants) may interact with them. This JSON-only Agent Permissions File lets sites express fine-grained resource-level rules and higher-level action guidelines in a single, versioned document.

> **Note:** This document does not cover agent-to-agent interactions or content usage policies (see IETF AIPref instead).

---

## File Discovery

Consumers **MAY** locate and fetch the JSON policy via a `<link>` tag, for example:

```html
<link rel="agent-permissions"
      href="https://example.com/.well-known/agent-permissions.json">
```

---

## Rationale

There are two forms of permissions:

1. **Resource permissions:**
   How you can interact with HTML elements (clicking, following links, filling forms…).

2. **Action permissions:**
   What you can do on the web page (sending DMs, booking a flight…).

On top of them, we also provide **manifest metadata** and **API indicators**.

### Manifest Metadata

Basic metadata for the manifest, specifically:
* `author`
* `schema_version`
* `last_updated`

### Resource Permissions (`resource_rules`)

A list of rules, each specifying:

* `verb`:
  One of the standard interaction types (`read_content`, `follow_link`, `click_element`, `fill_form`, `submit_form`, `execute_script`) or a custom verb.
* `selector`:
  A CSS or XPath-style selector that restricts what the rule applies to.
* `allowed`:
  `true` to permit; `false` to forbid.
* `modifiers` (optional):
  Additional constraints, such as:

  * `burst`: max concurrent actions
  * `rate_limit`: `{ max_requests, window_seconds }`
  * `time_window`: e.g. `"08:00–20:00 UTC"`
  * `human_in_the_loop`: requires explicit human confirmation

### Action Permissions (`action_guidelines`)

A list of higher-level directives, each including:

* `directive`: (RFC-2119 style)
  One of `MUST`, `MUST NOT`, `SHOULD`, `SHOULD NOT`.
* `description`:
  A human-readable explanation of the action.
* `exceptions` (optional):
  Natural-language description of any exceptions.

### API Indicators

When the webpage offers a service that can also be accessed through API, it is possible to specify links to documentation for the corresponding API. Each item in the list includes:
* `endpoint`: URL to the endpoint
* `description`: A description of the endpoint in the context of the webpage
* `type`: Type of documentation (only OpenAPI supported by default)
* `docs`: Link to the documentation

---

## JSON Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "Agent Permissions File",
  "type": "object",
  "required": ["metadata", "resource_rules"],
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
      }
    },
    "custom_verbs": {
      "description": "Optional description of non-standard verbs for resource_rules",
      "type": "object",
      "additionalProperties": {
        "type": "string"
      }
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
              "read_content",
              "follow_link",
              "click_element",
              "fill_form",
              "submit_form",
              "execute_script"
            ],
            "description": "Type of interaction. May also be one of the keys in custom_verbs"
          },
          "selector": {
            "type": "object",
            "required": ["type", "value"],
            "properties": {
              "type": {
                "type": "string",
                "enum": ["css", "xpath", "all"],
                "description": "'all' equals '*' wildcard"
              },
              "value": {
                "type": "string",
                "description": "CSS selector, XPath expression, or ignored if type='all'"
              }
            }
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
                "description": "Max concurrent actions"
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
                }
              },
              "time_window": {
                "type": "string",
                "pattern": "^[0-2]\\d:[0-5]\\d-[0-2]\\d:[0-5]\\d UTC$",
                "description": "Clock-time range, e.g. '08:00–20:00 UTC'"
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
        "required": ["type", "endpoint", "description"],
        "properties": {
          "type": {
            "type": "string",
            "enum": ["openapi"],
            "description": "Type of reference. Only OpenAPI is supported so far"
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
        }
      }
    }
  },
  "additionalProperties": false
}
```

---

## Semantics & Enforcement

1. **Fetch & cache once.**
   Observe any HTTP caching headers.
2. **Parse according to the schema.**
   On validation failure, log an error and disallow all interactions.
3. **Apply `resource_rules`** in order of appearance.
4. **Honor modifiers.**
5. **Log `action_guidelines` violations:**

   * `MUST NOT` → error log
   * `SHOULD NOT` → warning
   * `SHOULD` → info

> **Implementation notes:**
>
> * Resource rules are enforced at the browser interface level (e.g., blocking unintended interactions).
> * Action guidelines are enforced at the agent level (e.g., fed to the LLM as prompts).

---

## Example

```json
{
  "metadata": {
    "schema_version": "1.0.0",
    "last_updated": "2025-06-01T12:00:00Z",
    "author": "Example Corp"
  },
  "resource_rules": [
    {
      "verb": "read_content",
      "selector": { "type": "all", "value": "" },
      "allowed": true
    },
    {
      "verb": "follow_link",
      "selector": { "type": "css", "value": ".private-area" },
      "allowed": false
    },
    {
      "verb": "click_element",
      "selector": { "type": "xpath", "value": "//button[@id='buy']" },
      "allowed": true,
      "modifiers": {
        "burst": 5,
        "time_window": "08:00–20:00 UTC"
      }
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
  ],
  "api": [
    {
      "type": "openapi",
      "endpoint": "https://api.example.com/v1/send-message",
      "docs": "https://docs.example.com/v1/send-message",
      "description": "Sends a message to the user. Equivalent to using the DM feature."
    }
  ]
}
```
