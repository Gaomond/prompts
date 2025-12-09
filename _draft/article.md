## 本題

最近個人開発でWebサービスを作っていて、そこでAgentic Coding × Clean Architectureをやろうと試行錯誤していたら今回のプロンプトができたのでその知見を共有します。

### AIが微妙なコードを書くのはあなたのせいです

いきなり煽りみたいですみません。けど、これは僕自身がそういうコードを生成させまくってきたという話です。

AIはWeb上の平均的なコードを学習してる（んだと思う）ので、**プロンプトでコンテキストを絞ってあげない**と「密結合・マジックナンバーが多い・可読性低い...」みたいな微妙なコードが出てくる。これはChatGPT・Geminiのような対話型AIでもそうだし、Claude code・CodexのようなAgentic AIでも同じ。

↓こんなのとか。

```ts
import { Button } from "@/components/atoms/Button";
import { Input } from "@/components/atoms/Input";
import { MessageViewer } from "@/components/molecules/MessageViewer";
import type { HealthMessage } from "@/domain/model/health";

interface HealthCheckMonitorProps {
  inputValue: string;
  onInputChange: (value: string) => void;
  onSave: () => void;
  onFetch: () => void;
  message: HealthMessage | null;
  isLoading: boolean;
  error: Error | null | undefined;
}

export const HealthCheckMonitor = ({
  inputValue,
  onInputChange,
  onSave,
  onFetch,
  message,
  isLoading,
  error,
}: HealthCheckMonitorProps) => {
  return (
    <div className="space-y-6">
      <h1 className="text-2xl font-bold text-gray-900 text-center">
        System Health Check
      </h1>

     /* atomsをそのまま使ってますね... */
      <div className="flex flex-col gap-4 p-4 bg-gray-50 rounded-lg">
        <div className="flex-1">
          <Input 
            label="Target Endpoint"
            value={inputValue} 
            onChange={onInputChange}
            disabled={isLoading}
            placeholder="Enter health check endpoint..."
          />
        </div>
        <div className="flex justify-end gap-2">
          <Button 
            variant="secondary" 
            onClick={onSave} 
            disabled={isLoading}
          >
            Save Config
          </Button>
          <Button 
            variant="primary" 
            onClick={onFetch} 
            loading={isLoading}
          >
            Fetch Status
          </Button>
        </div>
      </div>

      /* ここは正しい（Moleculesを使ってる） */
      <MessageViewer message={message} isLoading={isLoading} error={error} />
    </div>
  );
};
```

よく言われることですが、**「AI使うにも知識が要る」** ってことだと思います。
逆に言えば、きちんと指示してやれば今のAIはかなりやってくれます。

### じゃあどうすればよいの？

プロンプトを工夫するだけでかなり良くなります。僕はUncle Bobのファンなので**Clean Architecture**でやります（別に好きな設計パターン何でもOKだと思います）。具体的には下記の通り。

#### 1\. 「Clean Architectureとはなにか」を全部書きます

単に「Clean Architectureを守ってください」とプロンプトに書くだけではAIはネット上の平均的（雑多......？）な「Clean Architecture」で実装します。

そうではなくて、**「僕の思うClean Architecture」** を教えてやる必要があります。例えば「DomainとUsecaseとAdapterがあって、DomainはUsecase以遠を関知してはいけない。内側の層が外側を知るときは、その実装ではなく、Interfaceを介してそれを利用する形をとる。ちなみに下記がそのルールに則って実装したサンプルコードで......」など。

それから、**Few-shot prompting**が結構効きます。

#### 2\. プロンプトを`F/E.md`、`B/E.md`、`E2E.md`...などの粒度で分割します

「このファイルにすべての指示を書いたから読んでね」だとAIが混乱することがあります。
「ファイルの前半ではAと指示されてるけど後半ではA'になってます」みたいな時は、AIは割と**どちらの指示も守らず適当にやる**、みたいなことを平気でしてきます。

F/Eはこのやり方、B/Eはこう、テストはこう......というふうに分けて、それぞれのファイルの最初に「F/E実装のときはこのファイル読んでね」と書いておくとAIが勝手に読み込んでくれます。

プロンプト作る側の人間としても、「やっぱF/Eはこういう設計方針にしよう」とか「あ、ここの解釈間違ってた」みたいなときにファイル分かれてたほうが修正しやすいです。

#### 3\. プロジェクト全体のコンテキストが分かるようなファイルも作成しておきます

AIは人間と違ってセッションごとにステートレスなので（ユーザーメモリ等の機能はありますが。）、「昨日こういうふうに実装してたのに今日は違う」みたいなことが割と起こります。

特にメソッド名の命名方法やUIデザインでそれが顕著です。

そこで「このPJはこういう背景と課題意識があって、こういう方法でそれを解決しようとしてるよ」みたいなのを書いておいてそれを毎回読むように指示しておくと、ある程度整合性が取れるようになります。今回は"project_definition.md"みたいなファイルを作ってそれを読ませることにします。

##### ユビキタス言語も定義しておきます

厳密なDDDの作法に則る必要は無いと思います。
ただ、プロダクトのコアとなる概念についてはユビキタス言語を定義しておくと、ファイル名・関数名・ドメインモデル名、更にはplaywright等でE2Eテストを作る際の{}の{}指定等で用語がズレず、トークンの無駄な消費を避けられます。

#### 4\. 「CLAUDE.md」などのAgentic AIが毎回必ず参照するファイルに、上記ファイル群を読むような指示を書いておきます

一生懸命指示を作っても、読んでくれないと意味がないので。

#### 5\. 静的解析ツールで縛れるルールは全部書きます

AIはFlakyですが、ruffやeslintっどのの静的解析ツールは**毎回必ず同じ基準で**エラーを検知してくれます。
「これこれこういうルールで型をつけてね」「こういうimportは禁止ですよ」とプロンプトで書くと中々守ってくれなかったりするんですが、静的解析ツールなら「Claudeさん、これ違反してますよ」と確実に指摘してくれます。
Agentic AIはこれら静的解析を実行することができるので、プロンプトに「実装中適宜`npm run check`を走らせて型安全に開発してね」などと書いておくと勝手にやってくれます。

##### 静的解析最大のメリット

AIにいちいち指摘入れなくても自動修正してくれる、というのも大きいんですが、個人的には **「最終的に出てきたコードは少なくとも静的解析は通ってる」** という状態を担保できるのが一番のメリットだと思ってて、この状態になるとAI生成コードのレビューの心理的負荷がぐっと下がります。
AIはコード生成の速度が速いのでこのことは重要です。

eslintなんかはパッケージごとのimportの方向もかなり細かく制御できる※ので、「静的解析が通ってる ＝ 少なくともDIPは守られてる」くらい厳格なルールにすることもできます。こうなると依存の方向はほとんど見なくて良くなるので、人間は純粋にドメインロジックが正しいか・効率的かなどのより価値のある部分に集中することができるようになります。

※例えばF/Eならこんな感じ。↓

```ts
  {
    files: ["src/domain/**/*.{ts,tsx}"],
    rules: {
      "no-restricted-imports": [
        "error",
        {
          patterns: [
            {
              group: ["**/usecase/**", "**/adapter/**", "**/components/**"],
              message:
                "❌ VIOLATION: Domain layer must be pure. Do not import from outer layers.",
            },
            {
              group: ["react", "react-dom"],
              message:
                "❌ VIOLATION: Domain layer must not depend on UI Framework (React).",
            },
          ],
        },
      ],
    },
  },

  // 2-2. Usecase
  {
    files: ["src/usecase/**/*.{ts,tsx}"],
    rules: {
      "no-restricted-imports": [
        "error",
        {
          patterns: [
            {
              group: ["**/components/**"],
              message:
                "❌ VIOLATION: UseCase layer must not depend on UI components.",
            },
            {
              group: ["**/adapter/**"],
              message:
                "❌ VIOLATION: UseCase must not depend on Adapter implementations. Use Domain interfaces (DIP).",
            },
          ],
        },
      ],
    },
  },

  // 2-3. Dumb Components
  {
    files: [
      "src/components/{atoms,molecules,organisms,templates}/**/*.{ts,tsx}",
    ],
    rules: {
      "no-restricted-imports": [
        "error",
        {
          patterns: [
            {
              // ロジック禁止（Propsで受け取れ）
              group: ["**/usecase/**", "@/usecase/**"],
              message:
                "❌ VIOLATION: Dumb components cannot import UseCases directly. Pass data/handlers via props.",
            },
            {
              group: ["**/adapter/**", "@/adapter/**"],
              message:
                "❌ VIOLATION: UI components cannot access Adapter layer directly.",
            },
          ],
        },
      ],
    },
  },
```

#### (おまけ) エラークラスやロギング用モジュール等は予め用意しておきます

アプリケーション全体の共通エラークラスを定義してそれをthrowしたり、ロギングモジュールを書いて統一的に構造化ログを取る、みたいなことはよくやると思いますが、ここもAIにやらせようとするとイメージしてるものを一発で出させるのが結構大変です。
ここは頑張って自分で書いて、プロンプトに「こういうの定義してるからそれ使って」と指示したほうが結果的に楽できます。

-----

### で、こんな感じのプロンプトになりました

長いのでここではB/Eだけ。記事後半で全体公開してます。今回はヘルスチェックのエンドポイントを実装させました。
「`/health`にClean Architectureなんて完全にオーバーエンジニアリングだ!」という声が聞こえてくる気がしますが、それは完全に正しいです。が、今回はあくまで例としてということで......

**B/E.mdの内容（抜粋）**

```markdown
# B/E.md
... (具体的な技術スタック等)

## 3. Architecture Pattern: Clean Architecture
**関心の分離**を徹底します。
「API/Infraの都合 (Adapter)」と「ビジネスロジック (UseCase)」と「データ定義 (Domain)」を明確に分割します。

### Layers & Dependency

依存の方向は **Adapter -> UseCase -> Domain** です。
円の中心に向かってのみ依存します。外側の変更が内側に影響を与えてはいけません。

1.  **Domain (Inner)**: `src/domain/`
    * アプリケーションのコアデータ型定義と、リポジトリのインターフェース定義。
    * FastAPIやDBドライバに依存してはならない。純粋なPythonクラスのみ。
    * **Entities**: ドメインモデル（例: `User`, `Order`）。
    * **Repository Interfaces**: データアクセスの抽象定義 (例: `IUserRepository`)。
    * **Domain Services**: 複数のEntityにまたがる純粋な計算ロジック。
2.  **UseCase (Middle)**: `src/usecase/`
    * アプリケーションのビジネスロジック。
    * **DTO (Data Transfer Object):** 入出力データ構造の定義には **Pydantic (`BaseModel`)** を使用し、型安全性とバリデーションを担保する。
    * **feature folder(`{feature name}/`)**:
    * **feature folder(`{feature name}/`)**: 
        * **Input Ports (`interface.py`)**: UseCaseのポート。
        * **Interactors (`interactor.py`)**: 機能固有のビジネスルール実装。
    * **ports(`ports/`)**: 機能横断(ユースケース全体)で使用する機能のポート(例: `ILogger`)。
3.  **Adapter (Outer)**: `src/adapter/`
    * 外部との通信の実装詳細。
    * **Entrypoints (`entrypoints/`)**: Driving Adapters (例: FastAPI Router)。
    * **Gateways (`gateways/`)**: Driven Adapters (例: MongoDB Repository 実装)。
    * **Infra**: ロガー等の実装 (`JsonLogger`)。


... (サンプルのFew-shotコード等)
```

### で、生成されたコードがこちらです

#### ディレクトリ構造

```text
.
└── app
    ├── adapter
    │   ├── entrypoints
    │   │   └── health_router.py          # ルーター (Controller)
    │   └── gateways
    │       └── mongo_health_repository.py # DB実装 (Interfaceの実装)
    ├── domain
    │   ├── model
    │   │   └── health_message.py         # ドメインモデル
    │   └── repository
    │       └── health_repository.py      # リポジトリインターフェース
    ├── usecase
    │   ├── dto
    │   │   └── health_dto.py             # 入出力データ構造
    │   └── health_check
    │       ├── interactor.py             # ビジネスロジック実装
    │       └── interface.py              # ユースケースインターフェース
    └── main.py
```

#### `/health`のコード（抜粋）

実際にはrepository実装やFastAPIのルーター実装等も同時に出力されてますが、ここではサンプルとしてinteractorを示します。

DIPが厳密に守られているのが見て取れます。

```python
# ./app/usecase/health_check/interactor.py
"""Health Check UseCase Implementation.

This module implements the health check use case logic.
"""

import uuid

from app.domain.constants import LOG_EVENTS
from app.domain.model.health_message import HealthMessage
from app.domain.repository.health_repository import IHealthRepository
from app.usecase.dto.health_dto import HealthOutput, SaveHealthInput
from app.usecase.health_check.interface import IHealthCheckUseCase
from app.usecase.ports.logger import ILogger


class HealthCheckInteractor(IHealthCheckUseCase):
    """Interactor for health check operations."""

    def __init__(self, health_repository: IHealthRepository, logger: ILogger) -> None:
        """Initialize the interactor.

        Args:
            health_repository: Repository for health message persistence
            logger: Logger for structured logging

        """
        self.health_repository = health_repository
        self.logger = logger

    async def save_message(self, input_data: SaveHealthInput) -> HealthOutput:
        """Save a health check message.

        Args:
            input_data: Input data containing the message

        Returns:
            The saved message details

        """
        self.logger.info(
            LOG_EVENTS.HEALTH_MESSAGE_SAVED,
            "Creating health message entity",
            context={"message": input_data.message},
        )

        # Create entity
        health_message = HealthMessage(
            id=str(uuid.uuid4()),
            message=input_data.message,
        )

        # Save via repository
        saved_message = await self.health_repository.save(health_message)

        self.logger.info(
            LOG_EVENTS.HEALTH_MESSAGE_SAVED,
            "Health message saved successfully",
            context={"message_id": saved_message.id},
        )

        # Map to output DTO
        return HealthOutput(
            id=saved_message.id,
            message=saved_message.message,
            created_at=saved_message.created_at.isoformat(),
        )

    async def get_latest_message(self) -> HealthOutput | None:
        """Get the latest health check message.

        Returns:
            The latest message, or None if not found
        """
        self.logger.info(
            LOG_EVENTS.HEALTH_MESSAGE_RETRIEVED,
            "Fetching latest health message",
        )

        message = await self.health_repository.find_latest()

        if message is None:
            self.logger.info(
                LOG_EVENTS.HEALTH_MESSAGE_RETRIEVED,
                "No health message found",
            )
            return None

        self.logger.info(
            LOG_EVENTS.HEALTH_MESSAGE_RETRIEVED,
            "Latest health message retrieved",
            context={"message_id": message.id},
        )

        return HealthOutput(
            id=message.id,
            message=message.message,
            created_at=message.created_at.isoformat(),
        )

```

Agentic AIに指示する際は、「課題チケットの内容を実装して」と指示するだけでOKです(チケットの中身はこのあと例示します)。このコードを出させるために、最初の指示に加えて一回だけ指摘を入れました。

**何かしらの設計パターンを適用しておくと、AIは責務ごとに実装していくことになるので、割とややこしい実装をさせても途中でわけわからなくなって迷走するようなことが少なくなります。**

#### 課題チケット

課題チケットの内容は下記のような感じです。ここはそんなに工夫してなくて、一般的な課題起票のやり方であれば何でもOKだと思います。
2~3年目のエンジニアに指示するようなつもりで課題背景から起票するのが大事です。

```markdown
## Overview

本タスクでは、ReactフロントエンドからFastAPIを経由してMongoDBにデータを保存し、その結果を画面に返すという「一気通貫の最小機能（Walking Skeleton）」を実装する。
これにより、CORS設定、APIルーティング、DB接続設定など、Webアプリケーションとして動作するために必要な全てのパイプラインが正常に接続されていることを証明する。

## User Story

- **As a** developer
- **I want to** ブラウザ上の入力フォームからテキストを送信し、それがデータベースに保存され、画面に成功メッセージが返ってくることを確認したい
- **so that** フロントエンドからDBまでの全レイヤー（Network, CORS, Auth, DB I/O）に問題がないことを保証し、安心して本番機能（US-05等）の実装に進めるようにするためだ。

## Background & Context

環境構築（Docker/uv）は完了しているが、各コンポーネントが連携して動作するかは未検証である。
特にSPA構成（React + FastAPI）では、**CORS（Cross-Origin Resource Sharing）エラー**や、Dockerコンテナとホスト間のネットワーク接続（WSL2特有の `localhost` 解決など）でハマるケースが多い。
個別にテストするのではなく、実際のデータフロー（UI -> API -> DB）を通すことで、これら全ての問題を一度に検証・解決する。


## 機能要件

- **簡易UI実装:** React側に「テキスト入力欄」と「送信ボタン」を持つ検証用ページ（`/test` 等）が作成されていること。
- **API実装:** `POST /api/health/echo` (仮) 等のエンドポイントを作成し、受け取ったデータをMongoDBへ保存する処理が実装されていること。
- **CORS設定:** フロントエンド（例: `http://localhost:5173`）からバックエンド（例: `http://localhost:8000`）へのリクエストが許可され、ブラウザコンソールにエラーが出ないこと。
- **データ永続化:** ブラウザから送信したデータが、Docker上のMongoDBに実際に保存されていること（CompassやCLIで確認可能であること）。
- **UIフィードバック:** 保存成功後、APIからのレスポンスを受け取り、画面上に「保存成功: [送信したテキスト]」などのメッセージが表示されること。
```
