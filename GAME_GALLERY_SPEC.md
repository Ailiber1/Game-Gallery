# ゲームギャラリー アプリケーション仕様書

## 1. 概要

Matrix風（サイバーパンク・ハッカー美学）のビジュアルテーマを持つ、ブラウザゲームのギャラリーアプリケーション。ユーザーは一覧からゲームを検索・フィルタリングし、選択して起動できる。

- **アプリ名**: GAME GALLERY
- **言語**: 日本語 UI
- **デザインテーマ**: ダークサイバーパンク / Matrix風（黒背景 + ネオングリーン `#00ff41`）

---

## 2. 技術スタック

| 項目 | 技術 |
|---|---|
| フレームワーク | React 19 + TypeScript |
| ビルドツール | Vite 6 |
| スタイリング | Tailwind CSS（CDN） |
| フォント | システムフォント（monospace + sans-serif）またはnpmセルフホスト |
| 音声 | Web Audio API（自前実装） |
| ホスティング | Firebase Hosting |
| ソース管理 | GitHub |
| 開発ツール | Claude Code |
| パッケージ管理 | npm |

---

## 3. ディレクトリ構成

```
project-root/
├── index.html            # エントリHTML
├── index.tsx             # Reactマウントポイント
├── App.tsx               # メインアプリコンポーネント
├── types.ts              # 型定義
├── data.ts               # ゲームデータ（マスタ）
├── components/
│   ├── Header.tsx        # ヘッダー（タイトル・検索・タブ）
│   ├── GameCard.tsx      # ゲームカード
│   ├── GameIcon.tsx      # プレースホルダーアイコン（SVG）
│   ├── BootModal.tsx     # ゲーム起動モーダル
│   └── Footer.tsx        # フッター
├── services/
│   └── audioService.ts   # サウンドエフェクトサービス
├── package.json
├── tsconfig.json
└── vite.config.ts
```

---

## 4. 型定義 (`types.ts`)

```typescript
export type Category = '人気' | 'おすすめ' | '最新作';

export interface Game {
  id: string;
  name: string;           // ゲーム名
  catchphrase: string;    // キャッチコピー
  description: string;    // 説明文
  genre: string;          // ジャンル名
  players: string;        // プレイ人数（例: "2-5人"）
  playTime: string;       // プレイ時間（例: "10分"）
  tags: string[];         // 検索・表示用タグ
  category: Category[];   // 所属カテゴリ（複数可）
  iconType: 'cards' | 'board' | 'puzzle' | 'action' | 'retro' | 'skill';
  imageUrl?: string;      // パッケージ画像パス（開発者が用意、1:1正方形推奨）
}
```

---

## 5. ゲームデータ（全16件）

| ID | name | genre | iconType | category | players | playTime |
|---|---|---|---|---|---|---|
| 1 | 大富豪 | トランプ | cards | 人気, おすすめ | 2-5人 | 10分 |
| 2 | 花札 | 和風 | cards | おすすめ | 2人 | 15分 |
| 3 | 将棋 | 和風 | board | 人気 | 2人 | 20分 |
| 4 | オセロ | 定番 | board | 人気, 最新作 | 2人 | 5分 |
| 5 | 数独 | パズル | puzzle | おすすめ, 最新作 | 1人 | 15分 |
| 6 | スネーク | レトロ | retro | 人気 | 1人 | 3分 |
| 7 | 2048 | パズル | puzzle | 人気 | 1人 | 10分 |
| 8 | タイピング | スキル | skill | 最新作 | 1人 | 2分 |
| 9 | 神経衰弱 | カード | cards | おすすめ | 1-4人 | 5分 |
| 10 | チェス | 頭脳戦 | board | 人気 | 2人 | 30分 |
| 11 | 麻雀ソリティア | パズル | puzzle | おすすめ | 1人 | 10分 |
| 12 | テトリス風 | レトロ | retro | 最新作 | 1人 | 5分 |
| 13 | リバーシ | 定番 | board | おすすめ | 2人 | 5分 |
| 14 | ポーカー | トランプ | cards | 人気 | 2-10人 | 15分 |
| 15 | ブラックジャック | トランプ | cards | 最新作 | 1-7人 | 3分 |
| 16 | 迷路 | パズル | puzzle | 最新作 | 1人 | 5分 |

各ゲームには `catchphrase`（キャッチコピー）と `description`（説明）、`tags`（検索用タグ配列）も含む。詳細は `data.ts` を参照。

---

## 6. コンポーネント仕様

### 6.1 App（ルートコンポーネント）

**状態管理:**

| state | 型 | 初期値 | 用途 |
|---|---|---|---|
| searchQuery | string | `''` | 検索クエリ |
| activeTab | Category | `'人気'` | 選択中のカテゴリタブ |
| bootingGame | Game \| null | `null` | 起動モーダル表示中のゲーム |

ゲーム一覧は `data.ts` からインポートした定数 `GAMES` をそのまま使用（stateにしない）。

**フィルタリングロジック（`filteredGames`）:**

1. `activeTab` に該当するゲームを先頭にソート（該当カテゴリ持ちが上位）
2. `searchQuery` がある場合、`name` / `description` / `tags` / `genre` で部分一致フィルタ（大文字小文字無視）
3. 最大 **12件** に制限して表示

**サムネイル表示:**

- 各ゲームのパッケージ画像は開発者が事前に用意し、`imageUrl` フィールドに設定する
- 画像サイズ: **1:1（正方形）** 推奨（カードのサムネイルエリアが1:1のため、クロップなしで表示される）
- 画像形式: PNG または JPEG
- 画像が未設定のゲームは `GameIcon`（SVGプレースホルダー）で表示する
- `Game` 型に `imageUrl?: string` フィールドを持たせる（オプション）

**レイアウト:**

```
<div class="min-h-screen matrix-bg">
  <div class="container mx-auto px-4 md:px-8 max-w-7xl py-12">
    <Header />
    <main>
      ゲームカードグリッド（4列: grid-cols-1 sm:2 md:4, gap-8）
      または 該当なしメッセージ
    </main>
    <Footer />
  </div>
  <BootModal />
</div>
```

---

### 6.2 Header

**構成要素:**

1. **ステータスバー**: `System Access // Landing Page` / `Connection: Secure`（点滅）/ `Loc: Tokyo-Node-01` — テキストサイズ `10px`, `uppercase`, `tracking-widest`, `opacity-60`
2. **タイトル**: `GAME GALLERY_` — `text-3xl md:text-4xl`, `glow-text` エフェクト
3. **検索バー**: 虫眼鏡アイコン付き `input`。プレースホルダー「ゲームを検索（例：トランプ、ボード）...」。黒背景、フォーカス時に緑ボーダー + リング
4. **カテゴリタブ**: `人気` / `おすすめ` / `最新作` の3ボタン。アクティブ時は背景緑 + 黒文字、非アクティブは枠線のみ。クリック時にクリック音再生

---

### 6.3 GameCard

**レイアウト（1カードの構成）:**

```
外枠: 黒背景 + 緑ボーダー(20%透過), ホバーで上昇+拡大+グロー
  内枠: 黒背景 + 緑ボーダー(30%透過)
    ┌─ サムネイルエリア（1:1正方形）
    │   imageUrlあり → img表示（object-cover, ホバーで拡大+コントラスト上昇）
    │   imageUrlなし → GameIcon（SVGプレースホルダー）を表示
    │   オーバーレイグラデーション（下から黒、透過）
    │   最新作カテゴリ → 右上に "NEW_CORE" バッジ（グロー付き）
    ├─ ゲーム名 + ID表示
    ├─ キャッチコピー（イタリック、左に緑縦線）
    ├─ タグ一覧（小さなピル型ボーダーバッジ）
    ├─ Players / Duration 情報（2カラム、モノスペース、小さめ）
    └─ "▶ RUN APPLICATION" ボタン
        ホバーで緑背景がスライドアップし、文字が黒に変化
```

**インタラクション:**

- カードホバー: `translateY(-10px) scale(1.02)` + グローシャドウ + ホバー音再生
- 画像ホバー: `scale(1.1)` + コントラスト上昇
- ボタンクリック: マジック音再生 → `onStart(game)` → BootModal表示

---

### 6.4 GameIcon

`iconType` に応じたSVGアイコンをプレースホルダーとして表示。背景に緑グリッドパターン（SVG pattern）。

| iconType | アイコン内容 |
|---|---|
| cards | 重なったカード2枚 |
| board | 3x3グリッドボード |
| puzzle | ダイヤモンド形4ピース |
| action / retro | ゲームコントローラー |
| skill | キーボード |

サイズ: 60x60, ストローク `#00ff41`, ドロップシャドウ(グロー)付き。

---

### 6.5 BootModal

ゲーム起動時のローディング演出モーダル。

**動作フロー:**

1. `game` が非nullになると表示開始
2. 起動音(`playBoot`)再生
3. プログレスバーが 0% → 100% まで自動進行（100msごとに+5%, 計2秒）
4. 進行中メッセージ表示:
   - `> Initializing graphics kernel...`
   - `> Linking network assets...`
   - 50%超: `> Loading player data...`
   - 100%: `> READY TO EXECUTE.`（点滅）
5. "EXECUTE (DEMO)" ボタンクリック → マジック音再生 → モーダル閉じる

**UI:**

- 全画面黒背景(90%透過) + backdrop-blur
- 中央に最大幅 `md` のパネル、`glow-border` 付き
- ヘッダ: `APPLICATION LOADING...`（glow-text）
- ゲームID・名前表示
- プログレスバー: 緑の矩形が左から伸びる
- メモリアドレス風テキスト: `MEM_BLOCK: 0x{id}F3`

---

### 6.6 Footer

- ナビゲーションリンク: `HOME` / `PROFILE` / `SETTINGS` / `LOGOUT`（装飾的、機能なし）
- コピーライト: `© 202X NEO-GENESIS. ALL RIGHTS RESERVED.`
- バージョン: `VER 2.5.0-ALPHA // STABLE`
- 上部に緑ボーダー線、`opacity-60`, モノスペースフォント

---

## 7. サウンドエフェクト仕様 (`audioService.ts`)

Web Audio APIで動的に音を生成。外部音声ファイル不要。

| メソッド | トリガー | 波形 | 概要 |
|---|---|---|---|
| `playHover()` | カードホバー | sine | 1200→1600Hz, 50ms, 極小音量 |
| `playClick()` | タブクリック | square | 880→110Hz, 100ms |
| `playMagic()` | RUNボタン / EXECUTE | sine×2 | 440→880Hz + 880→1760Hz, 1.2秒, フェードイン→アウト |
| `playBoot()` | モーダル表示時 | sine | 220→880Hz, 0.5秒 |

**実装ポイント:**

- `AudioContext` はシングルトンで遅延初期化
- `suspended` 状態チェック（ブラウザの自動再生ポリシー対応）
- 各メソッドで `OscillatorNode` + `GainNode` を都度生成・破棄

---

## 8. ビジュアルデザイン仕様

### 8.1 カラーパレット

| 用途 | カラー |
|---|---|
| 背景（body） | `#000800` |
| 背景（radial gradient） | `#001a00` → `#000000` |
| メインテキスト / アクセント | `#00ff41` |
| ボーダー（暗） | `#003b00` |
| ボーダー（パネル内） | `rgba(0,255,65,0.3)` |
| 強調テキスト | `#ffffff` |

### 8.2 CSSエフェクト

1. **スキャンライン**: `position: fixed`, 3px間隔の横線 + RGB微色差の縦線、`opacity: 0.4`, `pointer-events: none`, `z-index: 100`
2. **グローボーダー** (`.glow-border`): `box-shadow: 0 0 10px rgba(0,255,65,0.2), inset 0 0 5px rgba(0,255,65,0.1)`
3. **グローテキスト** (`.glow-text`): `text-shadow: 0 0 10px rgba(0,255,65,0.8), 0 0 20px rgba(0,255,65,0.3)`
4. **ノイズオーバーレイ** (`.noise::before`): grainy SVGノイズテクスチャ、`opacity: 0.05`
5. **カードホバー**: `translateY(-10px) scale(1.02)`, `box-shadow: 0 0 30px rgba(0,255,65,0.3)`, `cubic-bezier(0.165, 0.84, 0.44, 1)`
6. **画像フィルタ**: 通常時 `grayscale(0.2) contrast(1.1) brightness(0.9)`, ホバー時 `grayscale(0) contrast(1.2) brightness(1.1) scale(1.1)`
7. **スクロールバー**: 幅4px, トラック黒, サム緑 + グロー

### 8.3 フォント

- **英字・コード**: `@fontsource/jetbrains-mono` でセルフホスト、またはmonospace系システムフォント
- **日本語**: `@fontsource/noto-sans-jp` でセルフホスト、またはsans-serif系システムフォント
- セルフホスト時: `font-family: 'JetBrains Mono', 'Noto Sans JP', monospace`
- システムフォント時: `font-family: ui-monospace, SFMono-Regular, Menlo, monospace`

### 8.4 レスポンシブ

- グリッド: `grid-cols-1` → `sm:grid-cols-2` → `md:grid-cols-4`
- コンテナ: `max-w-7xl`, `px-4 md:px-8`
- タイトル: `text-3xl md:text-4xl`

---

## 9. Claude Code 実装時の注意事項

### 9.1 ビルド構成

- Vite + React プラグイン
- `vite.config.ts` でポート `3000`, ホスト `0.0.0.0`
- TypeScript: `target: ES2022`, `jsx: react-jsx`, `moduleResolution: bundler`
- パスエイリアス: `@/*` → プロジェクトルート
- 環境変数: Firebase設定値を `.env.local` で管理

### 9.2 デプロイ

- Firebase Hosting にデプロイ
- `firebase init` → `hosting` 選択 → `dist` ディレクトリ指定
- `npm run build` → `firebase deploy`

### 9.3 パッケージ画像の運用

- パッケージ画像は開発者が事前に用意し、プロジェクト内の画像ディレクトリ（例: `public/images/`）に配置する
- 画像仕様: **1:1正方形**、PNG or JPEG
- `data.ts` のゲームデータに `imageUrl: '/images/game-name.png'` のように設定する
- 画像未設定のゲームは `GameIcon`（SVGプレースホルダー）がフォールバック表示される
- 新規ゲーム追加時の手順:
  1. 1:1のパッケージ画像を用意
  2. `public/images/` に配置
  3. `data.ts` に新ゲームオブジェクトを追加（`imageUrl` にパスを指定）

### 9.4 実装優先度

1. **必須**: 型定義、ゲームデータ、グリッド表示、検索・タブフィルタ、BootModal
2. **重要**: CSSエフェクト（スキャンライン、グロー、カードホバー）、サウンド
3. **任意**: Firebase Authentication連携、ゲーム実体の実装

---

## 10. 画面フロー

```
[ページ読み込み]
  │
  ├─ Header表示（タイトル、検索、タブ「人気」選択済み）
  ├─ GameCard × 最大12枚 グリッド表示
  └─ Footer表示
       │
       ├─ [検索入力] → リアルタイムフィルタリング
       ├─ [タブ切替] → カテゴリ優先ソート（クリック音）
       └─ [RUN APPLICATIONクリック]（マジック音）
            │
            └─ BootModal表示（起動音）
                 ├─ プログレスバー 0→100%（2秒）
                 └─ [EXECUTE (DEMO)クリック]（マジック音）
                      └─ モーダル閉じる
```

---

## 11. 該当なし画面

検索結果が0件の場合:

- `ERROR 404: NO_DATA_FOUND`（大文字、太字、`opacity-20`）
- `該当するゲームユニットが見つかりません。`（glow-text）
- 「システムリセット」ボタン → 検索クエリをクリア
