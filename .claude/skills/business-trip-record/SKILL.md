---
name: business-trip-record
description: Use when the user provides boarding passes, hotel check-in info, or asks to record/file a business trip (出張記録/搭乗証明書/宿泊証明/旅費精算/税理士提出). Generates a printable HTML trip record under `/tmp/` and delivers it to the user via SendUserFile for manual upload to Google Drive.
---

# 出張記録スキル（business-trip-record）

会社の出張があったとき、税理士提出用の出張記録HTML（搭乗証明書＋宿泊記録＋旅費規程適用）を生成して、ユーザーへ送付します。

## 重要：個人情報の取扱い

出張記録には氏名・宿泊先・客室番号・JAL予約ID等の個人情報が含まれます。**このリポジトリ（公開GitHub Pagesサイト）には絶対にコミットしないこと**。

- 出力先：`/tmp/business-trips/` （セッション内一時領域、リポジトリ外）
- 配信方法：`SendUserFile` で proactive 送付
- ユーザーが手動でGoogle Driveへアップロードする想定
- 過去に `boarding-passes.html` を一時的にこのリポジトリへ作成したが削除済み。同様の誤りを繰り返さないこと

## トリガー

ユーザーが以下のいずれかをした時に発動：
- 搭乗証明書（JAL/ANA等）の画像・テキストを共有
- ホテルのキーカードホルダー・チェックイン情報を共有
- 「出張記録して」「税理士提出用に保管して」「経費申請」と依頼
- 旅費精算・出張費の計上について相談

## 出力ファイル

```
/tmp/business-trips/YYYY-MM-DD-<destination-slug>.html
```

例：
- `/tmp/business-trips/2026-05-16-takamatsu.html`
- `/tmp/business-trips/2026-07-03-osaka.html`

生成後、必ず `SendUserFile` で `status: "proactive"` 指定でユーザーへ送付し、「ブラウザで開き、印刷→PDF保存→Google Driveへアップロードしてください」と案内する。

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

HTML構造は以下の要件を満たすこと：

- `<!DOCTYPE html>` + `<html lang="ja">` + Tailwind CDN + Noto Sans JP
- `@media print` で toolbar 非表示、各セクションを `page-break-after: always` で改ページ
- 上部ツールバーに「印刷 / PDF保存」ボタン（`window.print()`）
- JAL搭乗証明書セクション：赤●ロゴ + `JAPAN AIRLINES` テキスト、Web ID と発行日時を右上に転記
- 宿泊記録セクション：ホテル名を金茶色（`#94714b`）で見出し化
- 旅費規程適用セクション：費目×（適用条文・単価／数量／計上方法）の3列表

詳細レイアウトは過去版（git history の `boarding-passes.html`）を参照。

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

## 配信フロー（必須）

1. `/tmp/business-trips/` ディレクトリを作成（存在しなければ `mkdir -p`）
2. HTMLを生成して `/tmp/business-trips/YYYY-MM-DD-<destination>.html` に書き出し
3. `SendUserFile` で配信（`status: "proactive"`）
4. ユーザーへ以下を案内：
   - 「ファイルを開いて画面上部の『印刷 / PDF保存』ボタンからPDF化してください」
   - 「PDFをGoogle Driveの経費フォルダ等にアップロードしてください」
5. **絶対に `git add` / `git commit` しない**

## 注意事項

- 金額は原則として記載しない（規程単価のみ）。税理士側で領収書と規程に基づき計上する前提。
- ユーザーが「金額も入れて」と明示した場合のみ金額欄を追加。
- 搭乗証明書の Web ID（`Web xxxxx...`）と発行日時は必ず転記。
- ホテル情報は「正式な領収書はホテル発行のものを別途参照」と注記すること（このページは記録メモであり領収書ではない）。
- 個人情報（パスポート番号・クレジットカード番号等）は転記しない。Wi-Fi パスワード等の宿泊施設情報も記載不要。
- **再掲：このリポジトリへのコミット禁止**。`/tmp` への一時生成 + `SendUserFile` 配信のみ。
