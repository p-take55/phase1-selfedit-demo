---
name: questions
description: >
  This skill should be used when the user asks to "ask me questions", "let me choose",
  "help me decide", "collect my opinion", "gather structured answers", "make a decision with me",
  "ユーザーに質問したい", "意思決定してもらいたい", "意見を聞きたい", "判断に悩む",
  or when durable user input should be collected through shared-document `questions`
  instead of an ad-hoc chat reply.
---

# questions

Use shared-document `frontmatter.questions[]` to collect user decisions in a
durable, structured way. Prefer this skill when the result should stay attached
to a document, be revisited later, or be answered in the WebUI by clicking
choices.

Do not use this skill for every clarification. For a single quick question that
does not need to persist, ask directly in chat. Reach for `questions` when the
answer should become part of the document itself.

## When to use

Use this skill for cases like:

- Ask the user to choose among design directions
- Ask for prioritization or tradeoff decisions
- Collect structured answers to multiple prompts
- Confirm preferences that should remain documented
- Surface uncertainty instead of silently guessing

Good fits:

- "どの案で進めるか決めてほしい"
- "この仕様の選択肢を並べて意見をもらいたい"
- "複数設問で回答してもらいたい"
- "判断に迷うので、選択肢つきで確認したい"

Avoid this skill when:

- One short free-form clarification in chat is enough
- The question is urgent and blocking right now, with no need to persist it
- The choice is purely internal to the agent and does not require user judgment

## Core idea

`questions` is not a new kind. Add it to any shared document that already holds
the surrounding context. Keep the context in the markdown body and the actual
choice set in frontmatter.

Model the data like this:

```yaml
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
      - id: full_restart
        label: Full restart
    answer:
      selected: []
      note: 必要なら補足を書く
```

Keep `answer.selected` as option ids, not labels.

## Recommended workflow

### 1. Decide whether `questions` is the right tool

Ask:

- Should the answer persist in the document?
- Should multiple options be comparable at a glance?
- Should the user answer through the WebUI instead of replying in prose?

If yes to at least one, `questions` is likely a good fit.

### 2. Reuse an existing kind

Do not introduce a new kind just to ask a question. Pick the shared document
that already owns the decision context:

- spec / proposal / issue / note / review / ADR / any existing project kind

If the kind lacks `questions` support, update `_aachat.schema` inside its
`_template.md` rather than creating a questionnaire-only kind.

### 3. Write good questions

Prefer a small set of meaningful prompts over a survey dump.

Keep each question:

- focused on one decision
- written in user language, not schema language
- answerable from the document context already present

Write option labels so they can stand alone in the WebUI. Put nuance in
`description`, not in overloaded labels.

Good:

- `label: Canary`
- `description: 一部ユーザーから段階展開`

Bad:

- `label: Canary because safety matters and blast radius is smaller`

### 4. Choose `single` vs `multiple`

Use `selection: single` when exactly one path should be chosen.

Use `selection: multiple` when the user can select several acceptable items,
for example:

- desired monitoring dimensions
- supported rollout constraints
- acceptable follow-up tasks

Do not abuse `multiple` to avoid making a decision. If one decision is needed,
use `single`.

### 5. Seed the answer state correctly

Use:

- `answer.selected: []` when no answer exists yet
- pre-filled ids only when the answer is already known from prior discussion

Do not invent user answers. If uncertain, leave the selection empty.

### 6. Explain what the user is deciding

Put the background, tradeoffs, and consequences in the markdown body below the
frontmatter. The body should answer:

- Why this question exists
- What each option changes
- What is out of scope

Keep frontmatter for structure; keep reasoning in the body.

## Authoring guidance

When generating or editing a document that uses `questions`:

1. Ensure the current kind's `_template.md` `_aachat.schema` accepts `questions`
2. Ensure `_template.md` or the target document includes the desired questions
3. Keep validation light: require ids, prompt, selection, options
4. Let the WebUI enforce click behavior for `single` / `multiple`
5. Save answers by rewriting the same document, not by creating proposal
   actions or workflow state machines

## Communication guidance

When presenting the document to the user:

- Say what decision is being requested
- Say where the answer should be given
- Point to the existing document instead of duplicating options in chat

Example phrasing:

- "判断が必要な点をこのドキュメントの `questions` に入れました。WebUI で選択肢を選ぶと、そのままドキュメントに保存されます。"
- "背景は本文に、選択は `questions` にまとめています。迷う点だけ選んでもらえれば次に進めます。"

## Non-goals

Do not turn `questions` into:

- a workflow engine
- proposal actions
- automatic state transitions
- command execution

Keep it simple: user clicks choices, the same document is updated.

## Heuristics

Prefer `questions` over direct chat when:

- the decision should remain attached to a document
- several options need side-by-side comparison
- another person may revisit the answer later

Prefer direct chat over `questions` when:

- one sentence of clarification is enough
- the answer is ephemeral
- the document would become noisier rather than clearer

## Minimal checklist

Before using `questions`, verify:

- the document already has or should have decision context
- the kind schema allows `questions`
- each question has `id`, `prompt`, `selection`, `options`
- each option has `id`, `label`
- unknown user answers are left empty
- the body explains enough context for the user to decide
