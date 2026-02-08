# Claude Code「Agent Teams」について（用途・導入・セキュリティ・デメリット）

---

## 0. 「Agent Teams」って何？（1分でわかる説明）
Agent Teamsは、Claude Code上で 複数の作業セッション（teammates） を同時に動かし、タスクを分割して並列に進める仕組みです。


実際に商用完全OKのデータを入力データとしてみてOCR実装についてためしてみた様子です。
<img width="527" height="279" alt="スクリーンショット 2026-02-08 212428" src="https://github.com/user-attachments/assets/ecb9f8be-e320-489a-a542-77f697dd0255" />

各Agentの役割は以下の通り
```
・Ops（環境・依存チェック）
目的：ベンチが“迷いなく確実に走る状態”を作る。失敗したら最短復旧手順を提示する。

・Runner（E2E実行）
目的：決められた順で run.ps1 を回し、run_idとログを確実に残す。途中停止したら原因特定→再実行まで持っていく。

・Analyst（成果物検証・指標まとめ・gate確認）
目的：runが「正しく走った」ことを成果物で担保し、数値を読み取り、gate(人間が定めたpassの基準)が意図どおり判定されているか確認する。

・Improver（精度改善分析・実装案・次実験計画）
目的：Analystの観察を受けて、外部から情報をフェッチし、次に何を試せばよいかを具体的にする。

・Reporter（最終レポート整形：REPORT.md + summary.json）
目的：チームの結果を、後から見ても再現できる形に整える。読む人（未来のagents/人間）が意思決定できる状態にする。
```


向いている作業：
- 仕様の穴埋め、設計案の比較、APIのフロント/バック協調、リファクタや移行計画など「並列化できる知的作業」
- “役割” を分けて、同時に前に進めたい作業（例：設計/実装/テスト/レビュー）
---

## 1. Subagents と Agent Teams の違い（混同しやすいポイント）
### Agent Teams（Agentとの相互連携可能）
- 複数セッション を並列に動かし、相互に連携して進める
- 割り込み可能（各チームメイトに個別に自然言語で指示を送れる）
- 複数の役割が協調的に連携する複雑なタスクに強い。

### Subagents（連携はリーダのみと行う）
- “単一セッション内” で専門役に委譲する（セッション自体は1つ）

参考（公式）：
- Subagents: https://code.claude.com/docs/en/sub-agents
- Agent Teams: https://code.claude.com/docs/en/agent-teams

---

## 2. 何に使える？（OCR実装自動化〜上流マネジメント）
### 2.1 上流工程（ブレスト／要件定義／設計）
- 論点出し（例外、非機能、運用、監査、SLO）
- 要件の分解（確認事項リスト、前提、優先度、受け入れ条件）
- 設計案A/B/C（メリデメ、移行、リスク、ロールバック）

### 2.2 APIのフロント／バック協調
- OpenAPI / JSON Schema変更の影響分析
- エラーモデル、互換性方針、段階移行の設計
- フロント：型生成・モック・UI側の受け入れ条件整理
- バック：段階リリース、後方互換、観測点の設計

### 2.3 大規模改装（リファクタ／分割移行）
- 依存関係の棚卸し、移行順序、切り戻し条件
- テスト戦略（回帰の焦点、スモーク、観測設計）
- 変更面積を減らす切り方（段階導入、境界の再設計）

### 2.4 OCR実装のOps化
- 疑似データ（合成帳票）設計：ノイズ・崩れ・画質劣化の再現
- 抽出スキーマ／正規化ルール／評価指標の設計
- 結果レポートの自動出力、人間との協調サイクル

---

## 3. デメリット
### 3.1 コストが増えやすい（並列＝トークン増）
- teammatesを増やすほど、同時に会話・推論が走るためトークン消費が増えやすい
- “並列化した結果、出力レビューの工数も増える” ことがある（統合が必要）

### 3.2 統制が難しくなる（権限・監査・ルール）
- leadの権限がチームに波及する（設定をミスると全体が危険）
- どの作業をAIに任せ、どこから人間承認にするかの線引きが必要

### 3.3 実験機能の制約
- Agent Teamsは実験機能。挙動や仕様が変わる可能性がある
- “安定運用” を求めるほど、ガードレールと運用設計が必要

### 3.4 価格（高いと言われやすいポイント）
- Team / Enterpriseは基本的に席課金（seat）になりやすく、上位プランは高額になりやすい
Opus 4.6は


---

## 4. 導入方法（Claude Code + Agent Teams）
### 4.1 Claude Code の導入（概要）
- Claude Codeをインストールしてログインして使う
- 利用条件・対応プランは公式で確認（Team/Enterpriseの扱い含む）
  - https://claude.com/pricing

### 4.2 Agent Teams を有効化
Agent Teamsは環境変数で有効化できる方式が公式に記載されています。

例：ユーザー設定 `~/.claude/settings.json`
~~~json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  },
  "teammateMode": "auto"
}
~~~
詳しいことはLLMに聞いて設定するとよさそうです。
参考（公式）：
- Agent Teams: https://code.claude.com/docs/en/agent-teams
- Settings（teammateMode等）: https://code.claude.com/docs/en/settings

---

## 5. セキュリティ Best Practices
### 5.1 まず原則（最重要）
- PIIをClaude Codeに入力しない（プロンプト、ファイル内容、ツール出力に含めない）
- PIIが必要な作業は “閉域（社内）” で完結させ、Claude側には疑似データ or 集計値のみ渡す

### 5.2 公式おすすめの対策
- データ対策（入力を無害化）
  - 疑似データ（合成帳票）で開発・検証
  - 匿名化/マスク済みサンプル（復元不能）だけ共有
  - 実データは閉域で評価→集計値だけ共有
- 実行環境対策（壊れても漏れない箱）
  - sandboxing（ファイル/ネットワーク隔離）
  - ワークスペース分離（PIIを物理的に置かない）
- 権限対策（できない状態を作る）
  - managed-settings.json で組織強制（ユーザーが上書き不可）
  - `.env` / `secrets/**` / `pii/**` などを Read deny
  - 外向き通信につながる Bash を deny（curl/wget/scp等）
- 拡張対策（MCP/プラグインを締める）
  - managed-mcp.json で許可制（固定セット）
  - allowlist/denylistで接続先やサーバを制限
- プロセス対策（人間の承認ゲート）
  - Plan → 承認 → 実行 の運用（上流・大改装ほど必須）
  - 生成物のレビュー（セキュリティ/運用/品質）

---

## 6. 強制ポリシーの実装例（managed-settings.json）
目的：
- ユーザーが勝手に権限を緩めない
- PII/シークレットを読めない
- 外向き送信につながるBashを封じる
- WebFetchは「取得のみOK」として allowlist 運用

配置場所（Managed・組織強制）：
- macOS: /Library/Application Support/ClaudeCode/managed-settings.json
- Linux/WSL: /etc/claude-code/managed-settings.json
- Windows: C:\Program Files\ClaudeCode\managed-settings.json
参考：https://code.claude.com/docs/en/settings

例（あくまで叩き台なので、ルールに合わせて調整が必要）
~~~json
{
  "allowManagedPermissionRulesOnly": true,
  "allowManagedHooksOnly": true,
  "disableBypassPermissionsMode": "disable",

  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  },

  "permissions": {
    "deny": [
      "Read(./pii/**)",
      "Read(./data/**)",
      "Read(./secrets/**)",
      "Read(./.env)",
      "Read(./.env.*)",
      "Read(~/.ssh/**)",
      "Read(~/.aws/**)",

      "Bash(curl *)",
      "Bash(wget *)",
      "Bash(scp *)",
      "Bash(rsync *)",
      "Bash(nc *)",
      "Bash(socat *)",
      "Bash(ftp *)"
    ],
    "ask": [
      "Bash(git push *)",
      "Bash(gh pr create *)"
    ],
    "allow": [
      "WebFetch(domain:code.claude.com)",
      "WebFetch(domain:docs.anthropic.com)",
      "WebFetch(domain:github.com)",

      "Bash(git status)",
      "Bash(git diff *)",
      "Bash(pytest *)",
      "Bash(npm test *)",

      "Read",
      "Edit",
      "Write"
    ]
  }
}
~~~

参考：
- Permissions: https://code.claude.com/docs/en/permissions
- Settings: https://code.claude.com/docs/en/settings

---

## 7. sandboxingについて
狙い：
- Bash実行の影響範囲を閉じ込める
- ネットワーク到達先を絞る（WebFetchはOKでも “Bashで外へ投げる” を減らす）

~~~json
{
  "sandbox": {
    "enabled": true,
    "autoAllowBashIfSandboxed": true,
    "allowUnsandboxedCommands": false,
    "excludedCommands": ["git"],
    "network": {
      "allowedDomains": [
        "github.com",
        "code.claude.com",
        "docs.anthropic.com"
      ]
    }
  }
}
~~~

参考：
- Settings（sandboxing含む）: https://code.claude.com/docs/en/settings

---

## 8. MCP（拡張）の統制
### 8.1 managed-mcp.json
配置場所：
- macOS: /Library/Application Support/ClaudeCode/managed-mcp.json
- Linux/WSL: /etc/claude-code/managed-mcp.json
- Windows: C:\Program Files\ClaudeCode\managed-mcp.json

~~~json
{
  "mcpServers": {
    "company-internal": {
      "type": "stdio",
      "command": "/usr/local/bin/company-mcp-server",
      "args": ["--config", "/etc/company/mcp-config.json"]
    }
  }
}
~~~

参考：
- MCP: https://code.claude.com/docs/en/mcp
--

## 参考リンク
- Agent Teams: https://code.claude.com/docs/en/agent-teams
- Subagents: https://code.claude.com/docs/en/sub-agents
- Permissions: https://code.claude.com/docs/en/permissions
- Settings: https://code.claude.com/docs/en/settings
- MCP: https://code.claude.com/docs/en/mcp
- Hooks: https://code.claude.com/docs/en/hooks
- Pricing: https://claude.com/pricing
