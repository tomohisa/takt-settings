# GitHub Copilot CLI Provider 設計書

## 概要

GitHub Copilot CLI (`@github/copilot`) を takt の新規プロバイダーとして登録する。
Cursor プロバイダーと同様の CLI ラッパーパターンで実装する。

## GitHub Copilot CLI の特徴

- **パッケージ**: `npm install -g @github/copilot`
- **コマンド**: `copilot`
- **Node.js 22+** が必要
- **認証**: GitHub トークン（`COPILOT_GITHUB_TOKEN` > `GH_TOKEN` > `GITHUB_TOKEN` > `gh` CLI > OAuth Device Flow）
- **全 Copilot プラン**で利用可能（Free, Pro, Pro+, Business, Enterprise）
- **GA**: 2026-02-25 に正式リリース

### 主要 CLI フラグ

| フラグ | 用途 |
|--------|------|
| `-p PROMPT` | 非インタラクティブ（ワンショット）実行 |
| `--model MODEL` | モデル選択 |
| `--resume SESSION_ID` | セッション再開 |
| `--autopilot` | 自律実行モード |
| `--yolo` / `--allow-all` | 全権限許可 |
| `--allow-all-tools` | 全ツール許可 |
| `--no-ask-user` | ユーザー確認無効化 |
| `--max-autopilot-continues N` | 自律実行の最大継続回数 |
| `-s` / `--silent` | エージェント応答のみ出力 |
| `--no-color` | カラー出力無効 |
| `--no-custom-instructions` | カスタム指示無効 |
| `--no-auto-update` | 自動更新無効 |
| `--agent AGENT` | カスタムエージェント指定 |
| `--additional-mcp-config JSON` | セッション用 MCP サーバー追加 |
| `--stream on/off` | ストリーミング制御 |
| `--share PATH` | セッションを Markdown にエクスポート |

### 制限事項

- **JSON 出力モード未対応**: `--output-format json` のような標準的な JSON 出力フラグがない（Issue #52 で要望中）
- **非インタラクティブモードで MCP サーバー未起動**: `-p` 使用時に MCP が動かない（Issue #633）
- **直接的な `--system-prompt` フラグなし**: カスタム指示は `AGENTS.md` / `.github/copilot-instructions.md` 経由

---

## 実装アプローチ

### A. 2 つの選択肢

#### 選択肢 1: CLI ラッパー（Cursor パターン準拠）— 推奨

Cursor プロバイダーと同様に `copilot` CLI を子プロセスとして起動する。

**メリット**:
- 既存パターンとの一貫性が高い
- SDK 依存なし（npm パッケージ追加不要）
- CLI のアップデートに自動追従

**デメリット**:
- JSON 出力がないため、stdout パースが不安定になり得る
- システムプロンプトの注入方法が限定的

#### 選択肢 2: SDK (`@github/copilot-sdk`) 経由

JSON-RPC ベースの SDK を使用してプログラマティックに制御する。

**メリット**:
- 構造化された JSON-RPC 通信
- セッション管理が堅牢
- ストリーミングサポート

**デメリット**:
- Technical Preview 段階で破壊的変更のリスク
- 追加の npm 依存
- ドキュメントが限定的

### 結論: 選択肢 1（CLI ラッパー）を推奨

SDK が Technical Preview であることを考慮し、安定した CLI ラッパーから始める。
SDK が安定したら移行を検討できる。

---

## 実装計画

### Phase 1: コア実装（必須ファイル）

#### 1.1 `src/infra/copilot/types.ts` — 型定義

```typescript
import type { PermissionMode } from '../../core/models/index.js';
import type { StreamCallback } from '../claude/index.js';

export interface CopilotCallOptions {
  cwd: string;
  abortSignal?: AbortSignal;
  sessionId?: string;
  model?: string;
  systemPrompt?: string;
  permissionMode?: PermissionMode;
  onStream?: StreamCallback;
  copilotGithubToken?: string;   // GitHub トークン
  copilotCliPath?: string;       // CLI パスオーバーライド
  maxAutopilotContinues?: number; // autopilot 最大継続回数
}
```

#### 1.2 `src/infra/copilot/client.ts` — CLI ラッパー

Cursor の `client.ts` をベースに以下を実装:

- **`execCopilot()`**: `copilot` CLI を `spawn()` で起動
- **引数構築 (`buildArgs`)**:
  ```
  copilot -p "{prompt}" --silent --no-color --no-auto-update --autopilot \
    --max-autopilot-continues 50 \
    [--model MODEL] \
    [--resume SESSION_ID] \
    [--yolo]  (permissionMode === 'full' の場合)
  ```
- **環境変数構築 (`buildEnv`)**:
  - `COPILOT_GITHUB_TOKEN` を注入
- **出力パース (`parseCopilotOutput`)**:
  - `--silent` + `--no-color` の stdout を内容として扱う
  - セッション ID の抽出（ログディレクトリや `--share` 出力から取得を検討）
- **エラー分類 (`classifyExecutionError`)**:
  - 認証エラー検出（`authentication`, `unauthorized`, `not logged in` パターン）
  - バイナリ未検出（ENOENT）
  - バッファオーバーフロー
- **Abort シグナル対応**: SIGTERM → SIGKILL フォールバック

**システムプロンプトの注入方法**:

`copilot` CLI には `--system-prompt` フラグがないため、以下の方法を検討:

1. **プロンプト先頭注入**（推奨・初期実装）:
   ```
   copilot -p "<<SYSTEM>>\n{systemPrompt}\n<</SYSTEM>>\n\n{userPrompt}"
   ```
   - Cursor と同じアプローチ（`buildPrompt()` でシステムプロンプトをプロンプト先頭に結合）

2. **一時 AGENTS.md ファイル**（将来の改善案）:
   - cwd に一時的な `AGENTS.md` を生成し、`--agent` フラグで指定
   - 終了後にクリーンアップ
   - 副作用が大きいため初期実装では避ける

#### 1.3 `src/infra/copilot/index.ts` — エクスポート

```typescript
export { CopilotClient, callCopilot, callCopilotCustom } from './client.js';
export type { CopilotCallOptions } from './types.js';
```

#### 1.4 `src/infra/providers/copilot.ts` — プロバイダー実装

```typescript
export class CopilotProvider implements Provider {
  setup(config: AgentSetup): ProviderAgent {
    // claudeAgent / claudeSkill は非対応 → throw
    // systemPrompt あり → callCopilotCustom
    // systemPrompt なし → callCopilot
  }
}
```

### Phase 2: 統合（既存ファイルの変更）

#### 2.1 `src/infra/providers/types.ts`

```diff
- export type ProviderType = 'claude' | 'codex' | 'opencode' | 'cursor' | 'mock';
+ export type ProviderType = 'claude' | 'codex' | 'opencode' | 'cursor' | 'copilot' | 'mock';
```

`ProviderCallOptions` にフィールド追加:
```diff
+ copilotGithubToken?: string;
```

#### 2.2 `src/infra/providers/index.ts`

```diff
+ import { CopilotProvider } from './copilot.js';

  this.providers = {
    claude: new ClaudeProvider(),
    codex: new CodexProvider(),
    opencode: new OpenCodeProvider(),
    cursor: new CursorProvider(),
+   copilot: new CopilotProvider(),
    mock: new MockProvider(),
  };
```

#### 2.3 `src/infra/config/global/globalConfig.ts`

- `PersistedGlobalConfig` に追加:
  - `copilotGithubToken?: string`
  - `copilotCliPath?: string`
- `resolveCopilotGithubToken()` 関数追加:
  - 優先順位: `TAKT_COPILOT_GITHUB_TOKEN` env > config.yaml > `COPILOT_GITHUB_TOKEN` env > `GH_TOKEN` env > `GITHUB_TOKEN` env > undefined
- `resolveCopilotCliPath()` 関数追加:
  - 優先順位: `TAKT_COPILOT_CLI_PATH` env > project config > global config > undefined（デフォルト `copilot`）
- `setProvider()` のユニオン型に `'copilot'` 追加
- `save()` / `load()` で `copilot_github_token` / `copilot_cli_path` を処理

#### 2.4 `src/core/models/schemas.ts` (GlobalConfigSchema)

Zod スキーマに追加:
```typescript
copilot_github_token: z.string().optional(),
copilot_cli_path: z.string().optional(),
```

#### 2.5 `src/core/models/persisted-global-config.ts`

```diff
+ copilotGithubToken?: string;
+ copilotCliPath?: string;
```

#### 2.6 `src/core/models/piece-types.ts`

`MovementProviderOptions` に Copilot オプション追加:
```typescript
copilot?: CopilotProviderOptions;

interface CopilotProviderOptions {
  maxAutopilotContinues?: number;  // デフォルト: 50
}
```

#### 2.7 `src/infra/config/env/config-env-overrides.ts`

環境変数マッピング追加:
- `TAKT_COPILOT_GITHUB_TOKEN`
- `TAKT_COPILOT_CLI_PATH`

#### 2.8 プロジェクト設定（`src/infra/config/project/`）

`ProjectConfig` に追加:
- `copilotCliPath?: string`

### Phase 3: パーミッションモードマッピング

| takt PermissionMode | Copilot CLI フラグ |
|---------------------|-------------------|
| `readonly` | （フラグなし、デフォルト動作） |
| `edit` | `--allow-all-tools --no-ask-user` |
| `full` | `--yolo` |

**注意**: `readonly` と `edit` の区別は Copilot CLI 側では完全にはサポートされていない。
`readonly` の場合は `--deny-tool 'write'` のようなツール制限を検討。

### Phase 4: セッション管理

Copilot CLI のセッション管理:
- `--resume SESSION_ID` でセッション再開
- セッション状態は `~/.copilot/session-state/` に保存
- **課題**: `--silent` モードでセッション ID の取得方法
  - `--share /tmp/session.md` でセッション情報を出力し、パースする案
  - ログディレクトリ (`~/.copilot/logs/`) から最新セッション ID を推定する案
  - 初期実装ではセッション再開を未サポートとし、各フェーズ独立実行とする

### Phase 5: テスト

#### 5.1 ユニットテスト

- `src/__tests__/copilot-provider.test.ts`
  - プロバイダーの setup/call テスト
  - claudeAgent/claudeSkill 非対応の検証
  - オプション変換テスト

- `src/__tests__/copilot-client.test.ts`
  - 引数構築テスト
  - 環境変数注入テスト
  - 出力パーステスト
  - エラー分類テスト
  - パーミッションモードマッピングテスト

#### 5.2 統合テスト

- プロバイダーレジストリからの取得テスト
- 設定解決テスト（環境変数 > config.yaml）

---

## 設定例

### ~/.takt/config.yaml

```yaml
provider: copilot
model: claude-sonnet-4.6

# Copilot 固有設定
copilot_github_token: ghp_xxxxx  # または TAKT_COPILOT_GITHUB_TOKEN 環境変数
copilot_cli_path: /usr/local/bin/copilot  # オプション

# プロバイダーオプション
provider_options:
  copilot:
    max_autopilot_continues: 50

# パーソナプロバイダーマッピング
persona_providers:
  coder: copilot
  reviewer: copilot

# パーミッションプロファイル
provider_profiles:
  copilot:
    default_permission_mode: edit
```

### Piece YAML での使用

```yaml
movements:
  - name: implement
    persona: coder
    provider: copilot
    model: claude-sonnet-4.6
    edit: true
    provider_options:
      copilot:
        max_autopilot_continues: 30
    instruction_template: |
      実装してください
    rules:
      - condition: "実装完了"
        next: review
```

---

## 変更対象ファイル一覧

### 新規作成

| ファイル | 内容 |
|---------|------|
| `src/infra/copilot/types.ts` | CopilotCallOptions 型定義 |
| `src/infra/copilot/client.ts` | CLI ラッパー実装 |
| `src/infra/copilot/index.ts` | エクスポート |
| `src/infra/providers/copilot.ts` | CopilotProvider クラス |
| `src/__tests__/copilot-provider.test.ts` | プロバイダーテスト |
| `src/__tests__/copilot-client.test.ts` | クライアントテスト |

### 変更

| ファイル | 変更内容 |
|---------|----------|
| `src/infra/providers/types.ts` | ProviderType に `'copilot'` 追加、ProviderCallOptions に `copilotGithubToken` 追加 |
| `src/infra/providers/index.ts` | CopilotProvider をレジストリに登録 |
| `src/infra/config/global/globalConfig.ts` | トークン/CLIパス解決関数、load/save 対応 |
| `src/core/models/schemas.ts` | GlobalConfigSchema に copilot フィールド追加 |
| `src/core/models/persisted-global-config.ts` | copilot 設定フィールド追加 |
| `src/core/models/piece-types.ts` | CopilotProviderOptions 型追加 |
| `src/infra/config/env/config-env-overrides.ts` | 環境変数マッピング追加 |
| `src/infra/config/project/projectConfig.ts` | copilotCliPath 追加 |

---

## 既知の課題と今後の検討事項

### 即座に対応が必要

1. **出力パース**: JSON 出力モードがないため、`--silent --no-color` の stdout テキストをそのまま content とする。構造化された応答パースは困難。
2. **セッション ID 取得**: `--silent` モードではセッション ID が stdout に含まれない可能性が高い。初期実装ではセッション再開を非サポートとする。

### 将来の改善

1. **SDK 移行**: `@github/copilot-sdk` が安定版になった時点で、JSON-RPC ベースの通信に移行し、セッション管理・ストリーミング・構造化出力を改善する。
2. **MCP サーバーサポート**: `-p` モードで MCP が動作しない制限（Issue #633）が解消されたら `--additional-mcp-config` を活用。
3. **AGENTS.md 活用**: システムプロンプトの注入を一時 AGENTS.md ファイル経由に改善。
4. **outputSchema サポート**: SDK 移行時に構造化出力を実現。
5. **ACP (Agent Client Protocol)**: `--acp` フラグでの JSON-RPC 通信を活用する可能性。

---

## 実装優先順位

1. **MVP（最小実行可能）**: CLI ラッパー + プロバイダー登録 + 基本的な設定解決
   - セッション再開なし
   - システムプロンプトはプロンプト先頭注入
   - `--silent --no-color` の stdout をそのまま AgentResponse.content に
2. **改善 1**: セッション ID 取得の仕組み（`--share` or ログ解析）
3. **改善 2**: パーミッションモードの精緻化（`--deny-tool` パターン）
4. **改善 3**: SDK 移行
