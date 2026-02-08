# Claude Code「Agent Teams」社内向けまとめ（用途・導入・セキュリティ・デメリット）
前提：PII（個人情報）は外部に出た時点でアウト。WebFetchは「外部から取得するだけならOK」。

---

## 0. 「Agent Teams」って何？（1分でわかる説明）
Agent Teamsは、Claude Code上で “複数の作業セッション（teammates）” を同時に動かし、タスクを分割して並列に進める仕組みです。

イメージ：
- lead（リード）が「何をやるか」を整理して teammates に振る
- teammates が各自で調査・実装・レビュー等を進める
- lead が成果を統合して、最終成果物にまとめる

向いている作業：
- 仕様の穴埋め、設計案の比較、APIのフロント/バック協調、リファクタや移行計画など「並列化できる知的作業」
- “役割” を分けて、同時に前に進めたい作業（例：設計/実装/テスト/レビュー）

注意点（重要）：
- teammateは lead の権限設定（permissions）を引き継ぎます。leadが危険な設定だとチーム全体が危険になります。
- 並列化は速い反面、トークン消費が増えやすく、コストも上がりやすいです。

参考（公式）：
- Agent Teams: https://code.claude.com/docs/en/agent-teams

---

## 1. Subagents と Agent Teams の違い（混同しやすいポイント）
### Agent Teams
- “複数セッション” を並列に動かし、相互に連携して進める
- 目的：並列化、役割分担、統合（leadがまとめる）
- 向く：上流工程、クロスレイヤ協調、並列レビュー、改装計画など

### Subagents
- “単一セッション内” で専門役に委譲する（セッション自体は1つ）
- 目的：特定タスクの委譲、コンテキスト節約、役割テンプレ化
- 向く：単発の調査・要約・特定観点レビューをさっと投げたい時

参考（公式）：
- Subagents: https://code.claude.com/docs/en/sub-agents
- Agent Teams: https://code.claude.com/docs/en/agent-teams

---

## 2. 何に使える？（OCR実装自動化〜上流マネジメント）
### 2.1 OCR（PIIを扱わない前提で）
- 疑似データ（合成帳票）設計：ノイズ・崩れ・画質劣化の再現
- 抽出スキーマ／正規化ルール／評価指標の設計
- 回帰検知レポートのテンプレ化（失敗スライスTopN、改善/悪化の切り分け）
- 速度も含めたベンチ設計（精度×スループット×運用コスト）

### 2.2 上流工程（ブレスト／要件定義／設計）
- 論点出し（例外、非機能、運用、監査、SLO）
- 要件の分解（確認事項リスト、前提、優先度、受け入れ条件）
- 設計案A/B/C（メリデメ、移行、リスク、ロールバック）

### 2.3 APIのフロント／バック協調
- OpenAPI / JSON Schema変更の影響分析
- エラーモデル、互換性方針、段階移行の設計
- フロント：型生成・モック・UI側の受け入れ条件整理
- バック：段階リリース、後方互換、観測点の設計

### 2.4 大規模改装（リファクタ／分割移行）
- 依存関係の棚卸し、移行順序、切り戻し条件
- テスト戦略（回帰の焦点、スモーク、観測設計）
- 変更面積を減らす切り方（段階導入、境界の再設計）

---

## 3. デメリット（社内説明で押さえるべき点）
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
- 最新の価格は必ず公式で確認（社内資料はリンクで逃がすのが安全）
  - Pricing: https://claude.com/pricing

---

## 4. 導入方法（Claude Code + Agent Teams）
### 4.1 Claude Code の導入（概要）
- Claude Codeをインストールしてログインして使う
- 利用条件・対応プランは公式で確認（Team/Enterpriseの扱い含む）
  - https://claude.com/pricing

（ここは社内の端末標準、プロキシ、証明書、許可されたインストール方法に合わせて決める）

### 4.2 Agent Teams を有効化
Agent Teamsは環境変数で有効化する方式が公式に記載されています。

例：ユーザー設定 `~/.claude/settings.json`
~~~json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  },
  "teammateMode": "auto"
}
~~~

参考（公式）：
- Agent Teams: https://code.claude.com/docs/en/agent-teams
- Settings（teammateMode等）: https://code.claude.com/docs/en/settings

---

## 5. セキュリティ Best Practices（PIIは外部に出た時点でアウト）
### 5.1 まず原則（最重要）
- PIIをClaude Codeに入力しない（プロンプト、ファイル内容、ツール出力に含めない）
- PIIが必要な作業は “閉域（社内）” で完結させ、Claude側には疑似データ or 集計値のみ渡す

### 5.2 対策カテゴリ（sandboxing と 疑似データは別案）
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

## 6. コピペ用：強制ポリシー（managed-settings.json）
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

例（叩き台。社内ルールに合わせて調整）
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

## 7. コピペ用：sandboxing（任意だが強い）
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

## 8. コピペ用：MCP（拡張）を企業で統制する
### 8.1 managed-mcp.json（固定セットのみ）
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

---

## 9. コピペ用：Hooks（実行前ゲート／任意）
用途：
- “外へ出しそうなコマンド” を追加で止める
- うっかりを防ぐ最終防衛線（ただしこれだけに頼らない）

`.claude/hooks/block-exfil.sh`
~~~bash
#!/bin/bash
set -euo pipefail

TOOL=$(jq -r '.tool_name')
CMD=$(jq -r '.tool_input.command // empty')

if [ "$TOOL" = "Bash" ]; then
  if echo "$CMD" | grep -Eqi '\b(curl|wget|scp|rsync|nc|socat|ftp)\b'; then
    jq -n '{
      hookSpecificOutput: {
        hookEventName: "PreToolUse",
        permissionDecision: "deny",
        permissionDecisionReason: "Blocked: outbound network-like command (PII policy)."
      }
    }'
    exit 0
  fi
fi

exit 0
~~~

`.claude/settings.json`（プロジェクト設定に追加）
~~~json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": ".claude/hooks/block-exfil.sh"
          }
        ]
      }
    ]
  }
}
~~~

参考：
- Hooks: https://code.claude.com/docs/en/hooks

---

## 10. 会社紹介用：短い説明（そのまま貼れる）
- Agent Teamsは、Claude Codeで複数の作業セッションを並列に動かし、上流工程や改装、API協調などを役割分担で速く進める仕組み。
- Subagentsは単一セッション内の委譲で、軽いタスクや観点分離に向く。並列で相互連携させるならAgent Teams。
- PIIが外部に出た時点でアウトの会社では、Claude CodeにPIIを渡さず、疑似データ・匿名化・集計値だけで回すのが前提。
- 導入は「Managed設定で権限を強制」「MCPを許可制」「sandboxingとhooksで多重防御」「Plan承認とレビューゲート」の組み合わせが現実的。
- デメリットはコスト（並列＝トークン増）と統制（権限・監査・運用ルール）の難しさ。価格は公式を参照： https://claude.com/pricing

---

## 参考リンク（社内資料に貼る用）
- Agent Teams: https://code.claude.com/docs/en/agent-teams
- Subagents: https://code.claude.com/docs/en/sub-agents
- Permissions: https://code.claude.com/docs/en/permissions
- Settings: https://code.claude.com/docs/en/settings
- MCP: https://code.claude.com/docs/en/mcp
- Hooks: https://code.claude.com/docs/en/hooks
- Pricing: https://claude.com/pricing
