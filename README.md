# prompts

Prompts for Agentic AI × Clean Architecture.

## 説明

- `.promts/`にプロンプトが格納されています。
- CLAUDE.mdにタスク実施前に`.prompts/`を読み込むように指示を書いているため、AI Agentは適宜必要なファイルを読み込んでから作業に着手するようになっています。
- Claude Code以外のAgentic AIを使用する場合は、`CLAUDE.md`のファイル名を書き換えてください。
- 各ファイルに技術スタックに関する記載があるため、適宜書き換えて使用してください。まず[01_inception_deck.md](./01_inception_deck.md)を書いて、「これに基づいて書き換えて」という風に他ファイルをAIに投げるのが良いと思います。

| ファイル名 | 概要 | ターゲット開発者 |
| :--- | :--- | :--- |
| 01_ubiquitous.md | プロジェクト全体で共通して使用するビジネス用語（ユビキタス言語）の定義ファイル。コードの命名規則の基礎となります。 | 全員 |
| 10_backend.md | バックエンド（Python/FastAPI）の技術スタック、Clean Architectureの適用方法、実装原則、ディレクトリ構造、およびテスト戦略を定義したガイドライン。 | B/E Dev (Python/FastAPI) |
| 11_performance.md | DAUに基づいたキャパシティプランニング、k6を用いた負荷テスト戦略、および再現性の高い負荷テスト用データ準備（シーディング）のガイドライン。 | B/E Dev / DevOps |
| 20_frontend.md | フロントエンド（React/TypeScript）の技術スタック、Clean Architecture + Atomic Designの適用方法、Smart vs Dumb ルール、およびテスト戦略（Vitest）を定義したガイドライン。 | F/E Dev (React/TS) |
| 21_design.md | デザインシステムにおける色やフォント、スペーシングなどのデザインに関する記述を追加するためのプレースホルダーファイル。 | F/E Dev / Designer |
| 30_BDD.md | AIにシニアBDDアーキテクトの役割を担わせるためのコアプロンプト。Gherkin作成における「Discovery → Formulation」の厳格なプロセスを定義します。**機能開発時はまず人間がこのプロンプトを使ってAIにGherkinファイルを生成させ、レビューする必要があります。** Agentic AIは、生成させたGherkinを仕様として実装を行います。 | AI / BDD QA |
| 31_E2E.md | Playwrightを用いたE2Eテストのガイドライン。安定したテストのためのdata-testid の命名規則（Test ID Policy）やデータ管理戦略を定義。 | QA / F/E Dev |
