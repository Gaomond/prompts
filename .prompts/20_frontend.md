# Frontend Architecture & Implementation Guidelines

F/Eのタスクでは本ファイルを参照してください。

## 1\. Purpose & Scope

このドキュメントは、本プロジェクトのフロントエンド開発における **「不変の憲法」** です。
AIエージェントはこのガイドラインに準拠し、Reactの柔軟性が招く「無秩序」を徹底的に排除しなければなりません。
本ドキュメントは「構造と責務」を定義します。具体的な色やサイズ等の「意匠」については、別途定義されるデザインガイドライン（Design Tokens等）に従ってください。

## 2\. Tech Stack (Strict)

以下の技術スタック以外の使用を禁止します。

* **Framework:** Vite
* **Language:** TypeScript (Strict Mode)
* **Styling:** Tailwind CSS (Utility-first) + CSS Modules (for complex animations)
* **State Management:** React Context + Hooks (Global/Client State), SWR (Server State)
* **Visual Effects:** Framer Motion or Canvas API (for high-load visual effects)
* **HTTP Client:** Fetch API (wrapped in Adapter)

## 3\. Architecture Pattern: Clean Architecture + Atomic Design

**関心の分離**を徹底します。
「見た目 (UI)」と「ロジック (UseCase)」と「データ定義 (Domain)」を明確に分割します。

### A. Layers & Dependency (Clean Architecture)

依存の方向は **UI (Components) -\> UseCase (Hooks) -\> Domain (Types)** です。
円の中心に向かってのみ依存します。外側の変更が内側に影響を与えてはいけません。

1. **Domain (Inner)**: `src/domain/`
      * アプリケーションのコアデータ型定義と、リポジトリのインターフェース定義。
      * UIライブラリやAPIクライアントに依存してはならない。純粋なTypeScriptの型のみ。
      * **Model**: `type User = { ... }`
      * **Repository Interface**: `interface IUserRepository { ... }` (DDDパターン: データアクセスの抽象定義もドメインの一部とする)
2. **UseCase (Middle)**: `src/usecase/`
      * アプリケーションのビジネスロジック。React Custom Hooksとして実装する。
      * UI操作（DOM操作）を含まない、純粋な「振る舞い」の定義。
3. **Adapter (Outer)**: `src/adapter/`
      * 外部との通信の実装詳細。
      * **Repository Impl**: Domain層で定義された `IUserRepository` などの実装。`fetch` はここに閉じ込める。
4. **UI (Presentation)**: `src/components/`
      * Atomic Designに基づくコンポーネント群。
      * **ロジックを持たない**: UIコンポーネント内で直接 `fetch` したり、複雑な計算をしてはならない。

### B. Dependency Inversion Principle (DIP)

上位モジュール（UseCase）が下位モジュール（Adapter/Repositoryの詳細）に依存することを禁止します。
双方が「抽象（Interface）」に依存しなければなりません。

* **Bad:** UseCaseが `ItemRepository.ts` (具象) を直接importする。
* **Good:** UseCaseは `IItemRepository` (抽象) を定義し、それに依存する。Adapter層がそのインターフェースを実装（Implement）し、実行時に注入される。

これにより、APIの実装やDBが変更されても、UseCase層のコードは変更する必要がなくなります。

### C. Atomic Design (Component Structure)

UIコンポーネントは以下の粒度で分割・管理します。

* **Atoms**: 最小単位。HTMLタグに近い。
  * *Example:* Button, Icon, Label, Badge
* **Molecules**: Atomの組み合わせ。UIとしての意味を持ち始める。
  * *Example:* SearchInput, UserListItem, ActionButton
* **Organisms**: 独立して機能するUIパーツの集合体。
  * *Example:* Header, GlobalNavigation, UserDetailCard
* **Templates**: ページレイアウトの骨組み。具体的な色・形等の情報は持たず、コンポーネント間の配置関係のみを管理する。プレースホルダーとして機能する。
  * *Example:* MainLayout, GridLayout, DashboardLayout
* **Pages**: ルーティングのエンドポイント。Templatesに具体的なデータを流し込む唯一の場所。

### D. Cross-Cutting Concerns (Dependency Injection)

ロギングや認証などの「横断的関心事」についても、依存の方向を守ります。

* **UseCase層:** 具体的なロガー実装（`SentryLogger`など）や、UI固有の仕組み（`React Context`）を知ってはいけません。必要な機能は **インターフェースとして定義し、引数で受け取ります（Method Injection）。**
* **Page層 (Smart):** DIコンテナとしての役割を持ちます。`useLogger()` などのHookを使って実体を取り出し、UseCase実行時に注入します。

## 4\. Implementation Principles

### A. The "Smart vs Dumb" Rule (Strict)

**ここが最も重要なルールです。**
コンポーネントの責務境界を以下のように厳格に定めます。例外は一切認めません。

* **Smart Components (Pages Only):**

  * **役割:** アプリケーションの「司令塔」。
  * **権限:** UseCase (Hooks) を呼び出し、データを取得・操作する。Global Stateにアクセスする。
  * **責務:** 取得したデータや、イベントハンドラ関数を `props` として下層の Dumb Components に渡す。
  * **禁止事項:** DOM構造（`div` や `span`）やスタイリングを極力持たないこと（Templatesに任せる）。

* **Dumb Components (Atoms / Molecules / Organisms / Templates):**

  * **役割:** 純粋な「見た目」のレンダリング。
  * **権限:** **一切のビジネスロジックを持たない。UseCaseをimportしてはならない。**
  * **責務:** データは全て `props` で受け取る。自身の状態は、UI固有の閉じた状態（アコーディオンの開閉、ホバー状態など）のみ許容される。
  * **禁止事項:** APIコール、Global Stateへのアクセス。

**要約:**
「`src/usecase/` 以下のHooksをimportできるのは、`src/components/pages/` 以下のファイルのみである。」

### B. Visual & Performance

高負荷なエフェクトや複雑なアニメーションについては、パフォーマンスへの影響を考慮し以下のルールを設けます。

* **Isolation:** 再レンダリングコストの高い描画ロジックは、独立したコンポーネントに切り出し、Reactの再レンダリングサイクルの影響を最小限に抑えること（`React.memo` や `useRef` の活用）。
* **Animation Library:** 単純なトランジションはCSS/Tailwindで完結させる。物理演算や複雑なシーケンスが必要な場合は Framer Motion または Canvas API を使用する。

### C. Domain Richness

コードは仕様書です。UIの「見た目」や「操作」ではなく、「ドメインの意図」を名前で表現してください。

* **Ubiquitous Language:** 変数名・ファイル名・クラス名は、必ずプロジェクトで定義されたユビキタス言語に従うこと。
* **Action Naming:** `handleClick()` や `update()` ではなく -\> **`submitOrder()`**, **`verifyUser()`** (具体的なビジネスアクション名)
* **Component Naming:** `RedButton` や `Wrapper` ではなく -\> **`PurchaseButton`**, **`ProductContainer`** (ドメイン上の役割名)

### D. SWR & Data Fetching Principles

SWRを使用する際も、データ取得の実装詳細（エンドポイントURL等）は必ず **Repository** に隠蔽しなければなりません。

* **Bad:** `useSWR('/api/user', (url) => fetch(url).then(r => r.json()))`
* **Good:** `useSWR(KEYS.USER, () => userRepository.fetchUser())`

### E. Log Event Standards

ログメッセージ（自然言語）は変更される可能性があるため、テストや監視には使用しません。
必ず **「イベントID（定数）」** を定義し、構造化ログとして記録・検証します。

* **Bad:** `logger.info("Fetching data...")` -\> テストで "Fetching" という文字列を監視する。
* **Good:** `logger.info("Fetching...", { event: LOG_EVENTS.FETCH_START })` -\> テストで `FETCH_START` IDを監視する。

## 5\. Directory Structure (The Map)

下記は、アーキテクチャを実現するためのフォルダ構造です。

```text
src/
├── domain/                 # Types & Pure Logic
│   ├── model/
│   └── repository/         # Repository Interfaces (DIP)
├── usecase/                # Application Logic (Hooks)
│   └── ports/              # Interfaces defined by UseCase           
├── adapter/                # API & Infra
│   ├── infra/              # Infra Implementations(e.g. SentryLogger)
│   └── repository/         # Repository Implementations
├── components/             # Atomic Design
│   ├── atoms/              # [Dumb] Button, Icon
│   ├── molecules/          # [Dumb] Cards, ListItems
│   ├── organisms/          # [Dumb] Sections, Complex Widgets 
│   ├── templates/          # [Dumb] Layouts
│   └── pages/              # [Smart] Routing Endpoints & Container
├── App.tsx                 # Routing Definition & Global Providers
└── main.tsx                # Entrypoint
```

## 6\. Testing Strategy

各レイヤーの責務分離に基づき、テストの「対象（Unit）」と「モック（Boundary）」を明確に定義します。
テストフレームワークは **Vitest + React Testing Library** を使用します。

### A. Level 1: Domain Unit Testing

* **Target:** `src/domain/`, `src/utils/`
* **Type:** Unit Test (Pure Logic)
* **Strategy:** 純粋な関数・型ロジックの検証。Reactや外部モジュールへの依存がないため、モックは原則不要。

### B. Level 2: Component Unit Testing (Dumb)

* **Target:** `src/components/{atoms,molecules,organisms,templates}`
* **Type:** Unit Test (UI Rendering)
* **Strategy:** HooksやContextを知らないため、**APIモック等は一切不要**。
* **Method:** `props` にダミーデータとスパイ関数(`vi.fn()`)を渡し、正しく表示・発火されるかを検証する。

```ts
// Example: ActionButton.test.tsx
// Dumb Componentは、Hooksを一切使わないので、そのままrenderするだけでテスト可能
test('triggers onClick when clicked', () => {
  const handleClick = vi.fn();
  render(<ActionButton label="Submit" onClick={handleClick} isLoading={false} />);
  
  fireEvent.click(screen.getByRole('button'));
  
  expect(handleClick).toHaveBeenCalled();
});
```

### C. Level 3: UseCase Unit Testing (Hooks)

* **Target:** `src/usecase/`
* **Type:** Unit Test (Business Logic)
* **Strategy:** **Repository（Adapter層）をモック**し、ビジネスロジックの正当性のみを検証する。
* **Mocking:** `src/adapter/repository/` 以下のモジュールを `vi.mock` で置き換える。

<!-- end list -->

```ts
// Example: useUserAction.test.ts
import { renderHook, act } from '@testing-library/react';
import { useUserAction } from './useUserAction';
// テスト対象が依存しているRepositoryの実装をimport
import { userRepository } from '@/adapter/repository/userRepository';

// 1. Repositoryモジュール全体をモック化
vi.mock('@/adapter/repository/userRepository', () => ({
  userRepository: {
    update: vi.fn(), // メソッドをスタブ化
  },
}));

test('calls repository successfully', async () => {
  const { result } = renderHook(() => useUserAction());
  
  await act(async () => {
    await result.current.updateUser({ id: '1', name: 'Alice' });
  });

  // UseCaseが正しくRepositoryを呼んだか検証
  expect(userRepository.update).toHaveBeenCalledWith({ id: '1', name: 'Alice' });
});
```

### D. Level 4: Page Unit Testing (Smart)

* **Target:** `src/components/pages/`
* **Type:** Unit Test (Container Logic)
* **Strategy:** **UseCase（Hooks）をモック**する。これにより、バックエンドや通信ロジックを完全に切り離し、Pageコンポーネントの「繋ぎこみ（Wiring）」の責務だけをテストする。
* **Mocking:** `src/usecase/` 以下のHooksを `vi.mock` で置き換える。

<!-- end list -->

```ts
// Example: UserProfilePage.test.tsx
import { UserProfilePage } from './UserProfilePage';
// モック対象のHookをimport
import * as UserHooks from '@/usecase/useUserAction';

test('passes loading state to children', () => {
  // 1. UseCase Hookをモック化し、任意のUI状態を再現する
  vi.spyOn(UserHooks, 'useUserAction').mockReturnValue({
    updateUser: vi.fn(),
    isLoading: true, // "ローディング中" という状態を強制的に作る
  });

  render(<UserProfilePage />);
  
  // ロジックがどうあれ、isLoading=trueならボタンが無効化されているはずだ、というUIの責務を検証
  expect(screen.getByRole('button')).toBeDisabled();
});
```

## 7\. Reference Implementation (Few-shot Example)

以下は具体的な実装例です。あくまで「構造の適用例」として参照してください。
ここでは **「Health Check (Server Echo)」** 機能を実装する場合の具体例を示します。

### [Layer 0] Common Utils (DRY & Error Handling)

API通信のボイラープレート（ヘッダー付与、エラーハンドリング）は、個別のRepositoryに書かず、共通のAdapterに集約します。

```ts
// src/domain/AppError.ts
// TODO: アプリケーション全体で使用する共通エラークラスを定義

// src/adapter/apiClient.ts
// fetchをラップし、共通のヘッダー処理とAppErrorへの変換を行う
import { AppError } from "@/domain/error/AppError";

const BASE_URL = import.meta.env.VITE_API_BASE_URL ?? "http://localhost:8000";

export const apiClient = async <T>(endpoint: string, options?: RequestInit): Promise<T> => {
  try {
    const res = await fetch(`${BASE_URL}${endpoint}`, {
      headers: { "Content-Type": "application/json", ...options?.headers },
      ...options,
    });

    if (!res.ok) {
      // ステータスコードに応じたエラーハンドリング(簡易実装)
      throw new AppError(res.status as any, `API Error: ${res.statusText}`);
    }

    // 204 No Contentの場合はnullを返す等の処理もここに記述
    if (res.status === 204) return null as T;

    return res.json();
  } catch (error) {
    if (error instanceof AppError) throw error;
    throw new AppError(500, "Network Error", error as Error);
  }
};
```

### [Layer 1] Domain & Adapter (DIP Applied)

```ts
// src/domain/model/health.ts
export interface HealthMessage {
  id: string;
  message: string;
  createdAt: string;
}

// src/domain/repository/healthRepository.ts (Interface)
export interface IHealthRepository {
  getLatest(): Promise<HealthMessage | null>;
  saveMessage(message: string): Promise<HealthMessage>;
}

// src/adapter/repository/healthRepositoryImpl.ts (Implementation)
// 共通化した apiClient を使用することで実装をシンプルに保つ
import { apiClient } from "@/adapter/apiClient";
import { IHealthRepository } from "@/domain/repository/healthRepository";
import { HealthMessage } from "@/domain/model/health";

export class HealthRepositoryImpl implements IHealthRepository {
  async getLatest(): Promise<HealthMessage | null> {
    return apiClient<HealthMessage | null>("/api/health/latest");
  }

  async saveMessage(message: string): Promise<HealthMessage> {
    return apiClient<HealthMessage>("/api/health/echo", {
      method: "POST",
      body: JSON.stringify({ message }),
    });
  }
}

// DI用のインスタンス
export const healthRepository = new HealthRepositoryImpl();
```

### [Layer 2] UseCase (Custom Hook)

SWR/Mutationを利用し、サーバー状態を管理します。

```ts
// src/usecase/useHealthCheck.ts
import useSWR from "swr";
import useSWRMutation from "swr/mutation";
import { IHealthRepository } from "@/domain/repository/healthRepository";

const KEYS = { HEALTH_LATEST: "/health/latest" };

// Fetch Hook
export const useFetchLatestHealthMessage = (repository: IHealthRepository) => {
  const { data, error, isLoading, mutate } = useSWR(
    KEYS.HEALTH_LATEST,
    () => repository.getLatest()
  );

  return {
    message: data ?? null,
    isLoading,
    error,      // AppErrorがここに伝播される
    refetch: mutate,
  };
};

// Mutation Hook
export const useSaveHealthMessage = (repository: IHealthRepository) => {
  const { trigger, isMutating, error } = useSWRMutation(
    KEYS.HEALTH_LATEST,
    async (_key, { arg }: { arg: string }) => repository.saveMessage(arg),
    { populateCache: true, revalidate: false } // 楽観的更新等の設定
  );

  return {
    saveMessage: trigger,
    isSaving: isMutating,
    error,
  };
};
```

### [Layer 3] UI (Atomic Design)

**Atoms / Molecules / Organisms (Dumb)**
ロジックを持たず、表示とイベント発火に専念します。

```tsx
// src/components/organisms/HealthCheckMonitor.tsx (Dumb)
import { HealthMessage } from "@/domain/model/health";
import { Button } from "@/components/atoms/Button";

interface Props {
  inputValue: string;
  onInputChange: (val: string) => void;
  onSave: () => void;
  onFetch: () => void;
  message: HealthMessage | null;
  isLoading: boolean;
  error?: Error;
}

export const HealthCheckMonitor = ({
  inputValue,
  onInputChange,
  onSave,
  onFetch,
  message,
  isLoading,
  error,
}: Props) => {
  return (
    <div className="p-4 border rounded">
      {/* 状態表示 */}
      <div className="mb-4 min-h-[60px]">
        {isLoading ? (
          <span className="text-gray-500">Loading...</span>
        ) : error ? (
          <span className="text-red-500">Error: {error.message}</span>
        ) : (
          <p className="text-xl font-bold">{message?.message ?? "No Data"}</p>
        )}
      </div>

      {/* 入力フォーム */}
      <div className="flex gap-2">
        <input 
          value={inputValue}
          onChange={(e) => onInputChange(e.target.value)}
          className="border p-2 rounded"
          disabled={isLoading}
        />
        <Button onClick={onSave} disabled={isLoading || !inputValue}>
          Echo
        </Button>
        <Button onClick={onFetch} variant="secondary">
          Reload
        </Button>
      </div>
    </div>
  );
};
```

**Pages (Smart)**
唯一、RepositoryやUseCaseを知っている場所です。

```tsx
// src/components/pages/HealthCheckPage.tsx (Smart)
import { useState } from "react";
import { healthRepository } from "@/adapter/repository/healthRepositoryImpl";
import { 
  useFetchLatestHealthMessage, 
  useSaveHealthMessage 
} from "@/usecase/useHealthCheck";
import { HealthCheckMonitor } from "@/components/organisms/HealthCheckMonitor";
import { HealthCheckTemplate } from "@/components/templates/HealthCheckTemplate";

export const HealthCheckPage = () => {
  // 1. Local State (UI State)
  const [inputText, setInputText] = useState("");

  // 2. UseCases (Hooks) - Repository Dependency Injection
  const { message, isLoading: isFetching, error: fetchError, refetch } = 
    useFetchLatestHealthMessage(healthRepository);
    
  const { saveMessage, isSaving, error: saveError } = 
    useSaveHealthMessage(healthRepository);

  // 3. Event Handlers (Wiring Logic)
  const handleSave = async () => {
    if (!inputText) return;
    await saveMessage(inputText);
    setInputText(""); // Clear input on success
  };

  // 4. Render
  const appError = saveError ?? fetchError;
  const isLoading = isFetching || isSaving;

  return (
    <HealthCheckTemplate>
      <HealthCheckMonitor
        inputValue={inputText}
        onInputChange={setInputText}
        onSave={handleSave}
        onFetch={() => refetch()}
        message={message}
        isLoading={isLoading}
        error={appError}
      />
    </HealthCheckTemplate>
  );
};
```
