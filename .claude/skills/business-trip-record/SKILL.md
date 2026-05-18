---
name: business-trip-record
description: Use when the user provides boarding passes, hotel check-in info, or asks to record/file a business trip (出張記録/搭乗証明書/宿泊証明/旅費精算/税理士提出). Generates a printable HTML trip record and uploads it to the dedicated Google Drive folder under 旅費規程/旅費規程スキーム/出張報告書/Claude生成出張記録.
---

# 出張記録スキル（business-trip-record）

会社の出張があったとき、税理士提出用の出張記録HTML（搭乗証明書＋宿泊記録＋旅費規程適用）を生成し、Google Drive の指定フォルダへ自動アップロードします。

## 重要：個人情報の取扱い

出張記録には氏名・宿泊先・客室番号・JAL予約ID等の個人情報が含まれます。**このリポジトリ（公開GitHub Pagesサイト）には絶対にコミットしないこと**。過去に `boarding-passes.html` を一時的にこのリポジトリへ作成したが削除済み。同様の誤りを繰り返さないこと。

- 一時生成先：`/tmp/business-trips/`（リポジトリ外）
- 最終保存先：Google Drive 既定フォルダ（下記）
- 配信方法：MCP 経由で Drive へ直接アップロード（`SendUserFile` は不要、ただしユーザーが手元コピーを要求した場合は併用可）

## Google Drive 保存先（固定）

**フォルダ階層**：
```
マイドライブ
└─ 旅費規程
   └─ 旅費規程スキーム
      └─ 出張報告書
         └─ Claude生成出張記録  ← ★ ここに溜める
```

**Folder ID（parentId として使用）**：`1Ouc0oIcLIo0l8PWoTIOyLWlwmeqgoX3s`
**View URL**：https://drive.google.com/drive/folders/1Ouc0oIcLIo0l8PWoTIOyLWlwmeqgoX3s

**ファイル命名規則**：
```
YYYY-MM-DD_<行先>_<訪問先または用件>.html
```

例：
- `2026-05-16_高松_らくあ様訪問.html` ← 2026年5月分（既に格納済み）
- `2026-07-03_大阪_◯◯展示会.html`
- `2026-08-20_福岡_△△商談.html`

## トリガー

ユーザーが以下のいずれかをした時に発動：
- 搭乗証明書（JAL/ANA等）の画像・テキストを共有
- ホテルのキーカードホルダー・チェックイン情報を共有
- 「出張記録して」「税理士提出用に保管して」「経費申請」と依頼
- 旅費精算・出張費の計上について相談

## アップロード手順

1. `/tmp/business-trips/` ディレクトリを作成
2. HTML を生成して `/tmp/business-trips/YYYY-MM-DD_<行先>_<用件>.html` に書き出し
3. `mcp__cba690db-cf76-43b0-bef8-8c854f4130c9__create_file` で Drive へアップロード：
   - `parentId`: `1Ouc0oIcLIo0l8PWoTIOyLWlwmeqgoX3s`
   - `title`: 上記命名規則
   - `contentMimeType`: `text/html`
   - `disableConversionToGoogleType`: `true`（HTMLのまま保存）
   - `textContent`: HTMLの全文
4. アップロード成功後、ファイルの View URL（`https://drive.google.com/file/d/<id>/view`）をユーザーへ案内
5. **絶対に `git add` / `git commit` しない**

## テンプレート構造（必ずこの5セクション順）

1. **出張概要**（BUSINESS TRIP SUMMARY）
   - 出張者・役職区分・所属
   - 期間・出張区分（宿泊出張／日帰り）
   - 訪問先・出張目的
   - 交通手段・宿泊先
2. **旅費規程適用**（TRAVEL EXPENSE BREAKDOWN）
   - 適用規程・適用区分
   - 費目別表（交通費／宿泊費／日当）
3. **搭乗証明書 1（往路）**
4. **搭乗証明書 2（復路）** ※片道のみなら省略可
5. **宿泊記録** ※日帰り出張なら省略

HTML構造の要件：

- `<!DOCTYPE html>` + `<html lang="ja">` + Tailwind CDN + Noto Sans JP
- `@media print` で toolbar 非表示、各セクションを `page-break-after: always` で改ページ
- 上部ツールバーに「印刷 / PDF保存」ボタン（`window.print()`）
- 「戻る」リンクは入れない（Drive で単体表示するため）
- JAL搭乗証明書セクション：赤●ロゴ + `JAPAN AIRLINES` テキスト、Web ID と発行日時を右上に転記
- 宿泊記録セクション：ホテル名を金茶色（`#94714b`）で見出し化
- 旅費規程適用セクション：費目×（適用条文・単価／数量／計上方法）の3列表

詳細レイアウトは Drive 上の `2026-05-16_高松_らくあ様訪問.html`（参考実装）を雛形にする。

## 適用する旅費規程（固定値）

**規程**：株式会社ＪｏｂＳｐａｒｋ「出張・赴任旅費規程」（令和8年5月1日施行）
**出張者**：加藤 アユミ（**役職区分：社長**）
**規程原本**：Google Drive `1HuAlABgQ2Iq-R0q6lfvZoBloiJUM7t7q`（必要時のみ `mcp__cba690db-cf76-43b0-bef8-8c854f4130c9__read_file_content` で再取得）

### 社長区分の単価（国内出張）

| 費目 | 適用条文 | 単価・実費 | 計上方法 |
| --- | --- | --- | --- |
| 交通費 | 第11条第3項①（航空料金） | プレミアムクラス／クラスJ又はエコノミー実費 | 実費精算（領収書・搭乗証明書参照） |
| 宿泊費 | 第12条第3項 | 国内 ¥40,000/泊（別途消費税） | 規定額により計上（第14条 ※実費超過時は別途） |
| 日当 | 第13条第3項 | 国内 ¥10,000/日（別途消費税） | 規定額により計上 |

### 出張区分の判定（第3条・第4条）

- **宿泊出張**：宿泊を伴う、または片道90km以上の移動
- **日帰り出張**：片道90km以上 or 片道3時間以上で当日帰着
- 国内主要区間の片道距離目安：羽田⇄高松 約540km、羽田⇄大阪 約400km、羽田⇄福岡 約880km

### 国外出張の場合（参考）

| 費目 | 社長区分 |
| --- | --- |
| 宿泊費 | ¥60,000/泊 |
| 日当 | ¥20,000/日 |
| 航空 | ビジネスクラス又は実費 |

## 出張目的の特定

1. `project-management.html` を `grep` で該当日付・地名を検索し、該当プロジェクト/打合せを特定
2. 訪問先・目的を文中から要約
3. 不明な場合はユーザーへ `AskUserQuestion` で確認

## 必須収集情報チェックリスト

スキル発動時、以下が揃っているか確認。不足は `AskUserQuestion` で聞く：

- [ ] 出張日（チェックイン〜チェックアウト）
- [ ] 行き先（都市・訪問先・担当者）
- [ ] 出張目的（プロジェクト名／打合せ内容）
- [ ] 交通機関（便名・区間）→ 搭乗証明書画像から抽出
- [ ] 宿泊先（ホテル名・室番号）→ キーカードホルダーから抽出
- [ ] 金額表示の要否（既定：**金額は記載しない**。実費は領収書参照、規程額は単価のみ表記）

## 注意事項

- 金額は原則として記載しない（規程単価のみ）。税理士側で領収書と規程に基づき計上する前提。
- ユーザーが「金額も入れて」と明示した場合のみ金額欄を追加。
- 搭乗証明書の Web ID（`Web xxxxx...`）と発行日時は必ず転記。
- ホテル情報は「正式な領収書はホテル発行のものを別途参照」と注記すること（このページは記録メモであり領収書ではない）。
- 個人情報（パスポート番号・クレジットカード番号等）は転記しない。Wi-Fi パスワード等の宿泊施設情報も記載不要。
- **再掲：このリポジトリへのコミット禁止**。`/tmp` への一時生成 + Drive アップロードのみ。
