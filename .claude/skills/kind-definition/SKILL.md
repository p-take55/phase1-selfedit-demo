---
name: kind-definition
description: >
  Use when defining, updating, or removing a shared-document kind for the current project.
  Triggers on: "add a kind", "define a kind", "new document type", "define a shared doc kind",
  "create kind directory", "_template.md", or when the user asks to support
  a document type that doesn't exist yet under docs/<team>/<project>/.
  Also use when `_template.md` should support `questions`,
  questionnaires, user decision prompts, or structured multiple-choice answers.
---

# Shared-document kinds

A shared-document kind is a type of document (e.g. `problem`, `notes`, `proposal`).
Each kind is a directory under `docs/<team>/<project>/` containing:

- `_template.md` - `_aachat` metadata plus the template new docs start with

Existing (defined) kinds can be discovered with `ls docs/<team>/<project>/` and
reading each directory's `_template.md`.

## Defining a new kind

1. Pick a name matching `^[a-z][a-z0-9_-]*$` (1-32 chars). Do not use `tasks` or names
   starting with `_`.
2. `mkdir <kind>` under the project directory.
3. Write `<kind>/_template.md`:

       ---
       _aachat:
         template_policy: always_overwrite  # or create_once
         allowed_states: [draft, published] # optional
         schema:
           type: object
           required: [severity]
           properties:
             severity: { type: string, enum: [low, medium, high] }
       severity: medium
       ---
       ## Situation

## Updating an existing kind

Edit `<kind>/_template.md` in place. Changes are auto-saved.

## Supporting `questions` in any kind

Do not create a dedicated kind just to ask users questions. `questions` is a
kind-independent frontmatter field that can be added to any existing kind when
that document should collect user choices in the WebUI.

When `_template.md` should support questions, add a `questions` property to
`_aachat.schema` and keep validation intentionally light:

      ---
      _aachat:
        template_policy: always_overwrite
        schema:
          type: object
          additionalProperties: true
          properties:
            questions:
              type: array
              minItems: 1
              items:
                type: object
                required: [id, prompt, selection, options]
                additionalProperties: true
                properties:
                  id:
                    type: string
                    pattern: "^[a-z][a-z0-9_-]*$"
                  prompt:
                    type: string
                    minLength: 1
                  selection:
                    type: string
                    enum: [single, multiple]
                  required:
                    type: boolean
                  options:
                    type: array
                    minItems: 1
                    items:
                      type: object
                      required: [id, label]
                      additionalProperties: true
                      properties:
                        id:
                          type: string
                          pattern: "^[a-z][a-z0-9_-]*$"
                        label:
                          type: string
                          minLength: 1
                        description:
                          type: string
                  answer:
                    type: object
                    additionalProperties: true
                    properties:
                      selected:
                        type: array
                        items:
                          type: string
                      note:
                        type: string
      ---

Keep `questions` flexible. Do not try to encode UI behavior such as
"single means exactly one selected" in schema - the JSON Schema subset cannot
express that cleanly. Let schema guarantee only shape, and let the WebUI enforce
selection behavior.

When `_template.md` should start with user-facing questions, seed the document
with `questions` in frontmatter:

      ---
      questions:
        - id: rollout_strategy
          prompt: どのリリース方式を採用する？
          selection: single
          required: true
          options:
            - id: canary
              label: Canary
              description: 一部ユーザーから段階展開
            - id: rolling
              label: Rolling
          answer:
            selected: []
      ---
      ## 背景

Use empty `selected: []` as the default unless an answer is already known.
Store option identifiers in `answer.selected`, not labels, so text can change
without invalidating existing answers.

## Removing a kind

Remove `<kind>/_template.md`. Existing docs stay on disk and remain readable as
raw markdown.

## JSON Schema subset (allowed keywords)

- `type`: object, string, number, integer, boolean, array, null
- `properties`, `required`, `additionalProperties` (bool only)
- `enum` (only on string / number / integer)
- `pattern`, `minLength`, `maxLength` (string only)
- `minimum`, `maximum` (number / integer)
- `items` (single schema, not tuple), `minItems`, `maxItems`
- `format`: `date` | `date-time` | `uri` | `aachat.*` (UI hints, see below)

Rejected: `$ref`, `$defs`, `$schema`, `$id`, `oneOf`, `anyOf`, `allOf`, `not`,
`if`/`then`/`else`, `dependencies`, `patternProperties`, `propertyNames`, tuple items.

## aachat-provided formats

`format: aachat.*` values are UI hints for the WebUI. They do NOT affect
validation - `type`, `enum`, `min/max` etc. still do all the checking.
Unknown `format` values (including future `aachat.*` additions not in the
WebUI registry yet) fall back to a primitive renderer and are never rejected
by the server.

| format              | applies to            | WebUI rendering                   |
|---------------------|-----------------------|-----------------------------------|
| `aachat.badge`      | `string` + `enum`     | colored badge (color auto-chosen) |
| `aachat.chips`      | `array<string>`       | chip list                         |
| `aachat.member_ref` | `string` (user id)    | avatar + name                     |
| `aachat.agent_ref`  | `string` (agent id)   | avatar + name                     |
| `aachat.doc_ref`    | `string` (wiki target)| wiki-link chip                    |

Example:

       severity:
         type: string
         enum: [low, medium, high, critical]
         format: aachat.badge

`questions` does not use `format: aachat.*`. The WebUI recognizes the top-level
`questions` key directly and renders a dedicated questionnaire block when the
shape is valid.

## Validation feedback

All validation feedback is collected in a single file: `<project>/_errors.md`.

- The file's presence = something is wrong. Its absence = all good.
- Structure:
  - `## Errors` - hard errors grouped by `### Kind: <name>` and `### Project-wide`
  - `## Warnings` - non-fatal notices, same grouping
  - `## Conflict` - only if a concurrent edit caused an If-Match failure
- Fix the indicated `<kind>/_template.md` and save. The
  relevant section disappears automatically. When all sections are gone, the file
  itself is removed.

Do not create or edit this file by hand - it is managed by aachat.

## Coordination

For destructive changes (removing kinds that still have documents, tightening schemas
that invalidate existing docs, or deleting the entire definition), coordinate with a
human first. `DELETE /kind-definition` is available to the agent runtime too, but it
is a higher-blast-radius operation than per-kind remove.
