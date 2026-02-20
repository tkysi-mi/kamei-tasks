# Fireflies → GitHub Issues 自動連携ガイド

## 概要

Fireflies.aiの議事録からアクションアイテムを自動抽出し、GitHub Issueとしてプロジェクトボードに追加する仕組み。

```
📹 会議（Zoom/Meet/Teams）
    ↓ Fireflies.aiが自動録音・文字起こし
📝 議事録 + アクションアイテム抽出
    ↓ API連携（自動 or 手動トリガー）
📋 GitHub Issues → プロジェクトボード（バックログ）
```

---

## 前提条件

| 項目 | 詳細 |
|------|------|
| Fireflies.ai アカウント | Business プラン以上（API利用に必要） |
| Fireflies API Key | Fireflies設定画面から取得 |
| GitHub Personal Access Token | repo + project スコープ |
| GitHub リポジトリ | kamei-tasks（Issue作成先） |
| GitHub Project | プロジェクトボード（自動追加設定済み） |

---

## Fireflies API

### エンドポイント
```
POST https://api.fireflies.ai/graphql
Authorization: Bearer {FIREFLIES_API_KEY}
```

### 議事録一覧の取得

```graphql
query {
  transcripts {
    id
    title
    date
    duration
    participants
  }
}
```

### アクションアイテムの取得

```graphql
query Transcript($transcriptId: String!) {
  transcript(id: $transcriptId) {
    title
    date
    participants
    summary {
      action_items      # ← アクションアイテム（配列）
      overview          # ← 会議の概要
      topics_discussed  # ← 議題リスト
    }
    meeting_attendees {
      displayName
      email
    }
  }
}
```

### レスポンス例

```json
{
  "data": {
    "transcript": {
      "title": "週次定例MTG 2026-02-20",
      "participants": ["亀井", "田中", "佐藤"],
      "summary": {
        "action_items": [
          "亀井さんがA社への見積もりを来週月曜までに作成する",
          "田中さんがLP改善のデザイン案を共有する",
          "佐藤さんがコンサル契約書のドラフトを確認する"
        ],
        "overview": "今週の進捗報告と来週のタスク確認を行った",
        "topics_discussed": [
          "A社プロジェクトの進捗",
          "LP改善施策",
          "新規クライアント対応"
        ]
      }
    }
  }
}
```

---

## GitHub API

### Issue作成
```
POST https://api.github.com/repos/{owner}/{repo}/issues
Authorization: Bearer {GITHUB_PAT}
```

```bash
curl -X POST \
  -H "Authorization: Bearer {GITHUB_PAT}" \
  -H "Accept: application/vnd.github+json" \
  "https://api.github.com/repos/tkysi-mi/kamei-tasks/issues" \
  -d '{
    "title": "[議事録] A社への見積もりを来週月曜までに作成する",
    "body": "## 元の議事録\n- 会議名: 週次定例MTG\n- 参加者: 亀井、田中、佐藤\n\n## アクションアイテム\nA社への見積もりを来週月曜までに作成する\n\n---\n*Fireflies.aiから自動生成*"
  }'
```

プロジェクトの Auto-add 設定により、自動でバックログに追加される。

---

## 連携スクリプト（Python）

```python
"""
Fireflies → GitHub Issues 連携スクリプト
使い方: python sync_tasks.py <transcript_id>
"""
import requests
import sys

# === 設定 ===
FIREFLIES_API_KEY = "your_fireflies_api_key"
GITHUB_PAT = "your_github_pat"
GITHUB_REPO = "tkysi-mi/kamei-tasks"


def get_action_items(transcript_id):
    """Firefliesからアクションアイテムを取得"""
    query = """
    query Transcript($id: String!) {
      transcript(id: $id) {
        title
        date
        participants
        summary { action_items overview }
        meeting_attendees { displayName }
      }
    }
    """
    resp = requests.post(
        "https://api.fireflies.ai/graphql",
        headers={
            "Authorization": f"Bearer {FIREFLIES_API_KEY}",
            "Content-Type": "application/json",
        },
        json={"query": query, "variables": {"id": transcript_id}},
    )
    return resp.json()["data"]["transcript"]


def create_issue(title, body):
    """GitHub Issueを作成"""
    resp = requests.post(
        f"https://api.github.com/repos/{GITHUB_REPO}/issues",
        headers={
            "Authorization": f"Bearer {GITHUB_PAT}",
            "Accept": "application/vnd.github+json",
        },
        json={"title": title, "body": body},
    )
    return resp.json()


def sync(transcript_id):
    """メイン処理: 議事録 → GitHub Issues"""
    meeting = get_action_items(transcript_id)

    action_items = meeting["summary"]["action_items"]
    meeting_title = meeting["title"]
    participants = ", ".join(meeting.get("participants", []))

    print(f"📝 会議: {meeting_title}")
    print(f"👥 参加者: {participants}")
    print(f"📋 アクションアイテム: {len(action_items)}件\n")

    for item in action_items:
        title = f"[議事録] {item}"
        body = (
            f"## 元の議事録\n"
            f"- **会議名**: {meeting_title}\n"
            f"- **参加者**: {participants}\n\n"
            f"## アクションアイテム\n{item}\n\n"
            f"---\n*Fireflies.aiから自動生成*"
        )
        result = create_issue(title, body)
        print(f"  ✅ Issue #{result['number']}: {item}")


if __name__ == "__main__":
    sync(sys.argv[1])
```

---

## 自動化のオプション

| 方法 | 難易度 | 説明 |
|------|--------|------|
| **手動実行** | ⭐ | 会議後に `python sync_tasks.py <id>` を実行 |
| **Webhook** | ⭐⭐⭐ | Firefliesが議事録完成時に自動トリガー |
| **Zapier/Make** | ⭐⭐ | ノーコードでFireflies→GitHub連携 |

### 推奨: まずは手動実行で始める

```bash
python sync_tasks.py <transcript_id>
```

慣れたらWebhookやZapierで自動化を検討する。

---

## 運用フロー

```
1. 📹 会議を実施（Firefliesが自動参加・録音）
2. ⏳ 5-10分後、Firefliesが議事録を自動生成
3. 📋 スクリプト実行でアクションアイテムをIssue化
4. ✅ プロジェクトボードのバックログに自動追加
5. 🔄 ボード上でステータスを管理（進行中→相手ボール→完了）
```

---

## 注意事項

- Fireflies APIは **Businessプラン以上** で利用可能
- 日本語の議事録でも動作するが、英語に比べて精度が下がる場合がある
- 重要な会議後はアクションアイテムの手動確認を推奨
- APIレート制限あり（[Fireflies API Limits](https://docs.fireflies.ai/fundamentals/limits) 参照）
