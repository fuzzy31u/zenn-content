---
title: "🌸【学校プリントを Slack に投げるだけ】Gemini で家族カレンダーを自動更新する AI エージェント Hanamaru を作った"
emoji: "🌸"
type: "tech"
topics: ["googlecloud", "gemini", "typescript", "claudecode"]
published: true
---

## はじめに

うちには小学校高学年の子どもが 2 人と保育園児が 1 人います。学校・塾・習い事・保育園からの予定連絡は、紙のプリント、LINE、メール、習い事専用アプリにバラバラに届きます。それを親が読んで、Google Calendar に手で転記する。1 件 1 件は数分でも、毎週積み重なると相当な認知負荷です。転記漏れで遠足のお弁当を忘れかけたこともあります。

この「読む → 転記する」を丸ごと AI エージェントに任せたい。そう思って作ったのが **Hanamaru** です。プリントの写真を Slack に投稿すると、Gemini が予定を抽出して、子どもごとの Google Calendar に自動登録してくれます。

本記事では、Hanamaru のアーキテクチャと実装のポイントをまとめます。

- リポジトリ: https://github.com/fuzzy31u/hanamaru

## 作ったもの

使い方はこれだけです。

1. 学校プリントの写真(または LINE のスクショ、テキスト)を Slack の `#hanamaru` チャネルに投稿する
2. Hanamaru が画像とテキストを読み、予定情報を抽出する
3. 抽出結果の信頼度が高ければそのまま Google Calendar に登録、曖昧ならスレッドで「これで登録していい?」と聞いてくる

```
[ Slack 投稿: 写真 / テキスト ]
         │
         ▼
[ Vertex AI Gemini 2.5 Flash ]  ← マルチモーダル抽出
         │
         ▼
[ Google Calendar(子どもごと) ]
```

1 枚の写真から複数イベントの抽出にも対応していて、投稿から登録完了まで 10 秒かかりません。カレンダーは子どもごとに分かれているので、家族全員の予定が色分けで一覧できます。

投稿には prefix 構文を用意しました。

```text
[#長女 | #長男 | #末っ子 | #自分 | #? | #!!] {本文 / 画像添付}
```

`#長女` のように誰の予定かを明示したり、`#?` で確認を強制したり、`#!!` で信頼度を無視して自動登録させたりできます。何も付けなければ AI が本文から判断します。

## アーキテクチャ

| レイヤ | 選択 | 理由 |
|---|---|---|
| コンピュート | Cloud Run (asia-northeast1) | scale-to-zero、家庭利用の流量なら実質無料枠 |
| AI モデル | Vertex AI Gemini 2.5 Flash | マルチモーダル + Structured Output + ADC 認証で API キー管理不要 |
| 永続化 | Firestore (Native mode) | 冪等性キー・確認待ち・属性辞書。TTL で自動削除 |
| シークレット | Secret Manager | Slack 秘密鍵と Google OAuth のみ。Vertex AI は ADC で完結 |
| IaC | Terraform | Cloud Run / Firestore / Secret Manager / IAM を全部コード管理 |
| アプリ | TypeScript (strict) + Hono | 型安全、コールドスタートが速い |

構成図にするとこうなります。

```
 Slack (#hanamaru)
       │ Events API webhook(署名検証 → 3 秒以内に 200 OK)
       ▼
 ┌───────────────────────────────┐
 │ Cloud Run (asia-northeast1)   │
 │ TypeScript + Hono             │
 └──┬──────────┬──────────┬──────┘
    │          │          │
    ▼          ▼          ▼
 Vertex AI   Firestore   Secret Manager
 Gemini      (Native)    (Slack鍵 / OAuth)
 2.5 Flash
    │
    ▼
 Google Calendar API(子どもごと)
```

Cloud Run の実行用サービスアカウントには `aiplatform.user` / `datastore.user` / `secretmanager.secretAccessor` だけを付与していて、プロジェクト全体の権限は渡していません。デプロイも GitHub Actions から鍵ファイルなしで行っていて、Workload Identity Federation で発行した一時クレデンシャルを使い、しかも `main` ブランチの `deploy.yml` からのみ Artifact Registry への push と Cloud Run への revision 作成を許可する、という Terraform の attribute condition で縛っています。エラーログ(severity >= ERROR)が出たら Cloud Monitoring のログベースアラートが Slack に通知してくれるので、障害に気づかず放置することもありません。

処理の流れは Slack Events API の webhook を受けて、署名検証 → 3 秒以内に 200 OK を返却 → 後段のパイプライン(抽出 → 属性決定 → 信頼度ルーティング → Calendar 書き込み → Slack 返信)を非同期で実行、という構成です。Slack は 3 秒以内に ack しないとリトライしてくるので、ここは同期処理にできません。

パイプラインはモジュールを細かく分けました。

```
src/
├── adapters/    # Gemini / Google Calendar / Slack / Secret Manager
├── pipeline/    # extractor → attributor → confidence → calendar-writer → replier
├── stores/      # idempotency / pending / attribution-hints (Firestore)
├── handlers/    # Slack events / reactions
└── config/      # 子ども定義・閾値・スキーマ
```

## 実装のポイント

### 1. Structured Output + Zod の二重チェック

抽出の心臓部は Gemini の `generateContent` 1 回です。`responseSchema` に JSON Schema を渡すと、Vertex AI 側が出力の JSON 妥当性を保証してくれます。

```ts
const response = await ai.models.generateContent({
  model: 'gemini-2.5-flash',
  contents: [{
    role: 'user',
    parts: [
      ...input.images.map(img => ({
        inlineData: { mimeType: img.mimeType, data: img.base64 },
      })),
      { text: input.text },
    ],
  }],
  config: {
    systemInstruction: buildSystemInstruction(input.postedAt),
    responseMimeType: 'application/json',
    responseSchema: extractionResponseSchema,
    temperature: 0.2,
  },
})

const parsed = ExtractionResponse.parse(JSON.parse(response.text))
```

ポイントは最後の 1 行で、モデルが返した JSON を Zod で再パースしています。`responseSchema` があっても enum 外の値や微妙な型ズレは起こり得るので、アプリ層の型はあくまで Zod で回収する。防御的な二重チェックです。

実際、運用を始めてすぐ「Gemini が `2026-06-10T09:00:00+09:00` のようにタイムゾーンオフセット付き ISO 8601 を返してくるが、スキーマがオフセットなしを前提にしていて Zod で落ちる」というバグを踏みました。LLM の出力形式は「だいたい合ってるが揺れる」前提で設計しておくと、こういうとき壊れ方が穏やかになります(この件はログに Zod のパース詳細を出していたので即特定できました)。

### 2. 信頼度ベースのルーティング

AI が抽出した予定をノーチェックで全部カレンダーに書くのは怖い。かといって毎回確認を求められるなら手動転記と変わりません。そこで抽出結果に信頼度スコアを付けさせ、閾値でルーティングしています。

```ts
const isHighConfidence = (e: ExtractedEvent) =>
  e.attributionConfidence >= 0.8 &&
  e.datetimeConfidence >= 0.8 &&
  e.attributedTo !== 'unknown'

const route = isHighConfidence(event) ? 'auto-register' : 'ask'
```

高信頼なら自動登録して事後通知、低信頼ならスレッドで確認を求めて、✅ リアクションで登録・❌ で破棄という流れです。誤登録した場合も、通知メッセージに ❌ を押せば取り消せます。

閾値は `config/thresholds.ts` に一元化して、最初は保守的に設定しました。「誤登録ゼロを 2 週間確認してから緩める」という運用です。家族のカレンダーに間違った予定が入ると信頼を一気に失うので、初期は「確認が多くてうざい」側に倒すのが正解だと思っています。

### 3. 属性決定は「prefix > 辞書 > AI」の 3 段フォールバック

「この予定は誰のか」の判定は、AI 任せにせず優先順位を付けました。

1. ユーザーが `#長女` prefix を付けていれば無条件でそれを採用(confidence 1.0)
2. なければ Firestore の属性辞書(`attribution_hints`)と文字列マッチ
3. それでも不明なら Gemini の推定を使い、低信頼なら確認に回す

面白いのは 2 の辞書が育っていくところです。ユーザーが確認メッセージに ✅ を押すと、その投稿に含まれるキーワード(塾の名前など)が辞書に追加されます。使うほど確認が減っていく、軽量な学習ループです。

### 4. 冪等性 — Slack はしつこくリトライしてくる

Slack Events API は 3 秒以内に応答がないと同じイベントを最大 3 回再送してきます。ネットワークの揺らぎでも重複配信は普通に起きます。何もしないと同じ遠足が 3 件登録される。

対策として、Slack のイベント ID をキーに Firestore のトランザクションで「処理権」を取得するイデンポテンシーストアを実装しました。トランザクション内で「ドキュメントが存在しなければ作成して処理続行、存在すれば早期 return」とすることで、同時に届いた重複イベントも片方しか処理されません。この部分は Firestore エミュレータを使った統合テストで、並行リクエストを実際に投げて検証しています。

### 5. テストは Vitest で 53 件

ユニットテスト 37 件と、Firestore エミュレータを使う統合テスト 16 件を書きました。prefix パーサや信頼度判定のような純粋ロジックはユニットで網羅し、冪等性やオーケストレータの分岐はエミュレータ込みの統合テストで押さえる、という切り分けです。Gemini や Slack など外部 API はアダプタ層に隔離してモックしています。

## 開発プロセス: 設計ドキュメント駆動 + Claude Code

実装は設計ドキュメントの起草からメインブランチへのマージまで 4 日でした(6/6 設計 → 6/7 実装計画 → 6/10 マージ)。家事育児の合間の作業でこのスピードが出たのは進め方の影響が大きいので、記録しておきます。

実装はほぼ全編 Claude Code とのペアで進めました。ただし、いきなりコードを書かせるのではなく、

1. ブレストして設計ドキュメント(`docs/superpowers/specs/`)を先に固める
2. 設計から実装計画(`docs/superpowers/plans/`)に落とす
3. 計画の 1 項目 = 1 コミットの粒度で、テストと一緒に実装していく

という流れです。git log を見返すと `feat(schema)` → `feat(config)` → `feat(pipeline)` → `feat(stores)` → `feat(handlers)` と、計画どおりレイヤーの下から積み上がっているのが分かります。

マージ前にはコードレビューを別セッションで実施し、クリティカルな指摘 7 件を修正しました。中身は「200 OK を返した後の非同期処理が Cloud Run の CPU スロットリング(`cpu_idle`)で止まる」「終日イベントの終了日は Google Calendar API では排他的(+1 日)にする必要がある」「エラーメッセージに `err.message` をそのまま載せると内部情報が家族の Slack に漏れる」など、動かしてみるまで気づきにくいものばかりで、レビュー工程を挟んだ価値は大きかったです。

設計を先に書くのは遠回りに見えて、AI とやる場合はむしろ近道でした。設計ドキュメントがあると Claude Code への指示が「計画のタスク 3 をやって」で済み、コンテキストのブレがほぼなくなります。逆に設計が曖昧なままだと、生成されるコードもその曖昧さを引き継ぎます。

## まとめと今後

- 紙のプリント転記は Gemini 2.5 Flash のマルチモーダル抽出 + Structured Output で実用レベルに自動化できた
- 信頼度ルーティングと冪等性など、地味な足回りが家庭内サービスの信頼を支えている

今後は LINE からの直接取り込み、繰り返し予定の検出(毎週の習い事など)あたりを考えています。

正直、いちばんの成果は技術ではなく「プリントの写真を撮って Slack に投げる」が我が家の習慣として定着したことです。ツールは使われ続けてこそ。次は配偶者も投稿できるようにするのが宿題です。

## リンク

- リポジトリ: https://github.com/fuzzy31u/hanamaru (MIT License)
