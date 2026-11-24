# niri バックグラウンド操作関連 PR の取り込み可否一覧

確認対象コミット: `398026a8` (`HEAD` / `origin/main`)
確認方法: `niri-ai-background-operation.md` に記載されている各 PR について upstream (`niri-wm/niri`) の head コミットを取得し、`git merge-tree --write-tree` による 3-way マージ判定および `git apply --check` によるパッチ適用判定を実施した (ビルド確認は対象外)

## 1. niri 本体に対する PR の判定結果一覧

| PR | タイトル | upstream 状態 | 判定 | コンフリクト発生ファイル (`git merge-tree`) |
| --- | --- | --- | --- | --- |
| [#3984](https://github.com/niri-wm/niri/pull/3984) | Input: Prevent focus change on virtual keyboards | OPEN | **取り込み可能** | なし (`src/input/mod.rs` にクリーンに適用可能) |
| [#3800](https://github.com/niri-wm/niri/pull/3800) | Add virtual output support | OPEN | **コンフリクトする** | `niri-config/src/output.rs` |
| [#1966](https://github.com/niri-wm/niri/pull/1966) | feat: add remote desktop portal | OPEN | **コンフリクトする** | `src/dbus/mutter_screen_cast.rs`, `src/input/backend_ext.rs`, `src/niri.rs` |
| [#4228](https://github.com/niri-wm/niri/pull/4228) | Add Mutter RemoteDesktop support | CLOSED (Draft) | **コンフリクトする** | `Cargo.toml`, `src/dbus/mod.rs`, `src/dbus/mutter_screen_cast.rs`, `src/niri.rs` |

---

## 2. 各 PR の詳細確認結果

### PR #3984: Input: Prevent focus change on virtual keyboards

- 判定: **取り込み可能**
- URL: https://github.com/niri-wm/niri/pull/3984
- upstream 状態: OPEN (文書記載時点では Draft だったが現在は Ready for review)
- PR head コミット: `65a9c877f0a64659afa2f69ee7fb0e8ec6222668`
- merge-base: `6f1a2c5f0e8274223d4204b1f8d6f7f91538967e` (現在の `HEAD` より 298 コミット前)
- 変更規模: 1 ファイル (`+6` / `-0`)
- 確認結果:
  - `git merge-tree` と `git apply --check` の両方でコンフリクトなく現在のソースへそのまま取り込み可能である
  - 変更対象は `src/input/mod.rs` のポインタクリック時とタッチ開始時の 2 箇所であり、layer-shell サーフェス上の操作で出力フォーカスを変更しないようにする修正である

### PR #3800: Add virtual output support

- 判定: **コンフリクトする**
- URL: https://github.com/niri-wm/niri/pull/3800
- upstream 状態: OPEN
- PR head コミット: `7fd585dd44ed67644977dd9561bb6e8b3837f8f7`
- merge-base: `ee8a04bbaa9a20c53b9544cdd098ff54d8c509b4` (現在の `HEAD` より 191 コミット前)
- 変更規模: 19 ファイル (`+1792` / `-103`)
- 確認結果:
  - 3-way マージ (`git merge-tree`) において `niri-config/src/output.rs` でコンフリクトが発生する
  - 単純パッチ適用 (`git apply --check`) では `niri-config/src/output.rs` に加えて `src/niri.rs` でも適用失敗となる
  - 取り込みには現在の `HEAD` に対する rebase とコンフリクト解消が必要である

### PR #1966: feat: add remote desktop portal

- 判定: **コンフリクトする**
- URL: https://github.com/niri-wm/niri/pull/1966
- upstream 状態: OPEN
- PR head コミット: `4305f1040f16d275947496516728fe5b890a251e`
- merge-base: `d0cd58ec5ac6b2d41d42478fb4f3cc7c0c687db9` (現在の `HEAD` より 159 コミット前)
- 変更規模: 18 ファイル (`+2150` / `-251`)
- 確認結果:
  - 3-way マージ (`git merge-tree`) において以下の 3 ファイルでコンフリクトが発生する
    - `src/dbus/mutter_screen_cast.rs`
    - `src/input/backend_ext.rs`
    - `src/niri.rs`
  - 単純パッチ適用 (`git apply --check`) では上記 3 ファイルに加えて `Cargo.toml` でも適用失敗となる
  - 取り込みには現在の `HEAD` に対する rebase とコンフリクト解消が必要である

### PR #4228: Add Mutter RemoteDesktop support

- 判定: **コンフリクトする**
- URL: https://github.com/niri-wm/niri/pull/4228
- upstream 状態: CLOSED (Draft のままクローズ済み)
- PR head コミット: `dcbf7400854648e1305cde5d8057c8c2029eecbc`
- merge-base: `49fc6117fd6c043adaa2ead316b82db5ed735d36` (現在の `HEAD` より 293 コミット前)
- 変更規模: 20 ファイル (`+3722` / `-58`)
- 確認結果:
  - upstream で既に CLOSED となっているほか、3-way マージ (`git merge-tree`) において以下の 4 ファイルでコンフリクトが発生する
    - `Cargo.toml`
    - `src/dbus/mod.rs`
    - `src/dbus/mutter_screen_cast.rs`
    - `src/niri.rs`
  - 単純パッチ適用 (`git apply --check`) では上記に加えて `Cargo.lock` と `src/handlers/mod.rs` でも適用失敗となる

---

## 3. 参考: 文書内で言及されている外部リポジトリの PR / MR

`niri-ai-background-operation.md` 内で言及されている外部プロジェクトの PR / MR の状況は以下の通りである

- **Smithay PR #2051** (https://github.com/Smithay/smithay/pull/2051 `Ei protocol support using reis`)
  - 状態: **取り込み済み** (Smithay upstream にて `2026-06-16` にマージ済み、マージコミット `71a5d9e0dd79cc0dea9bd99d3b2cdb192bd2a1a2`)
  - 現在の niri の `Cargo.toml` が参照している Smithay リビジョン (`a38bb709a0400759b6a4fab6129e0f1dd72808c3`) はこのマージコミットより 203 コミット進んでおり、既に Smithay 側の EIS 実装を含んでいる (利用時は `Cargo.toml` で Smithay の対応 feature を有効化する必要がある)
- **wlroots MR !2584** (https://gitlab.freedesktop.org/wlroots/wlroots/-/merge_requests/2584)
  - 状態: 別コンポジタライブラリ (`wlroots`) の `ext-transient-seat-v1` 参考実装であり、Smithay ベースである niri のソースに直接取り込む対象ではない
