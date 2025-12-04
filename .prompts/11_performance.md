
# Performance & Testing Guidelines

B/Eのテスト実装タスクでは本ドキュメントを参照してください。

## 1. Capacity Planning & Metrics

本プロジェクトのキャパシティ計画は、**Base DAU** を唯一の変数として算出します。
全てのメトリクスは、DAUに対する係数（CRUD Rate / User Behavior）によって定義されます。

### A. Base Assumptions (Variables)

これらの値はプロジェクトのフェーズや性質に応じて調整してください。

| Variable | Value (Example) | Description |
| :--- | :--- | :--- |
| **Base DAU** | **50** | 想定する1日あたりのアクティブユーザー数 |
| **Creator Rate** | 10% | 新規エンティティの作成（Create）を行うユーザーの割合 |
| **Creates / User** | 2.0 | アクティブユーザー1人あたりの平均作成数/日 |
| **Updates / Entity** | 20 | 1つのエンティティに対して行われる更新アクション（Update）の平均回数 |

### B. Calculated Metrics (Daily)

上記係数に基づき、1日あたりに生成されるデータ量を算出します。

| Metric | Formula | Value (DAU=50) | Description |
| :--- | :--- | :--- | :--- |
| **Active Creators** | `DAU * Creator Rate` | **5** | 実際にデータ作成を行う人数 |
| **Total New Entities** | `Active Creators * Creates/User` | **10** | 1日に新規作成されるエンティティの総数 (INSERT) |
| **Total Updates** | `Total New Entities * Updates/Entity` | **200** | 1日に発生する更新操作の総数 (UPDATE) |

### C. Stress Scenarios (The Peak Moment)

負荷試験で保証すべき「瞬間最大風速」の定義。
DAUの規模にかかわらず、**「全ユーザーが同時に集合する」** シチュエーションを前提とします。

| Scenario Metric | Formula | Value (DAU=50) | Description |
| :--- | :--- | :--- | :--- |
| **Max CCU** | `DAU * 100%` | **50** | 全員同時接続状態 |
| **Peak Write RPS** | `Max CCU * 1 req/sec` | **50 req/sec** | 全員が秒間1回の書き込み（Create/Update）を行う状態 |
| **Peak Read RPS** | `Max CCU * 2 req/sec` | **100 req/sec** | 全員が秒間2回の読み取り（Read/List）を行う状態 |

---

## 2. Load Testing Strategy (k6)

機能テスト（Pytest）とは分離し、上記「Stress Scenarios」を満たせるかを検証します。

### A. Test Scenarios

1. **Smoke Test:**
    * **負荷:** 1 VU (Minimal)
    * **目的:** エラー率 0% であることの確認。
2. **Spike Test (最重要):**
    * **負荷:** 0 VU -> `Max CCU` (in 10s) -> Stay -> 0
    * **目的:** DBコネクションプールの枯渇、オートスケールの追従性確認。
3. **Soak Test:**
    * **負荷:** `Max CCU * 20%` (Average Load), Duration: 1h
    * **目的:** メモリリーク、長時間稼働時のレイテンシ悪化の検知。

---

## 3. Seeding Strategy (Data Preparation)

シーディングデータ量もマジックナンバーを使用せず、**「想定運用期間」** に基づいて算出します。
これにより、フェーズに応じた適切な「DBの汚れ具合（インデックス負荷）」を再現します。

### A. Seeding Variables

| Variable | Value | Description |
| :--- | :--- | :--- |
| **Operation Years** | **3** | 何年運用した状態をシミュレートするか |
| **Days** | 1095 | `Operation Years * 365` |

### B. Target Data Volume

k6実行前にスクリプトにて投入するデータ量。
検索（Read）パフォーマンスへの影響を確認するため、アクティブデータとコールドデータを区別して投入します。

| Data Type | Formula | Value (Target) | Role in Testing |
| :--- | :--- | :--- | :--- |
| **Cold Entities** (Historical) | `Total New Entities/Day * Days` | **10,950** | **[Noise]** インデックス性能（Read Latency）の検証用重り。クエリ対象外となる古いデータを大量に配置する。 |
| **Hot Entities** (Active) | `Total New Entities/Day * 0.5` | **5** | **[Target]** 負荷試験時の更新対象（Update/Delete先）。キャッシュヒット率の検証などに使用する。 |

### C. Implementation Notes

* **Consistency:** DBへのデータ投入時は、整合性が必要な関連データ（カウンターやリレーション）も同時に初期化すること。
* **Tooling:** バルクインサート機能を活用し、テスト開始前に高速にセットアップを行うこと。
