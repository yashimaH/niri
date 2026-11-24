# niri ファイル構成と AI バックグラウンド操作対応の検討箇所

本ドキュメントは Wayland コンポジタ `niri` のリポジトリ全体のファイル構成を整理し、AI やリモートからセッションを作成してバックグラウンド操作に対応するために修正または検討が必要になる箇所へコメントを付記したものである
なお、該当するソースコード側にも `// [AI バックグラウンド操作検討]` (TOML ファイルでは `# [AI バックグラウンド操作検討]`) のプレフィックスで対応コメントを追記している

---

## 1. ワークスペース全体構成

niri は Cargo ワークスペースで構成されており、ルートパッケージ (`niri` 本体) と 3 個のサブクレートから成る

```text
niri/
├── Cargo.toml                 # ワークスペース定義および niri 本体クレート設定 [要修正 / 検討]
├── Cargo.lock                 # 依存クレートのロックファイル
├── build.rs                   # ビルドスクリプト (システムライブラリ検出等)
├── rustfmt.toml / clippy.toml / typos.toml # コード整形、静的解析、スペルチェック設定
├── flake.nix / flake.lock     # Nix 開発環境およびパッケージ定義
├── niri.spec.rpkg             # RPM パッケージビルド定義
├── resources/                 # セッション起動スクリプト、systemd / dinit、portal 設定等 [要検討]
├── docs/                      # MkDocs ベースの公式 Wiki ドキュメント一式
├── src/                       # niri コンポジタ本体および CLI のソースコード [主たる修正対象]
├── niri-config/               # KDL 設定ファイルパーサのサブクレート [要修正 / 検討]
├── niri-ipc/                  # IPC 通信プロトコル型定義とクライアントソケットのサブクレート [要修正]
└── niri-visual-tests/         # レイアウトやシェーダ描画の視覚テスト用サブクレート
```

---

## 2. ルートおよび `resources/` / `docs/`

### ルート設定ファイル
- `Cargo.toml`
  - ワークスペースおよび `niri` 本体の依存関係と feature フラグ定義である
  - **[検討コメント (フェーズ 2)]**: 現在ピン留めされている `smithay` リビジョン (`a38bb709`) には `reis` ベースの EIS 実装 (Smithay PR `#2051`) が既に取り込まれているため、EIS 入力サーバを実装する際は `[dependencies.smithay]` に `backend_eis` 等の feature を追加する
  - **[検討コメント (フェーズ 2)]**: `org.gnome.Mutter.RemoteDesktop` 対応を既存の `xdp-gnome-screencast` feature に含めるか、独立した feature として切り出すかを検討する
- `build.rs`
  - ビルド時のネイティブ依存確認処理を行う

### `resources/` (システム統合リソース)
- `resources/niri-session`
  - ログインマネージャ等からメインセッションとして niri を起動するラッパースクリプトである
- `resources/niri.desktop`
  - `/usr/share/wayland-sessions/` に配置される Wayland セッション定義である
- `resources/niri.service` / `resources/niri-shutdown.target`
  - systemd ユーザーサービスとしての起動定義である
  - **[検討コメント (フェーズ 1)]**: AI 専用のヘッドレス niri インスタンスをバックグラウンドサービスとして常駐させる場合、メインの `niri.service` とは独立した unit および環境スコープで起動する設計が必要である
- `resources/dinit/niri` / `resources/dinit/niri.target`
  - dinit サービス定義である
- `resources/niri-portals.conf`
  - `xdg-desktop-portal` に対して使用するバックエンド (`default=gnome;gtk;` 等) を指定する設定ファイルである
  - **[検討コメント (フェーズ 2)]**: `default=gnome;gtk;` により `ScreenCast` と `RemoteDesktop` は `xdg-desktop-portal-gnome` へルーティングされるため、niri 側で `org.gnome.Mutter.RemoteDesktop` を実装すれば既存の portal 構成のまま同意ダイアログ付きの正規ルートが利用できる
- `resources/default-config.kdl`
  - 初回起動時に生成されるデフォルト設定ファイルである
- `resources/mutter-x11-interop.xml`
  - `mutter_x11_interop` Wayland プロトコル XML 定義である
- `resources/cursor.rgba` / `resources/rustdoc-index.html`
  - フォールバックカーソル画像およびドキュメント用 HTML リソースである

### `docs/` (ドキュメント)
- `docs/mkdocs.yaml` / `docs/pyproject.toml` / `docs/hooks/`
  - ドキュメントサイトのビルド構成である
- `docs/wiki/*.md`
  - 機能別および設定項目別の公式ドキュメント群である (`niri-config/tests/wiki-parses.rs` によりコード例のパース検証も行われる)

---

## 3. `src/` (niri コンポジタ本体)

### エントリポイントとコア状態管理
- `src/main.rs`
  - プロセス起動、CLI 引数解析、設定読み込み、イベントループ構築、Wayland / X11 / IPC ソケット公開、D-Bus サーバ起動を行うエントリポイントである
  - **[修正コメント (フェーズ 1)]**: `State::new` 呼び出しの第 5 引数 `headless` が `false` にハードコードされているため、`NIRI_BACKEND=headless` 環境変数や `--headless` フラグからヘッドレスバックエンドを選択できるように修正する必要がある
  - **[検討コメント (フェーズ 1 / フェーズ 2)]**: `--session` 指定時は `WAYLAND_DISPLAY` 等を消去した上で `import_environment()` によりユーザーの systemd と D-Bus 環境変数をグローバルに上書きする
    同一ユーザーで人間のメインセッションと並行して AI 用ヘッドレスインスタンスを起動する場合、メイン環境の上書きや D-Bus 名の競合を起こさないよう、`dbus-run-session` 等によるセッションバス分離または `--session` に依存しない起動モードを整備する必要がある
- `src/lib.rs`
  - クレート直下のモジュール宣言一覧である
  - **[修正コメント (フェーズ 2)]**: `screencasting` と並び、Mutter `RemoteDesktop` および EIS サーバの状態管理モジュール (例: `pub mod remote_desktop;`) を追加する箇所である
- `src/cli.rs`
  - `clap` によるコマンドライン引数 (`niri --session`、`niri msg ...` 等) の定義である
  - **[修正コメント (フェーズ 1)]**: ヘッドレス起動オプションや、仮想出力を動的に追加または削除する IPC サブコマンド (`create-virtual-output` / `remove-virtual-output` 等) を追加する対象である
- `src/niri.rs`
  - コンポジタの中核となる `State` および `Niri` 構造体、Wayland グローバル登録、描画ループと VBlank 制御、出力管理、クライアント管理を集約する中核ファイルである
  - **[修正コメント (フェーズ 1)]**: `State::new` での `Backend::Headless` 初期化時に、テスト時とは異なりレンダラ生成と初期仮想出力 (例: `HEADLESS-1`) の自動追加を行う必要がある
  - **[検討コメント (フェーズ 1 / フェーズ 2)]**: `Niri::new` 内の `client_is_unrestricted` フィルタは `SecurityContext` を持たない全ホストクライアントを許可しており、`virtual_keyboard`、`virtual_pointer`、`screencopy`、`image_copy_capture`、`wlr_data_control`、`ext_data_control` が無許可で利用可能になっている
    フェーズ 2 で EIS + RemoteDesktop portal による許可制へ寄せる際、これら既存プロトコルの制限方針を検討する必要がある
  - **[検討コメント (フェーズ 2)]**: 起動時は `seat.add_keyboard` と `seat.add_pointer` のみを実行し、タッチ capability は物理デバイス接続時に動的追加しているため、EIS 経由でタッチ入力を受ける際の `seat` 公開タイミングを検討する必要がある
  - **[検討コメント (フェーズ 3)]**: `Niri` は単一の `seat: Seat<State>` と単一の `keyboard_focus` を前提に設計されており、マルチシートや `ext-transient-seat-v1` には未対応である
    そのため同一セッション内でフォーカスを奪わずに人間と AI が別ウィンドウを同時操作することはできず、短期は別インスタンス分離が必須となる
- `src/frame_clock.rs`
  - リフレッシュレートおよび VRR に基づく次フレーム提示時刻計算ロジックである
- `src/cursor.rs`
  - XCursor テーマ読み込み、カーソル形状管理、テクスチャキャッシュを担当する
- `src/a11y.rs`
  - AccessKit を用いたアクセシビリティツリー構築と通知を担当する (`dbus` feature 有効時)
- `src/rubber_band.rs`
  - スクロール端などのラバーバンド補間計算ヘルパである

### `src/backend/` (描画および出力バックエンド)
- `src/backend/mod.rs`
  - `Backend` 列挙型 (`Tty` / `Winit` / `Headless`) と共通インターフェースのディスパッチ層である
  - **[修正コメント (フェーズ 1)]**: 仮想出力の動的作成や削除メソッドの追加、および `Headless` バックエンドで PipeWire dmabuf 転送を行う場合の `gbm_device()` 対応 (`None` 固定からの変更) を検討する箇所である
- `src/backend/headless.rs`
  - 物理ディスプレイを持たないヘッドレスバックエンド実装である
  - **[最重要修正コメント (フェーズ 1)]**: 現在は自動テスト用のスタブに近い実装であり、AI 専用ヘッドレスセッションとして実用化するために以下の実装が必要である
    1. `init` または起動処理での `GlesRenderer` 初期化とデフォルト仮想出力 (例: `1920x1080@60`) の作成
    2. DRM レンダーノードと GBM アロケータの導入による `primary_render_node()` および `import_dmabuf()` (`unimplemented!()` の解消) の実装
    3. `render()` における実オフスクリーン描画と、クライアント接続状況に依存しない固定リフレッシュタイマでのフレーム進行
    4. IPC や設定変更 (`on_output_config_changed`) に連動した仮想出力の追加、解像度変更、削除処理
- `src/backend/tty.rs`
  - `libseat`、`udev`、`libinput`、DRM / KMS、マルチ GPU (`GpuManager`) を用いた実機 TTY バックエンドである
  - **[検討コメント (フェーズ 1)]**: メインの物理モニタセッション上に仮想出力を追加する場合 (PR `#3800` 方式) は、物理 DRM コネクタ管理とは独立したオフスクリーン仮想出力の管理機構をここへ追加する (オフ状態の仮想出力削除時のクラッシュ防止に注意する)
    ただし同一セッション上の仮想出力は人間と seat やフォーカスを共有するため、AI のバックグラウンド操作用途では別プロセスの `Headless` インスタンス分離を優先する
- `src/backend/winit.rs`
  - 既存の Wayland / X11 セッション上でネストされたウィンドウとして動作する開発およびテスト用バックエンドである

### `src/dbus/` (D-Bus インターフェース群)
- `src/dbus/mod.rs`
  - 各種 D-Bus サーバの起動管理 (`DBusServers::start`) とバス名取得 (`request_name`) を行う
  - **[修正コメント (フェーズ 2)]**: `org.gnome.Mutter.RemoteDesktop` サーバの起動とイベントループ連携 (`conn_remote_desktop`) を追加する箇所である
  - **[検討コメント (フェーズ 1 / フェーズ 2)]**: `request_name` が `ReplaceExisting` フラグを使用しているため、同一ユーザーの D-Bus セッション上で複数の niri インスタンスが D-Bus サーバを起動するとバス名の奪い合いが発生する点に注意が必要である
- `src/dbus/mutter_screen_cast.rs`
  - `xdg-desktop-portal-gnome` 向けに `org.gnome.Mutter.ScreenCast` D-Bus API を提供する
  - **[最重要修正コメント (フェーズ 2)]**: `ScreenCast::create_session` 内で現在 `remote-desktop-session-id` プロパティをエラーとして拒否している箇所を修正し、`RemoteDesktop` セッションとの紐付けを実装する必要がある
  - **[検討コメント (フェーズ 2 / フェーズ 3)]**: キャスト対象の出力またはウィンドウの論理座標から EIS の `EI region` を動的に生成して対応付ける必要がある (特定の出力名をハードコードしないこと、および仮想出力向けに `RecordVirtual` をサポートするかを検討する)
- `src/dbus/mutter_service_channel.rs`
  - portal バックエンドプロセスへ特権 Wayland ソケットを発行する `org.gnome.Mutter.ServiceChannel` 実装である
  - **[検討コメント (フェーズ 2)]**: RemoteDesktop やクリップボード連携時に追加の `service_client_type` 許可が必要か確認する箇所である
- `src/dbus/mutter_display_config.rs`
  - GNOME 設定パネル等と連携する `org.gnome.Mutter.DisplayConfig` 実装である
- `src/dbus/gnome_shell.rs` / `src/dbus/gnome_shell_introspect.rs` / `src/dbus/gnome_shell_screenshot.rs`
  - GlobalShortcuts portal、ウィンドウ一覧 Introspect、スクリーンショット用の GNOME Shell 互換 D-Bus API 群である
- `src/dbus/freedesktop_a11y.rs` / `src/dbus/freedesktop_locale1.rs` / `src/dbus/freedesktop_login1.rs` / `src/dbus/freedesktop_screensaver.rs`
  - アクセシビリティキーボード監視、ロケール XKB 取得、logind スリープやロック連携、スクリーンセーバ抑止の D-Bus 実装群である
  - **[検討コメント (フェーズ 1)]**: `freedesktop_login1` はシステムのサスペンド復帰シグナル等を購読するため、ヘッドレス別インスタンス動作時に不要な電源キー抑止や副作用が起きないか確認する (`--session` 非指定時は電源キー抑止は行われない)

### `src/screencasting/` (PipeWire 映像配信)
- `src/screencasting/mod.rs`
  - `mutter_screen_cast` からの要求を受けて出力やウィンドウ単位のフレーム合成と再描画制御を行う
  - **[検討コメント (フェーズ 1 / フェーズ 2)]**: `prepare_pw_cast` は `backend.gbm_device()` が `None` の場合に SHM バッファのみを提供する
    `Headless` バックエンドでのパフォーマンス要件に応じて GBM dmabuf 対応、および物理 VBlank が存在しない出力でのフレーム送出タイミング制御を検討する
- `src/screencasting/pw_utils.rs` / `src/screencasting/pw_utils/shm_mapping.rs`
  - PipeWire メインループ統合、ストリームネゴシエーション、dmabuf / SHM バッファへのレンダリング書き出し実装である

### `src/input/` (入力イベント処理とグラブ制御)
- `src/input/mod.rs`
  - キーボード、ポインタ、タッチ、タブレット、ジェスチャの全入力イベント処理とキーバインド発火を行う中核モジュールである
  - **[修正コメント (フェーズ 2)]**: EIS バックエンド (`reis` / Smithay `libei`) から届く入力イベントを `process_input_event` 等へ接続し、keysym 入力やスクロール単位変換を実装する箇所である
  - **[修正コメント (フェーズ 2)]**: `on_device_added` / `on_device_removed` では `self.niri.touch` が `libinput` デバイスのみを追跡しているため、EIS タッチデバイス追加時に `seat.add_touch()` / `seat.remove_touch()` の整合が崩れないよう修正が必要である
  - **[検討コメント (フェーズ 3)]**:
    - `on_keyboard`: 仮想キーボードや EIS デバイスからのキー注入時に、コンポジタのキーバインド (ワークスペース切り替え等) を発火させるかクライアントへの転送のみに限定するか、デバイスごとのポリシー制御を検討する
    - `on_pointer_button` / `on_touch_down`: 仮想キーボード等の layer-shell サーフェスをクリックまたはタップした際に出力フォーカスが移動してしまう問題 (PR `#3984` 該当箇所) の対策箇所である
    - `compute_absolute_location`: マルチモニタや仮想出力構成で絶対座標ポインタやタッチ位置がずれないよう、`NiriInputDevice::output` と EI region の対応付けを行う箇所である
    - `process_input_event`: メインセッションで人間と AI が共存する場合、「AI 操作中」の UI 表示や、人間が物理入力した際の AI 注入一時停止といった競合制御を入れるゲートポイントとなる
- `src/input/backend_ext.rs`
  - 各種入力デバイスを抽象化して所属 `Output` を返す `NiriInputBackend` / `NiriInputDevice` トレイト定義である
  - **[修正コメント (フェーズ 2)]**: EIS 入力デバイス型に対して `NiriInputDevice` を実装し、`ScreenCast` セッションや EI region に対応する `Output` を動的に返すようにする箇所である
- `src/input/click_grab.rs` / `move_grab.rs` / `resize_grab.rs` / `spatial_movement_grab.rs` / `touch_overview_grab.rs` / `pick_window_grab.rs` / `pick_color_grab.rs`
  - クリック、ウィンドウ移動やリサイズ、空間スクロール、タッチ Overview、ウィンドウや色選択の各ポインタ / タッチグラブ実装である
- `src/input/scroll_tracker.rs` / `scroll_swipe_gesture.rs` / `swipe_tracker.rs`
  - 離散 / 連続スクロール蓄積およびマルチタッチスワイプ追跡ロジックである

### `src/handlers/` (Smithay Wayland ハンドラ実装)
- `src/handlers/mod.rs`
  - `SeatHandler`、`SelectionHandler`、`DataDeviceHandler`、`WlrDataControlHandler`、`ExtDataControlHandler`、`VirtualPointerHandler`、`SecurityContextHandler`、`ScreencopyHandler` 等の Smithay ハンドラ実装を集約する
  - **[検討コメント (フェーズ 1 / フェーズ 2)]**: `VirtualPointerHandler` はフェーズ 1 の暫定ポインタ注入経路 (`zwlr_virtual_pointer_v1`) であり、確認ダイアログなしで `process_input_event` へ直結している
  - **[修正コメント (フェーズ 2)]**: Mutter `RemoteDesktop` のクリップボード同期 (`Selection` API) を実装する場合、`SelectionHandler` / `DataDeviceHandler` まわりでリモートセッションとのクリップボード橋渡しを実装する箇所となる
- `src/handlers/virtual_keyboard.rs`
  - `zwp_virtual_keyboard_v1` のイベント処理、クライアント固有 keymap の一時適用と復元、IME パススルー処理を担当する
  - **[検討コメント (フェーズ 1 / フェーズ 2 / フェーズ 3)]**: フェーズ 1 の暫定キーボード注入経路である
    単一 seat の XKB 状態を共有して keymap を差し替えるため、デバイスの生成や破棄に伴うフォーカス揺れや modifier 状態の残留課題があり、EIS で keysym / keymap を扱う際にも同様の配慮が必要である
- `src/handlers/image_copy_capture.rs`
  - 標準プロトコル `ext-image-copy-capture-v1` / `ext-image-capture-source-v1` による出力およびウィンドウキャプチャ実装である
  - **[検討コメント (フェーズ 1)]**: `Headless` バックエンド実用化時に、オフスクリーン描画結果をキャプチャセッションへ供給できるよう連携を確認する箇所である
- `src/handlers/compositor.rs` / `xdg_shell.rs` / `layer_shell.rs` / `background_effect.rs`
  - `wl_compositor` サーフェスコミット、`xdg_shell` ウィンドウ、`wlr_layer_shell` レイヤサーフェス、背景ブラー効果の各ハンドラ実装である

### `src/protocols/` (独自および拡張 Wayland プロトコル実装)
- `src/protocols/mod.rs` / `src/protocols/raw.rs`
  - プロトコルモジュール束ねおよび `wayland-scanner` によるプロトコルコード生成定義である
- `src/protocols/screencopy.rs`
  - `zwlr_screencopy_manager_v1` プロトコル実装である
  - **[検討コメント (フェーズ 1)]**: フェーズ 1 で `wayvnc` 等がヘッドレス niri から画面フレームを取得する主要経路であり、screencopy 接続の有無にかかわらずヘッドレス出力の合成更新が健全に動作するよう `backend/headless.rs` および `niri.rs` 側と連携させる
- `src/protocols/output_management.rs`
  - `zwlr_output_management_v1` による出力設定参照および変更プロトコル実装である
- `src/protocols/foreign_toplevel.rs` / `src/protocols/ext_workspace.rs`
  - タスクバーやスイッチャ向けのトップレベルウィンドウ管理およびワークスペース状態公開プロトコル実装である
- `src/protocols/gamma_control.rs` / `src/protocols/mutter_x11_interop.rs`
  - ガンマ制御 (`zwlr_gamma_control_manager_v1`) および Mutter X11 interop スタブ実装である

### `src/ipc/` (IPC クライアントおよびサーバ処理)
- `src/ipc/mod.rs` / `src/ipc/client.rs` / `src/ipc/server.rs`
  - `niri msg` コマンド側の送信処理 (`client.rs`) と、コンポジタ側の Unix ドメインソケット待受およびリクエスト実行処理 (`server.rs`) である
  - **[修正コメント (フェーズ 1)]**: `IpcServer::start` は `niri.<WAYLAND_DISPLAY>.<PID>.sock` というインスタンス固有ソケットを作成するため、AI エージェントは `NIRI_SOCKET` を AI 用ヘッドレスインスタンスに向けるだけで既存の全ウィンドウ操作アクションを安全に利用できる
    これに加えて、仮想出力の動的作成や削除を扱う IPC リクエスト処理を `server.rs` と `client.rs` に追加する

### `src/layout/` (スクローラブルタイリングとワークスペース管理)
- `src/layout/mod.rs` / `monitor.rs` / `workspace.rs` / `scrolling.rs` / `floating.rs` / `tile.rs`
  - モニタ単位のワークスペース管理、水平スクロールカラム配置、フローティング配置、タイル装飾を実装するレイアウトエンジンである
  - **[検討コメント (フェーズ 1 / フェーズ 3)]**: 出力の追加や削除時のワークスペース自動退避と復帰ロジック (`layout/mod.rs`) を備えているため、仮想出力を動的に追加や削除した際のワークスペース移動挙動を確認する
- `src/layout/focus_ring.rs` / `shadow.rs` / `tab_indicator.rs` / `insert_hint_element.rs` / `opening_window.rs` / `closing_window.rs` / `tests.rs` / `tests/`
  - フォーカスリング、ドロップシャドウ、タブ表示、挿入ヒント、開閉アニメーション、およびレイアウト単体テスト群である

### `src/window/` および `src/layer/` (ウィンドウとレイヤ状態)
- `src/window/mod.rs` / `mapped.rs` / `unmapped.rs`
  - ウィンドウルール解決、初回 configure 前の `Unmapped` 状態、マップ後の `Mapped` ウィンドウ状態を管理する
- `src/layer/mod.rs` / `mapped.rs`
  - `wlr-layer-shell` サーフェスのルール解決と配置や描画要素化を管理する

### `src/render_helpers/` (OpenGL ES レンダリング補助とシェーダ)
- `src/render_helpers/mod.rs` / `renderer.rs` / `offscreen.rs` / `memory.rs` / `texture.rs` / `surface.rs` / `snapshot.rs` / `damage.rs` / `render_elements.rs` / `primary_gpu_texture.rs`
  - GLES レンダラ抽象化、SHM / テクスチャへのオフスクリーン描画 (`render_to_shm` / `render_to_texture` 等)、ダメージ追跡を行う
  - **[検討コメント (フェーズ 1)]**: `Headless` バックエンドで実オフスクリーン描画を実装する際、本モジュール内のオフスクリーン描画ヘルパを再利用できる
- `src/render_helpers/blur.rs` / `background_effect.rs` / `border.rs` / `shadow.rs` / `clipped_surface.rs` / `effect_buffer.rs` / `framebuffer_effect.rs` / `gradient_fade_texture.rs` / `resize.rs` / `resources.rs` / `shader_element.rs` / `solid_color.rs` / `xray.rs` / `debug.rs` / `shaders/`
  - ブラー、角丸クリップ、枠線、影、開閉やリサイズ時のカスタム GLSL シェーダ群である

### `src/ui/` (コンポジタ内蔵オーバーレイ UI)
- `src/ui/mod.rs` / `hotkey_overlay.rs` / `exit_confirm_dialog.rs` / `config_error_notification.rs` / `screenshot_ui.rs` / `mru.rs` / `screen_transition.rs`
  - 起動時ホットキー案内、終了確認、設定エラー通知、インタラクティブスクリーンショット選択、MRU ウィンドウスイッチャ (`Alt+Tab`)、画面遷移エフェクトである
  - **[検討コメント (フェーズ 1 / フェーズ 3)]**: AI 専用ヘッドレスインスタンスでは起動時に `hotkey_overlay` が出ないよう設定 (`hotkey-overlay { skip-at-startup }`) するか、ヘッドレス時の扱いを配慮すると画面キャプチャへの映り込みを防げる
    また将来メインセッション共存時に「AI 操作中」インジケータを表示する場合は `src/ui/` にオーバーレイ要素を追加する設計が自然である

### `src/utils/` (各種ユーティリティ)
- `src/utils/mod.rs` / `id.rs` / `region.rs` / `scale.rs` / `signals.rs` / `spawning.rs` / `transaction.rs` / `vblank_throttle.rs` / `watcher.rs` / `xwayland/`
  - ID 採番 (`CastSessionId` 等)、座標やスケール計算、シグナル処理、子プロセス起動 (`WAYLAND_DISPLAY` / `DISPLAY` 受け渡し)、設定ファイル監視、`xwayland-satellite` 連携を行う
  - **[検討コメント (フェーズ 1)]**: `spawning.rs` はグローバルな `CHILD_ENV` / `CHILD_DISPLAY` を参照してアプリを起動するが、別プロセスとして分離された AI 用 niri インスタンスであればプロセス空間が独立しているため干渉しない

### `src/tests/` (統合テストスイート)
- `src/tests/mod.rs` / `fixture.rs` / `server.rs` / `client.rs` / `test_input_backend.rs` / `virtual_pointer.rs` / `remove_output.rs` / `binds.rs` / `animations.rs` / `floating.rs` / `fullscreen.rs` / `layer_shell.rs` / `transactions.rs` / `window_opening.rs`
  - 現在の `Backend::Headless` を利用して Wayland クライアントとコンポジタを結合検証する自動テスト群である
  - **[検討コメント (フェーズ 1 / フェーズ 2)]**: `Headless` バックエンドを実運用向けに改修する際は既存テスト (`fixture.rs` / `server.rs`) を壊さないように維持しつつ、仮想出力や EIS 入力の e2e テストをここへ追加できる (`virtual_pointer.rs` や `remove_output.rs` が参考になる)

---

## 4. サブクレート群 (`niri-config` / `niri-ipc` / `niri-visual-tests`)

### `niri-config/` (設定ファイルパーサ)
- `niri-config/Cargo.toml`
- `niri-config/src/lib.rs` / `macros.rs` / `error.rs` / `utils.rs` / `utils/merge_with.rs`
  - `knuffel` を用いた KDL 設定ファイルのデコード基盤および `include` マージ処理である
- `niri-config/src/output.rs`
  - `output` セクションの設定構造体定義である
  - **[修正コメント (フェーズ 1)]**: 設定ファイルから仮想出力を宣言的に定義可能にする場合に拡張対象となる (PR `#3800` でも変更されておりコンフリクトが発生しやすい箇所である)
- `niri-config/src/debug.rs`
  - `debug` セクションの設定構造体定義である
  - **[検討コメント (フェーズ 1 / フェーズ 2)]**: `dbus-interfaces-in-non-session-instances` フラグが定義されており、`--session` なしで起動したインスタンスで D-Bus サービスを有効化する挙動に関係する
- `niri-config/src/input.rs` / `binds.rs` / `layout.rs` / `window_rule.rs` / `layer_rule.rs` / `workspace.rs` / `appearance.rs` / `animations.rs` / `gestures.rs` / `misc.rs` / `recent_windows.rs`
  - 入力デバイス設定、キーバインド、レイアウト、ウィンドウ / レイヤルール、ワークスペース、外観、アニメーション、ジェスチャ、クリップボード等の各種設定定義である
  - **[検討コメント (フェーズ 3)]**: 将来的に仮想入力や EIS 入力に対するキーバインド許可ポリシー等を設定可能にする場合は `input.rs` または新規設定項目への追加を検討する
- `niri-config/tests/wiki-parses.rs`
  - `docs/wiki/` 内の KDL 設定サンプルがすべて正常にパースできるかを検証するテストである

### `niri-ipc/` (IPC プロトコル定義)
- `niri-ipc/Cargo.toml` / `niri-ipc/README.md`
- `niri-ipc/src/lib.rs`
  - IPC の `Request`、`Response`、`Action`、`OutputAction`、`Event`、`Output`、`Window`、`Workspace` 等の JSON シリアライズ型定義である
  - **[修正コメント (フェーズ 1)]**: 仮想出力の作成や削除 (`CreateVirtualOutput` / `RemoveVirtualOutput` 等) を追加する場合に `Request` や `Response` を拡張する箇所である
    また `niri_ipc::Output` 構造体のフィールド変更 (`max_bpc` 等) が発生しやすいため、古い PR を取り込む際は型の整合確認が必要である
- `niri-ipc/src/socket.rs`
  - 環境変数 `NIRI_SOCKET` からソケットパスを解決して送受信する同期クライアントヘルパである
- `niri-ipc/src/state.rs`
  - `EventStream` から受信したイベント列を畳み込んでクライアント側で現在のコンポジタ状態を復元するステートマシンである

### `niri-visual-tests/` (視覚確認用テストアプリ)
- `niri-visual-tests/Cargo.toml` / `README.md` / `resources/style.css`
- `niri-visual-tests/src/main.rs` / `smithay_view.rs` / `test_window.rs` / `cases/*.rs`
  - GTK / Smithay を用いてウィンドウ開閉やグラデーション、タイル描画を手動および視覚的に検証するスタンドアロンツールであり、バックグラウンド操作対応での直接の修正は不要である

---

## 5. バックグラウンド操作対応のフェーズ別修正箇所まとめ

| フェーズ | 目的 | 主な修正および検討対象ファイル |
| --- | --- | --- |
| フェーズ 1 (短期) | AI 専用ヘッドレスセッション分離と仮想出力対応 | `src/main.rs`, `src/cli.rs`, `src/niri.rs`, `src/backend/headless.rs`, `src/backend/mod.rs`, `src/backend/tty.rs`, `src/protocols/screencopy.rs`, `src/handlers/image_copy_capture.rs`, `src/ipc/server.rs`, `src/ipc/client.rs`, `niri-ipc/src/lib.rs`, `niri-config/src/output.rs`, `resources/niri.service` |
| フェーズ 2 (中期) | `org.gnome.Mutter.RemoteDesktop` + `libei` / `EIS` 正規ルート実装 | `Cargo.toml`, `src/lib.rs`, `src/dbus/mod.rs`, `src/dbus/mutter_screen_cast.rs`, `src/dbus/mutter_service_channel.rs`, `src/screencasting/mod.rs`, `src/input/mod.rs`, `src/input/backend_ext.rs`, `src/handlers/mod.rs`, `src/handlers/virtual_keyboard.rs` (および新規 `src/dbus/mutter_remote_desktop.rs`, `src/remote_desktop.rs` 等) |
| フェーズ 3 (長期) | メインセッション共存時の `seat` / フォーカス制御と競合対策 | `src/niri.rs`, `src/input/mod.rs`, `src/input/backend_ext.rs`, `src/handlers/virtual_keyboard.rs`, `src/ui/` (`niri-config/src/input.rs`) |
