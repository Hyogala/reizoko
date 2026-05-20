# 🥦 うちの冷蔵庫アプリ — Claude Code 引き継ぎ指示書

## プロジェクト概要

家族で使える冷蔵庫管理Webアプリ。食材の賞味期限管理・AIレシピ提案・買い物リスト連携が主な機能。
Firebase Firestoreをバックエンドに使い、リアルタイム同期で家族全員がデータを共有できる。

---

## 現在のファイル構成

```
fridge-app.html   # アプリ本体（単一HTMLファイル、全機能が入っている）
```

現時点では単一HTMLファイルで完結している。Claude Codeで開発を続ける場合はフレームワーク化（React + Vite など）を推奨するが、まず単一ファイルのまま機能追加しても問題ない。

---

## Firebase 設定情報

```js
const firebaseConfig = {
  apiKey: "AIzaSyCOXwDJKgpD3ciCE4fctiMJ59MFd2tUIRI",
  authDomain: "reizoko-12322.firebaseapp.com",
  projectId: "reizoko-12322",
  storageBucket: "reizoko-12322.firebasestorage.app",
  messagingSenderId: "422304148803",
  appId: "1:422304148803:web:35f5bdf20c9a3c014a13f6"
};
```

### Firestore コレクション構造

| コレクション名 | 用途 | 主なフィールド |
|---|---|---|
| `inventory` | 食材リスト | `name`, `category`, `quantity`, `expiry`, `createdAt` |
| `recipes` | 保存レシピ | `title`, `description`, `time`, `difficulty`, `tags`, `ingredients`, `steps`, `tips`, `favorite`, `createdAt` |
| `shopping` | 買い物リスト | `name`, `quantity`, `recipe`, `done`, `createdAt` |

### Firebase SDK バージョン
- Firebase JS SDK: `10.12.0`（CDN経由で読み込み）
- `type="module"` の ES module 形式

---

## 実装済み機能一覧

### 📦 食材管理
- 食材の追加・削除
- 賞味期限管理（色分け: 緑=余裕, 黄=3日以内, 赤=期限切れ）
- 期限切れ・期限間近のアラート表示
- カテゴリ・期限でのフィルタリング（すべて / 期限間近 / 野菜 / 肉・魚）

### ➕ 食材追加
- 手動入力フォーム（名前・カテゴリ・数量・賞味期限）
- よく使う食材のクイックチップ（18種類）
- 食材名のオートコンプリート
- 📷 写真から食材を自動認識（Claude Vision API使用）
- 賞味期限の自動推定（写真認識時）

### 🤖 AIレシピ提案
- 冷蔵庫の食材からレシピを3つ提案
- 賞味期限が近い食材を優先使用
- 料理の種類・調理時間・自由リクエスト指定可能
- レシピをFirebaseに保存

### 🍳 レシピ帳
- 保存済みレシピ一覧
- お気に入り登録・フィルタリング
- レシピの削除
- 不足食材を買い物リストへ自動追加（AI判定）

### 🛒 買い物リスト
- 手動で商品追加（Enterキー対応）
- AIで不足食材を自動判定・追加
- 購入済みチェックボックス
- 購入済みを一括削除
- 関連レシピ名を表示

### 🔄 インフラ・共有
- Firebase Firestoreでデータ永続保存
- `onSnapshot` によるリアルタイム同期（全端末即時反映）
- 家族での共有対応（同じHTMLを配布するだけ）

---

## ⚠️ 未実装・要対応タスク（優先度順）

### 🔴 最優先: Googleログイン認証（セキュリティ）

**背景**: 現在Firestoreのセキュリティルールが `allow read, write: if true` になっており、URLを知っている人なら誰でもデータにアクセス・改ざんできる危険な状態。

**実装内容**:
1. Firebase Authentication で Google ログインを有効化（Firebaseコンソールで設定済みかどうか確認すること）
2. アプリ起動時にログイン画面を表示
3. ログイン済みユーザーのみアプリを使えるようにする
4. ヘッダーにログインユーザー名・アバター・ログアウトボタンを表示
5. Firestoreセキュリティルールを以下に変更:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /{document=**} {
      allow read, write: if request.auth != null;
    }
  }
}
```

**実装コード例**:
```js
import {
  getAuth,
  GoogleAuthProvider,
  signInWithPopup,
  onAuthStateChanged,
  signOut
} from 'https://www.gstatic.com/firebasejs/10.12.0/firebase-auth.js';

const auth = getAuth(app);
const provider = new GoogleAuthProvider();

// ログイン
async function login() {
  await signInWithPopup(auth, provider);
}

// 認証状態の監視
onAuthStateChanged(auth, user => {
  if (user) {
    // ログイン済み → アプリを表示
    showApp(user);
  } else {
    // 未ログイン → ログイン画面を表示
    showLoginScreen();
  }
});
```

**UIの要件**:
- ログイン画面はシンプルに「Googleでログイン」ボタンのみ
- ログイン後はヘッダーにアバター画像（`user.photoURL`）を表示
- 家族共有を想定しているのでメールアドレス制限は不要（Googleアカウントを持っていれば誰でもOK）

---

### 🟡 中優先: 家族ごとのデータ分離

**背景**: 現在は全ユーザーが同じFirestoreコレクションを共有している。家族Aと家族Bのデータが混在してしまう。

**実装方針**:
- コレクションパスに `families/{familyId}/` を追加する
- または、ログインユーザーのメールドメインでグループ化する
- シンプルな案: 最初のログインユーザーが「招待コード」を生成し、家族はそのコードを入力して同じグループに入る

---

### 🟡 中優先: Claude API キーの安全な管理

**背景**: 現在、フロントエンドのHTMLに直接Anthropic APIを呼んでいる。これはブラウザのネットワークタブからAPIキーが丸見えになるリスクがある（ただしclaude.aiのアーティファクト環境では認証が代わりに処理されていたため問題なかった）。

**実装方針**:
- Firebase Cloud Functions を使ってバックエンドでClaudeを呼ぶ
- または Vercel Edge Functions などのサーバーレス関数を使う
- フロントエンドからは `/api/generate-recipe` のような自前エンドポイントを叩く形にする

---

### 🟢 低優先: 追加したい機能（オーナーからのリクエスト）

- [ ] 賞味期限が近づいたときのプッシュ通知（Firebase Cloud Messaging）
- [ ] 食材のバーコードスキャンで自動入力
- [ ] 食材の消費履歴・統計グラフ
- [ ] レシピのカテゴリ分類・タグ検索
- [ ] 買い物リストのカテゴリ別並び替え（スーパーの売り場順など）

---

## 技術スタック

| 項目 | 内容 |
|---|---|
| フロントエンド | 純粋なHTML / CSS / JavaScript（フレームワークなし） |
| データベース | Firebase Firestore |
| 認証 | Firebase Authentication（未実装） |
| AI | Anthropic Claude API（claude-sonnet-4-20250514） |
| フォント | Fraunces（見出し）、Zen Kaku Gothic New（本文）|
| ホスティング | 未定（Netlify / Firebase Hosting / Vercel など推奨） |

---

## デザイン仕様

### カラーパレット
```css
--accent:       #3a7d44   /* メインカラー（緑） */
--accent-dark:  #2a5c32   /* 濃い緑 */
--accent-light: #e8f5e9   /* 薄い緑（背景） */
--orange:       #e07b39   /* サブカラー（レシピ・AI系） */
--yellow:       #e8b84b   /* 警告色（期限間近） */
--red:          #d94f4f   /* 危険色（期限切れ） */
--ink:          #1c2118   /* メインテキスト */
--ink2:         #6b7a65   /* サブテキスト */
--bg:           #f4f7f2   /* 背景 */
--line:         #e2e8df   /* ボーダー */
```

### レイアウト
- モバイルファースト（最大幅 480px、中央寄せ）
- ヘッダーとタブバーは `position: sticky` でスクロール追従
- カードベースのUI（`border-radius: 16px`、`box-shadow`あり）

---

## 開発時の注意点

1. **Firebase SDK はCDN（ES module）で読み込んでいる** — `type="module"` が必須。`npm install firebase` して使う場合はバンドラー（Vite等）が必要になる。

2. **`window.xxx =` で関数を公開している** — `type="module"` 内の関数はデフォルトでスコープが閉じているため、HTMLの `onclick="xxx()"` から呼ぶためには `window.xxx = function` の形で公開が必要。React化した場合はこの問題はなくなる。

3. **Anthropic APIの直接呼び出し** — 現在フロントから直接 `https://api.anthropic.com/v1/messages` を呼んでいる。本番環境ではCloud Functions等を経由させること。

4. **セキュリティルールは必ず変更すること** — `allow read, write: if true` のまま公開しないこと。

---

## Claude Codeへの最初の指示（コピペ用）

```
fridge-app.html を開いて内容を確認してください。
このファイルはFirebaseを使った冷蔵庫管理Webアプリです。

まず最優先タスクとして、Googleログイン認証を実装してください。

要件:
- Firebase Authentication（Google Provider）でログイン機能を追加
- 未ログイン時はログイン画面のみ表示（アプリ本体は非表示）
- ログイン後はヘッダーにユーザーアバター・名前・ログアウトボタンを表示
- Firestoreのセキュリティルールを「認証済みユーザーのみ読み書き可」に変更する手順も教えてください
- 既存のデザイン（カラー・フォント・スタイル）は変えないでください
```
