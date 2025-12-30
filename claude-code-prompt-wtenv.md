# wtenvを「並列開発のコントロールセンター」に進化させる

## プロジェクト概要

既存のwtenv（Rust製git worktree管理ツール）を拡張し、複数のClaude Codeセッションを並列で管理できる「並列開発のコントロールセンター」として機能させる。

**重要な制約:**
- ブラウザやGUIアプリは作成しない
- 全てターミナル（TUI）で完結
- 各worktreeで独立したClaude Codeセッションが動作
- 既存のwtenvコードベースを活用・拡張

## 実装する機能

### Phase 1: コア機能（優先度: 高）

#### 1. `wtenv status` - ワークツリー状態の可視化

**要件:**
```bash
wtenv status
```

**期待する出力:**
```
┌─────────────────────────────────────────────────────────────┐
│ Worktrees Overview (3 active)                               │
├─────────────────────────────────────────────────────────────┤
│ 🔄 feature-a                      main → feature-a          │
│    Status: Running                Process: pnpm test:e2e    │
│    Modified: 3 files              Last commit: 2h ago       │
│    Path: ../myapp-feature-a                                 │
│                                                              │
│ 🔨 feature-b                      main → feature-b          │
│    Status: Running                Process: pnpm build       │
│    Modified: 1 file               Last commit: 30m ago      │
│    Path: ../myapp-feature-b                                 │
│                                                              │
│ ✅ bugfix-123                     main → bugfix-123         │
│    Status: Clean                  No process                │
│    Ahead: 2 commits               Modified: 0 files         │
│    Path: ../myapp-bugfix-123      Last commit: 5m ago       │
│                                                              │
│ 📊 Total: 3 worktrees  |  Disk: 2.3 GB                     │
└─────────────────────────────────────────────────────────────┘
```

**実装詳細:**
- `git worktree list --porcelain`で全worktreeを取得
- 各worktreeで以下を取得:
  - ブランチ名: `git branch --show-current`
  - 変更ファイル数: `git status --porcelain | wc -l`
  - 最終コミット時刻: `git log -1 --format=%ar`
  - コミット差分: `git rev-list --count @{upstream}..HEAD`
- カラー出力（`colored`クレート使用）
- ボックス描画（`unicode-width`または手動で罫線文字使用）

#### 2. `wtenv ps` - プロセス管理

**要件:**
```bash
wtenv ps              # 実行中プロセス一覧
wtenv ps feature-a    # 特定worktreeのプロセス
wtenv kill feature-a  # プロセス停止
wtenv kill --all      # 全プロセス停止
```

**期待する出力:**
```
Active Processes in Worktrees:

feature-a (PID: 12345)
  Command: pnpm test:e2e
  Started: 9m 12s ago
  Working Dir: /home/user/projects/myapp-feature-a
  Status: Running

feature-b (PID: 12346)
  Command: pnpm build
  Started: 4m 01s ago
  Working Dir: /home/user/projects/myapp-feature-b
  Status: Running

Total: 2 processes
```

**実装詳細:**
- `.worktree/processes.json`にプロセス情報を保存
```json
{
  "processes": [
    {
      "worktree_path": "/path/to/worktree",
      "branch": "feature-a",
      "pid": 12345,
      "command": "pnpm test:e2e",
      "started_at": "2025-12-30T10:00:00Z",
      "cwd": "/path/to/worktree"
    }
  ]
}
```
- `postCreate`コマンド実行時にPIDを記録
- プロセスの生存確認: Unix系では`kill -0 <pid>`
- プロセス停止: `kill <pid>`
- 定期的なクリーンアップ（存在しないPIDを削除）

#### 3. `wtenv ui` - インタラクティブTUI

**要件:**
```bash
wtenv ui
```

**TUIの仕様:**
- **上下キー**: worktree選択
- **Enter**: 詳細表示
- **d**: worktree削除（確認付き）
- **o**: ディレクトリをエディタで開く
- **t**: ログ表示（該当する場合）
- **k**: プロセス停止
- **r**: 画面更新
- **q**: 終了

**実装詳細:**
- `ratatui` + `crossterm`クレートを使用
- レイアウト:
  ```
  ┌─ Worktree Manager ─────────────────────┐
  │ [↑↓] Navigate  [Enter] Details  [q] Quit│
  ├────────────────────────────────────────┤
  │  > 🔄 feature-a    Running test:e2e    │
  │    🔨 feature-b    Running build       │
  │    ✅ bugfix-123   Clean               │
  ├─ Details ──────────────────────────────┤
  │ Branch: main → feature-a               │
  │ Status: Modified (3 files)             │
  │ Process: pnpm test:e2e (PID: 12345)    │
  │ Duration: 9m 12s                       │
  └────────────────────────────────────────┘
  ```
- 状態管理:
  - 選択中のインデックス
  - worktreeリスト
  - プロセス情報
- 1秒ごとに自動更新（非同期）

### Phase 2: 高度な機能（優先度: 中）

#### 4. `wtenv diff-env` - 環境変数の差分管理

**要件:**
```bash
wtenv diff-env feature-a feature-b
wtenv diff-env --all  # 全worktreeの環境変数を比較
```

**期待する出力:**
```diff
Environment differences between feature-a and feature-b:

.env:
  API_ENDPOINT
-   feature-a: http://localhost:3000
+   feature-b: http://localhost:3001

  DATABASE_URL
-   feature-a: postgres://localhost/dev_a
+   feature-b: postgres://localhost/dev_b

.env.local:
  DEBUG_MODE
-   feature-a: true
+   feature-b: (not set)
```

**実装詳細:**
- `.env`, `.env.local`, `.env.development`などを検索
- `dotenv`クレートで環境変数をパース
- 差分をdiff形式で表示
- カラー出力（追加=緑、削除=赤、変更=黄）

#### 5. `wtenv analyze` - リソース分析

**要件:**
```bash
wtenv analyze
wtenv analyze --disk  # ディスク使用量のみ
```

**期待する出力:**
```
📊 Resource Analysis

Disk Usage by Worktree:
  feature-a:    850 MB  (node_modules: 450 MB)
  feature-b:    820 MB  (node_modules: 450 MB)
  bugfix-123:   800 MB  (node_modules: 450 MB)
  Total:        2.47 GB

💡 Optimization Suggestions:
  1. Shared node_modules can save ~900 MB
  2. Old worktrees detected:
     - old-feature (30 days old, merged)
```

**実装詳細:**
- `walkdir`クレートで再帰的にファイルサイズを集計
- 特定ディレクトリ（`node_modules`, `target`, `.next`など）を個別集計
- 最終コミット日時で古いworktreeを検出
- マージ済みブランチの検出: `git branch --merged`

#### 6. `wtenv notify` - 通知機能

**要件:**
```yaml
# .worktree.yml
notifications:
  on_complete: true
  on_error: true
  methods:
    - desktop
    - sound
```

**実装詳細:**
- `notify-rust`クレートでデスクトップ通知
- `postCreate`コマンドの実行結果を監視
- 完了・エラー時に通知
- サウンド: `\x07`（BEL文字）をターミナルに出力

### Phase 3: 追加機能（優先度: 低）

#### 7. `wtenv history` - 履歴とタイムトラッキング

**要件:**
```bash
wtenv history
wtenv history feature-a
```

**実装詳細:**
- `.worktree/history.json`に以下を記録:
  - worktree作成日時
  - コミット数
  - 最終アクティビティ
- `git log --since="7 days ago" --oneline | wc -l`でコミット数
- 統計情報の表示

#### 8. `wtenv clean` - スマートクリーンアップ

**要件:**
```bash
wtenv clean --interactive
wtenv clean --merged       # マージ済みのみ
wtenv clean --older-than 30d
```

**実装詳細:**
- マージ済みブランチ: `git branch --merged main`
- 最終コミット日時で古いものを検出
- 対話的UI（`dialoguer`クレート）で確認
- `git worktree remove`で削除

## アーキテクチャ設計

### ディレクトリ構造

```
wtenv/
├── src/
│   ├── main.rs              # エントリポイント
│   ├── cli.rs               # CLI定義（clap）
│   ├── commands/
│   │   ├── mod.rs
│   │   ├── create.rs        # 既存
│   │   ├── list.rs          # 既存
│   │   ├── remove.rs        # 既存
│   │   ├── status.rs        # 新規: status表示
│   │   ├── ps.rs            # 新規: プロセス管理
│   │   ├── ui.rs            # 新規: TUI
│   │   ├── diff_env.rs      # 新規: 環境変数diff
│   │   ├── analyze.rs       # 新規: リソース分析
│   │   └── clean.rs         # 新規: クリーンアップ
│   ├── worktree/
│   │   ├── mod.rs
│   │   ├── manager.rs       # worktree管理
│   │   ├── info.rs          # worktree情報取得
│   │   └── process.rs       # プロセス管理
│   ├── ui/
│   │   ├── mod.rs
│   │   ├── dashboard.rs     # TUIダッシュボード
│   │   └── components.rs    # 再利用可能なUIコンポーネント
│   ├── utils/
│   │   ├── mod.rs
│   │   ├── git.rs           # Git操作ユーティリティ
│   │   ├── display.rs       # 表示ユーティリティ
│   │   └── notify.rs        # 通知機能
│   └── config.rs            # 設定ファイル管理
├── Cargo.toml
└── .worktree/               # 状態管理ディレクトリ（.gitignore）
    ├── processes.json       # 実行中プロセス情報
    └── history.json         # 履歴データ
```

### 必要なクレート

```toml
[dependencies]
# 既存
clap = { version = "4.5", features = ["derive", "cargo"] }
serde = { version = "1.0", features = ["derive"] }
serde_yaml = "0.9"
colored = "2.1"
indicatif = "0.17"

# 新規追加
ratatui = "0.29"           # TUI
crossterm = "0.28"         # ターミナル制御
tokio = { version = "1", features = ["full"] }  # 非同期処理
walkdir = "2.5"            # ディレクトリ走査
notify-rust = "4.11"       # デスクトップ通知
dialoguer = "0.11"         # 対話的プロンプト
dotenv = "0.15"            # .envパース
chrono = "0.4"             # 日時処理
sysinfo = "0.32"           # プロセス情報
anyhow = "1.0"             # エラーハンドリング
```

## 実装の優先順位

### フェーズ1（1-2週間）
1. `wtenv status` - 状態可視化
2. `wtenv ps` - プロセス管理の基礎
3. プロセス情報の永続化（`processes.json`）

### フェーズ2（2-3週間）
4. `wtenv ui` - TUI実装
5. `wtenv diff-env` - 環境変数diff

### フェーズ3（1-2週間）
6. `wtenv analyze` - リソース分析
7. `wtenv notify` - 通知機能

### フェーズ4（オプショナル）
8. `wtenv history` - 履歴
9. `wtenv clean` - クリーンアップ

## Claude Codeセッション統合

### 既存の`postCreate`を拡張

```yaml
# .worktree.yml
version: "1.0"
copy:
  - .env
  - .env.local

postCreate:
  - command: pnpm install
    description: "依存関係をインストール中..."
    optional: false
    track: true  # 新規: プロセス追跡を有効化

  - command: pnpm test:e2e
    description: "E2Eテスト実行中..."
    optional: true
    track: true
    background: true  # 新規: バックグラウンド実行

# 新規: Claude Code統合
claude_code:
  auto_start: false  # worktree作成時に自動起動
  sessions:
    - name: "main"
      command: "code ."
    - name: "terminal"
      command: "tmux new-session -s {branch}"
```

### プロセス追跡の実装

`postCreate`でコマンド実行時:
1. `track: true`の場合、プロセスを`processes.json`に記録
2. `background: true`の場合、バックグラウンドで実行
3. プロセス完了時、通知を送信（`notifications.on_complete: true`の場合）

```rust
// src/worktree/process.rs

use std::process::{Command, Child};
use serde::{Serialize, Deserialize};

#[derive(Serialize, Deserialize)]
pub struct ProcessInfo {
    pub worktree_path: String,
    pub branch: String,
    pub pid: u32,
    pub command: String,
    pub started_at: chrono::DateTime<chrono::Utc>,
    pub cwd: String,
}

impl ProcessInfo {
    pub fn is_running(&self) -> bool {
        // Unix系でプロセス存在確認
        #[cfg(unix)]
        {
            use std::process::Command;
            Command::new("kill")
                .args(["-0", &self.pid.to_string()])
                .output()
                .map(|o| o.status.success())
                .unwrap_or(false)
        }

        // Windows系
        #[cfg(windows)]
        {
            use sysinfo::{System, Pid};
            let mut sys = System::new_all();
            sys.refresh_processes();
            sys.process(Pid::from_u32(self.pid)).is_some()
        }
    }
}
```

## テスト戦略

### ユニットテスト
- Git操作のモック
- プロセス管理ロジック
- 環境変数パース

### 統合テスト
- 実際のgit repositoryで動作確認
- worktree作成→状態確認→削除の一連の流れ

### 手動テスト
- TUIの操作性
- 通知機能
- 並列セッションの動作

## 開発の進め方

1. **既存コードの理解**
   - wtenvの現在の実装を確認
   - 拡張ポイントの特定

2. **Phase 1の実装**
   - `wtenv status`から開始（最も簡単）
   - `wtenv ps`でプロセス管理の基礎を構築

3. **TUIの実装**
   - `ratatui`の基本的な使い方を学習
   - シンプルなダッシュボードから開始

4. **反復的な改善**
   - 各機能を小さく実装
   - ユーザビリティテスト
   - フィードバックを反映

## 成功の定義

### 最小限の成功（MVP）
- [ ] `wtenv status`で全worktreeの状態を一覧表示
- [ ] `wtenv ps`で実行中プロセスを表示
- [ ] `wtenv ui`で基本的なTUIが動作

### 完全な成功
- [ ] 上記 + 全Phase 1-3機能の実装
- [ ] 複数のClaude Codeセッションを並列管理可能
- [ ] 安定した動作（クラッシュなし）
- [ ] 良好なUX（直感的な操作）

## 参考資料

- [ratatui公式ドキュメント](https://ratatui.rs/)
- [Rust非同期プログラミング](https://rust-lang.github.io/async-book/)
- [git-worktree公式ドキュメント](https://git-scm.com/docs/git-worktree)

---

## 最初のステップ

1. wtenvの既存リポジトリをクローン
2. `Cargo.toml`に新しい依存関係を追加
3. `src/commands/status.rs`を作成し、`wtenv status`を実装
4. 動作確認後、次の機能に進む

**質問があれば遠慮なく聞いてください！**
