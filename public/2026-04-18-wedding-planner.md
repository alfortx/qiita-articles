---
title: Claude CodeとMCPだけで結婚式準備を管理する自然言語タスク管理システムを作った
tags:
  - AI
  - MCP
  - taskmanagement
  - Claude
  - ClaudeCode
private: false
updated_at: '2026-07-12T20:08:32+09:00'
id: 4316e95f1f8838a06241
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

## TL;DR

- **コード不要**でClaude CodeとMCP（Model Context Protocol）だけで結婚式準備のタスク管理システムを構築
- **自然言語で質問**するだけで、iOSメモ・Gmail・iCloudの情報を自動収集
- **Skill機能**により「花屋さんの件どうなった?」と聞くだけでメール・メモを自動探索
- Yahoo自動転送に対応した柔軟な検索設計で、将来の仕様変更にも強い

## 環境

- macOS (Apple Notes MCPを使用)
- Claude Code
- MCP サーバー:
  - Apple Notes MCP
  - Gmail MCP
- Python 3.x (OAuth認証用)

## 背景・課題

結婚式準備では、以下のような情報が複数の場所に分散します：

- **iOSメモ**: 打ち合わせ内容、宿題、決定事項
- **Gmail**: 式場・業者からの連絡メール
- **iCloud**: 見積書、注文書などのPDF

毎回「あのメールどこだっけ？」「メモに何て書いたっけ？」と探すのは非効率。かといってコードを書いて管理ツールを作るのも面倒...。

そこで、**Claude CodeのMCP連携とSkill機能を使って、自然言語で質問するだけで情報を集約できる仕組み**を作りました。

## 実装手順

### 1. プロジェクト構造

```
wedding-planner/
├── .claude/
│   ├── settings.json        # MCP サーバー設定
│   ├── skills/              # Skill定義
│   │   ├── wedding-gmail.md # Gmail検索Skill
│   │   └── wedding-notes.md # Apple Notes検索Skill
│   └── commands/            # カスタムコマンド
├── data/
│   ├── todos.json           # タスクデータ
│   └── orders.json          # 注文データ
├── outputs/
│   └── report.md            # 総合レポート
└── CLAUDE.md                # プロジェクト指示書
```

### 2. Apple Notes MCPの設定

`.claude/settings.json` でApple Notes MCPを有効化：

```json
{
  "mcpServers": {
    "apple-notes": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-apple-notes"]
    }
  }
}
```

**重要**: `CLAUDE.md` で読み取り対象フォルダを限定：

```markdown
### Apple Notes MCP
Apple Notes MCPでメモを読む際は、**必ず「結婚式」フォルダのみを対象にすること。**
他のフォルダのメモは絶対に読まない。

- `list_notes` を呼ぶ際は必ず `folderId` を指定する
- `folderId`: `x-coredata://496BE4E0-9728-4490-94D1-4D55A35581A5/ICFolder/p328`
```

### 3. Gmail MCPの設定

OAuth認証を使ってGmail MCPを設定。Yahoo Mailから転送されたメールを検索します。

`.claude/settings.json`:

```json
{
  "mcpServers": {
    "gmail": {
      "command": "python",
      "args": ["-m", "mcp_gmail.server"],
      "env": {
        "GOOGLE_CLIENT_ID": "your-client-id",
        "GOOGLE_CLIENT_SECRET": "your-client-secret"
      }
    }
  }
}
```

### 4. Skill定義で自然言語質問に対応

**wedding-gmail.md** の冒頭（最重要部分）：

```yaml
---
description: ユーザーが結婚式準備の進捗・TODO・業者とのやりとりを確認したい時、GmailのYahoo転送メール（手動・自動転送の両方対応）から結婚式関連情報とTODOを抽出する。式場・花屋・衣装・料理・プランナー・ゲストなど結婚式関連の質問で自動実行。開発・技術的な質問では実行しない。
---

## 自動実行トリガー

### ✅ このSkillを使うべき質問
- 結婚式準備の進捗・状況確認
- 業者（式場・花屋・衣装・料理・プランナーなど）とのやりとり
- メールでの連絡内容の確認
- TODO・タスク・期限の確認
- 打ち合わせ内容・返信待ちの確認

**キーワード例**:
- 業者名: 式場、花屋、衣装、料理、プランナー、ゲスト、TAKAMI、グランドニッコー
- 動作: 進捗、どうなった、いつまで、確認、やること、次の、最近、最新、連絡、メール
- 状態: 返信、待ち、予定、期限、締切

### ❌ このSkillを使わない質問
- プロジェクト構造・コードの説明
- バグ修正・開発タスク
- ツール・設定の使い方
- Git操作・ファイル操作
- MCP接続の技術的な問題
```

**検索クエリ（手動・自動転送両対応）**:

```
(from:yahoo.co.jp OR to:alfortx@yahoo.co.jp) (結婚式 OR ウェディング OR 式場 OR TAKAMI OR グランドニッコー OR 衣装 OR 宿泊 OR プランナー OR 太田 OR 打ち合わせ OR 料理 OR ゲスト OR 親族)
```

**ポイント**:
- `from:yahoo.co.jp`: 手動転送（Fromが書き換わる）
- `to:alfortx@yahoo.co.jp`: 自動転送（Toは元のまま）
- 両方をORで繋ぐことで、転送方法の変更に柔軟に対応

### 5. CLAUDE.mdでSkill自動判断基準を定義

```markdown
## Skillの自動判断基準

Claude Codeは以下のロジックでSkillを自動選択します。

### 結婚式準備に関する質問 → Skill自動実行
ユーザーが「メール」「メモ」と明示しなくても、以下のような質問でSkillが自動実行されます：

**トリガーとなる質問パターン**:
- 進捗・状況確認: 「〜の進捗は？」「〜どうなった？」「〜の状況は？」
- 期限・予定確認: 「次の〜はいつ？」「〜の期限は？」「〜までに何がある？」
- TODO確認: 「やることは？」「宿題ある？」「確認することは？」
- 業者関連: 「式場から連絡あった？」「花屋さんどうなった？」「プランナーさんの話は？」

### 開発・技術に関する質問 → Skill不要
以下のような質問ではSkillは実行されません：

**判断基準**:
- コード・実装の説明
- バグ修正・開発タスク
- プロジェクト構造・ファイル構成
- ツール・設定の使い方
```

## ハマったポイントと解決策

### 1. Yahooメール転送の仕様が分からない

**問題**: Yahoo自動転送を使った場合、Gmailでどういうヘッダーになるのか不明。`from:yahoo.co.jp` で検索できなくなる可能性がある。

**解決策**:
1. 実際にYahoo転送メール（手動転送・connpassなどの自動転送）のヘッダーを検証
2. 手動転送: `From: yahoo.co.jp`
3. 自動転送: `From: 元の送信元`, `To: yahoo.co.jp`
4. **両方をカバーする検索クエリ**を作成:
   ```
   (from:yahoo.co.jp OR to:alfortx@yahoo.co.jp)
   ```

### 2. Skillが自動実行されない

**問題**: 「花屋さんの件どうなった?」と質問しても、「メール」「メモ」と明示しないとSkillが呼ばれない。

**解決策**:
1. Skill定義の `description` フィールドを拡充
2. 「いつ使うべきか」「どんな質問か」「使わない条件」を明記
3. `## 自動実行トリガー` セクションを追加して、✅使うべき質問 / ❌使わない質問を明確化
4. `CLAUDE.md` に「Skillの自動判断基準」セクションを追加

**修正後の description**:
```yaml
description: ユーザーが結婚式準備の進捗・TODO・業者とのやりとりを確認したい時、GmailのYahoo転送メール（手動・自動転送の両方対応）から結婚式関連情報とTODOを抽出する。式場・花屋・衣装・料理・プランナー・ゲストなど結婚式関連の質問で自動実行。開発・技術的な質問では実行しない。
```

### 3. Gmail検索が本文まで見ていない

**問題**: 件名だけでなく本文も検索したい。

**解決策**: Gmail検索は `subject:` を付けない限り、デフォルトで件名+本文を検索する。現在の設定で問題なし。

## 使い方

自然言語で質問するだけ：

```
# 結婚式準備に関する質問（自動でメモ・メールを探索）
「花屋さんの件どうなった?」
「次の打ち合わせいつ?」
「式場から連絡あった?」
「やることある?」

# その他の指示
「最近のメモを読んでTODOを更新して」
「iCloudの結婚式フォルダから注文情報を抽出して」
「今週のTODO一覧を見せて」
```

## まとめ

Claude CodeのMCP連携とSkill機能を活用することで、**コードを書かずに**自然言語だけで複数データソースを統合したタスク管理システムを構築できました。

**ポイント**:
- **Skill定義の `description` と自動実行トリガー**を丁寧に書くことで、自然言語での質問に対応
- **`from:` と `to:` の両方**を使うことで、転送方法の変更に強い検索を実現
- **`CLAUDE.md` でプロジェクト固有の制約**（フォルダID、検索条件など）を明示

今後は、カレンダー連携やLINE Botでのスマホ対応も検討中です。

コード不要で自然言語だけで動くシステム、意外と実用的です！
