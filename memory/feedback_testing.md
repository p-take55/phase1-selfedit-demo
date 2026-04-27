---
name: TypeScript プロジェクトのテスト方針
description: TypeScript プロジェクトにおける alert 系テストのスキップ可否
type: feedback
---

TypeScript プロジェクトでは alert 系のテストは skip していい。

**Why:** ユーザーの明示的な指示。ブラウザ環境依存の alert は JSDOM 等のテスト環境で動作しないケースが多く、skip で問題ない。

**How to apply:** TypeScript プロジェクトで `alert()`、`confirm()`、`prompt()` 等のブラウザ alert 系 API に関するテストが出てきたら、skip または未テストのままにしてよい。
