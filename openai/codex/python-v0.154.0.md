# openai/codex python-v0.154.0 (Python SDK `openai-codex` 0.154.0) 変更内容の解説

- 対象リリース: https://github.com/openai/codex/releases/tag/python-v0.154.0 (リリースタイトル "Python SDK 0.154.0"、タグ作成 2026-09-10 19:46 UTC、公開 2026-09-10 19:51 UTC、PyPI 公開 2026-09-10 19:51 UTC)
- 対象バージョン: python-v0.154.0 (PyPI `openai-codex==0.154.0`、同梱ランタイム `openai-codex-cli-bin==0.154.0`、タグ先コミット `9fd29dfd8c` "Prepare manual Python SDK 0.154.0 release")
- 比較元バージョン: python-v0.147.0 (PyPI `openai-codex==0.147.0`、タグ先コミット `025a88adbd` "Update Python SDK runtime to 0.147.0"、2026-08-18)
- 比較URL: https://github.com/openai/codex/compare/python-v0.147.0...python-v0.154.0
- Python SDK ディレクトリ: https://github.com/openai/codex/tree/python-v0.154.0/sdk/python

## 比較元の決定根拠

- リリースノート本文には Full Changelog（compare）リンクが無いため、タグと PyPI のリリース履歴から決定した
- `git tag -l 'python-v*'` の結果は `python-v0.1.0b1`, `python-v0.1.0b2`, `python-v0.1.0b3`, `python-v0.144.4`, `python-v0.147.0`, `python-v0.154.0` の 6 つ。0.154.0 の直前のタグは python-v0.147.0
- PyPI `openai-codex` の公開履歴も 0.1.0b1 → 0.1.0b2 → 0.1.0b3 → 0.144.4 (2026-07-17) → 0.147.0 (2026-08-18) → 0.154.0 (2026-09-10) で、直前の公開版は 0.147.0
- GitHub Releases 上に存在する Python SDK のリリースは python-v0.154.0 だけ（python-v0.147.0 / python-v0.144.4 はタグのみで、`releases/tags/<tag>` API は 404）。よって「前リリース」はリリース一覧では確認できず、タグ・PyPI に基づく
- Python SDK のバージョン番号は Codex CLI（rust-v）のバージョンに追従しており、0.148〜0.153 の Python SDK は公開されていない。番号の連続性からの推測ではなく上記の履歴による
- python-v0.147.0 は python-v0.154.0 の祖先ではない（0.147.0 は main から分岐したリリースコミット上に打たれている）。merge-base は `bc7a487039` (#39147, 2026-08-18) で、0.147.0 側にだけあるのは "Update PyPI publisher for metadata 2.5" と "Update Python SDK runtime to 0.147.0" の 2 コミット（リリース作業のみ）。範囲全体では 1,158 コミットだが、`sdk/python/` に触れたコミットは 15 件

以上より比較元は python-v0.147.0 で一意に決まる。

## 全体の要点

`sdk/python/` の差分は 35 ファイル (+4,708 / -851)。うち生成コード `generated/v2_all.py` が +2,402 行前後を占める。すべて `@copyberry`（OpenAI 社内からの投影 bot）による PR。以下 `[差分]` は実装差分で確認した事実、`[PR]` は PR 本文、`[RN]` はリリースノート、`[推測]` は推測。

1. **`ExternalMessage`**（#44086）: 他のエージェント・ツール・アプリ由来の「信頼できないコンテンツ」を、ユーザー入力ではなくツール権限（tool-level authority）で投入する新しい入力型。`thread.run()` / `thread.turn()` にそのまま渡せる。アイドル中なら新しいターンを開始、進行中の通常ターンがあればそこに参加（join）する
2. **履歴選択とターン単位オプション**（#44084）: `thread_resume` / `thread_fork` に `include_turns`、`run()` / `turn()` に `turn_service_tier`（このターンだけのサービスティア）と `source`（誰が始めたかのラベル）を追加。省略時の挙動は従来通り
3. **`ReasoningEffort` に `max` と `ultra` を追加**（#39662）
4. **生成モデル・通知の刷新**（#44032, #44564 ほか）: app-server プロトコルの JSON スキーマから型を生成する方式に変更。12 種類の通知が新たに型付き payload を得た。副作用として **`HookMetadata` が RootModel（判別共用体）になり、`hook.command` → `hook.root.command` へのアクセス変更が必要（破壊的）**
5. **ターンのイベント配信モデルの変更**（#44086 → #44400）: TurnHandle ごとに独立した購読（subscription）を持つようになり、「購読を開始した時点」からのイベントを受け取る。過去イベントの再生（replay）はしない。**手で組み立てた／後から参加した TurnHandle の結果は部分的になり得る**（挙動変更）。また turn/start の応答より先に届いた完了イベントを取りこぼさなくなった（バグ修正）
6. **ランタイム互換性チェック**（#44084）: 新オプションを使うとき CLI 0.151.0 未満なら送信前に `CodexError`。`packaging` が新しい依存に追加された
7. **リリース・CI 基盤の整備**（#44053, #44061, #44067, `RELEASING.md`）: 安定版 CLI リリース後に Python パッケージを自動公開するワークフローなど。利用者への直接影響は無し

---

## New Features（リリースノート記載項目）

### 1. `max` と `ultra` の reasoning effort を追加（#39662）

- PR: https://github.com/openai/codex/pull/39662 (+29/-6, 6 files)、コミット `240bbfc14a`（2026-08-20）
- 分類: **新機能**（後方互換）

何が変わったか [差分]:

```python
# sdk/python/src/openai_codex/generated/v2_all.py
class ReasoningEffort(str, Enum):
    none = "none"
    minimal = "minimal"
    low = "low"
    medium = "medium"
    high = "high"
    xhigh = "xhigh"
    max = "max"      # 追加
    ultra = "ultra"  # 追加
```

`examples/13_model_select_and_turn_params/` の `REASONING_RANK` にも `"max": 6, "ultra": 7` が追加された。TypeScript SDK の `ModelReasoningEffort` にも同時に追加されている [PR]。

変更前後: 0.147.0 では `ReasoningEffort("max")` は列挙メンバーに無い。ただし `_missing_` フックにより未知の値も受け付ける設計なので、サーバーが返す `"max"` で例外にはならなかった [差分: `_missing_` の存在]。0.154.0 では `ReasoningEffort.max` / `ReasoningEffort.ultra` として静的に参照できる。

利用者への影響: `thread.run(..., effort=ReasoningEffort.ultra)` のように指定できる。実際にどのモデルが `max`/`ultra` を受け付けるかは SDK ではなくモデル側のカタログに依存する [推測]。既存コードへの影響なし。

### 2. `ExternalMessage` を同期・非同期の `run()` / `turn()` に追加（#44086）

- PR: https://github.com/openai/codex/pull/44086 (+1030/-86, 19 files)、コミット `1a4096e273`（2026-09-09）
- 分類: **新機能**（後方互換。ただし内部のイベント配信が同 PR で書き換えられ、後述 #44400 で再調整）
- ドキュメント: [api-reference.md#externalmessage](https://github.com/openai/codex/blob/python-v0.154.0/sdk/python/docs/api-reference.md#externalmessage)、[FAQ](https://github.com/openai/codex/blob/python-v0.154.0/sdk/python/docs/faq.md)、[examples/16_external_message](https://github.com/openai/codex/tree/python-v0.154.0/sdk/python/examples/16_external_message)

何が変わったか [差分: `_inputs.py`, `api.py`, `__init__.py`]:

```python
@dataclass(slots=True)
class ExternalMessage:
    tool_name: str
    content: str | Sequence[JsonObject | FunctionCallOutputContentItem]
    namespace: str | None = None

RunInput = Input | str | ExternalMessage   # 旧: Input | str
```

- `openai_codex.ExternalMessage` としてトップレベルから import 可能
- `_to_wire_turn_input()` が新設され、`ExternalMessage` は `input=[]` かつ `toolOutput={"name": tool_name, "namespace": namespace, "output": content}`（生成モデル `TurnToolOutput`）として `turn/start` に送られる。`tool_name` が空文字・非文字列なら `ValueError`
- `TurnHandle.steer()` / `AsyncTurnHandle.steer()` の引数型は `RunInput` → `Input | str` に狭められ、`ExternalMessage` を steer には渡せない（ドキュメント曰く、進行中ターンへ外部メッセージを届けるには `thread.turn(message)` を使う）
- `ExternalMessage` はユーザー入力のリストに混ぜられない（単体で `input` 全体として渡す）

挙動 [PR, RN, ドキュメント]:

- コンテンツは「ツール権限」で扱われる。ユーザー指示・developer 指示より下位で、ユーザーの承認や認可を与えるものではない。サンドボックスや承認ポリシーはスレッド側の設定がそのまま効く
- スレッドがアイドルなら新しいターンを開始し、進行中の通常ターンがあればそれに参加する。履歴上は `functionCallOutput` アイテムとして保存される（先行するツール呼び出しや call ID は不要）
- 参加した場合、元の handle と参加した handle はそれぞれ独立にイベントストリームを受け取る（詳細は後述の「ターン購読モデル」）

使い方の例（`examples/16_external_message/sync.py` より）:

```python
from openai_codex import Codex, ExternalMessage, Sandbox

with Codex(config=runtime_config()) as codex:
    thread = codex.thread_start(sandbox=Sandbox.read_only)
    thread.run("When deployment notifications arrive, summarize their status ...")

    result = thread.run(
        ExternalMessage(
            tool_name="notifications",
            namespace="slack",
            content="Staging deployment failed: the health check returned HTTP 503.",
        ),
        source="slack_notification",
    )
```

利用者への影響: 新規 API なので既存コードは変更不要。**CLI 0.151.0 以上が必要**で、`CodexConfig.codex_bin` で古い CLI を指定しているとリクエスト送信前に `CodexError` になる（後述の互換性チェック）。SDK 同梱ランタイム（0.154.0）を使う場合は問題ない。

### 3. resume/fork の `include_turns`、ターン単位の `turn_service_tier`、`source` メタデータ（#44084）

- PR: https://github.com/openai/codex/pull/44084 (+661/-102, 14 files)、コミット `8afccec87a`（2026-09-09）
- 分類: **新機能**（後方互換）

何が変わったか [差分: `api.py`]:

| メソッド | 追加引数 | ワイヤ上のフィールド |
| --- | --- | --- |
| `Codex.thread_resume()` / `AsyncCodex.thread_resume()` | `include_turns: bool \| None = None` | `thread/resume` の `excludeTurns`（`None if include_turns is None else not include_turns` と反転して送る） |
| `Codex.thread_fork()` / `AsyncCodex.thread_fork()` | `include_turns: bool \| None = None` | `thread/fork` の `excludeTurns`（同上） |
| `Thread.run()` / `Thread.turn()`（Async も同様） | `turn_service_tier: str \| None = None` | `turn/start` の `serviceTierForTurn` |
| 同上 | `source: str \| None = None` | `turn/start` の `turnTrigger` |

挙動 [差分: 生成モデルの description, ドキュメント]:

- `include_turns`: サーバー応答の `thread.turns` に履歴を詰めるかどうかを制御する。`False` なら応答に履歴を含めない（モデルのコンテキストからは削除されない）。省略/`None` ならサーバーのデフォルト。`excludeTurns` の説明には「ページネーションされたスレッドでは全履歴のハイドレーションは非推奨。`thread/turns/list` / `thread/items/list` と組み合わせて使う」とある
- `turn_service_tier`: このリクエストで**新たに開始する**ターンだけにサービスティアを上書きする。`"default"` で標準速度。スレッドのティアは変えない。進行中ターンへの参加時は無視される。従来の `service_tier` はスレッドのデフォルトを変更する（このターン以降に効く）
- `source`: 誰がこのターンを開始したかのラベル（例 `"review_ui"`）。メタデータであり、権限付与やスケジューリングはしない。参加時は無視される

また `run()` は同 PR 以降 `turn(...)` を呼んで `turn.run()` に委譲する形に整理され、`run()` と `turn()` のシグネチャは「同時生成」（`# BEGIN GENERATED: Thread.flat_methods` ブロックに `run` も含まれる）されるようになった [差分]。

利用者への影響: 省略時の挙動は従来通り。**新オプションを指定した場合のみ CLI 0.151.0 以上が必要**。`thread_resume(..., include_turns=False)` は大きな履歴を持つスレッドの再開を軽くする用途に使える [推測: `excludeTurns` の description より]。

### 4. 生成プロトコルモデル・通知の更新と、turn-start 応答前に届く完了イベントの保持（#44032, #44400）

- PR: https://github.com/openai/codex/pull/44032 (+2468/-633, 11 files)、コミット `45134c0463`（2026-09-09）
- PR: https://github.com/openai/codex/pull/44400 (+195/-153, 10 files)、コミット `ddea03ad04`（2026-09-10）
- 関連: https://github.com/openai/codex/pull/44564 (+2341/-6, 47 files、コミット `3319d9b296`) — スレッド添付（thread attachments）API の追加で生成モデルも更新。https://github.com/openai/codex/pull/44325 (`0adfc1f2f2`) — アップロード応答の prompt hash 追加
- 分類: **内部変更 + 新機能（型付き通知）+ バグ修正 + 生成モデルの破壊的変更**

#### 4a. スキーマからの型生成方式（内部変更）[PR #44032, 差分]

- これまでは固定バージョンのランタイムバイナリを起動してスキーマを取得していたが、リポジトリ内 `codex-rs/app-server-protocol/schema/json` から生成するようになった（`pyproject.toml` の `[tool.codex.codegen] schema-dir`、`scripts/update_sdk_artifacts.py` の `--schema-dir`）。`just write-app-server-schema` で Rust 側スキーマを更新すると Python 側も更新される
- 便宜 API（`run()` など）のパラメータは明示的な許可リストで管理され、プロトコルに新フィールドが増えても自動的にメソッド引数が増えなくなった
- 利用者への直接影響なし

#### 4b. 型付き通知の追加（新機能）[差分: `generated/notification_registry.py`]

`NOTIFICATION_MODELS` の登録数は 70 → 82。新規に型付き payload を得た通知メソッドは以下の 12 件:

- `autoApprovalReview/strictReviewRequired` → `StrictReviewRequiredNotification`
- `mcpServer/event/stream/notification` → `McpServerEventStreamNotification`
- `modelProvider/authRecoveryStarted` / `modelProvider/authRecoveryCompleted` → `AuthRecoveryNotification`
- `project/changed` → `ProjectChangedNotification`
- `thread/attachment/updated` → `ThreadAttachmentUpdatedNotification`（#44564）
- `thread/project/updated` → `ThreadProjectUpdatedNotification`
- `thread/queue/changed` → `ThreadQueueChangedNotification`
- `thread/realtime/item/started` / `thread/realtime/item/completed` / `thread/realtime/item/transcript/delta`
- `thread/reverted` → `ThreadRevertedNotification`

`KnownNotificationPayload` 型エイリアスがレジストリから導出され、`models.NotificationPayload` は `KnownNotificationPayload | RawResponseItemCompletedNotification | UnknownNotification` に置き換えられた（`models.py` から従来 import できた通知クラス名は re-export で維持）。

変更前後: 0.147.0 ではこれらの通知は `UnknownNotification` として届き、`.params`（dict）を読む必要があった。0.154.0 では `Notification.payload` が対応する pydantic モデルになる。未知のメソッドや検証に失敗した payload は引き続き `UnknownNotification` [ドキュメント]。

利用者への影響（移行 [RN]）: 上記の通知を `.params` で読んでいたコードは、`isinstance(payload, UnknownNotification)` が偽になるため動かなくなる。名前付きフィールドを読むように変更する。

#### 4c. `HookMetadata` が RootModel に（破壊的変更）[差分: `generated/v2_all.py`]

変更前（0.147.0）:

```python
class HookMetadata(BaseModel):
    command: str | None = None
    handler_type: Annotated[HookHandlerType, Field(alias="handlerType")]
    ...
```

変更後（0.154.0）:

```python
class HookMetadata(RootModel[HookMetadata1 | HookMetadata2 | PromptHookMetadata | AgentHookMetadata]):
    root: HookMetadata1 | HookMetadata2 | PromptHookMetadata | AgentHookMetadata
```

- `HookMetadata1`（`handler_type: Literal["command"]`, `command: str`, `async_`）、`HookMetadata2`（`handler_type: Literal["mcpTool"]`, `server`, `tool`）、`PromptHookMetadata`、`AgentHookMetadata` の判別共用体。ハンドラ種別ごとに固有フィールドが必須化された
- 移行 [RN]: `hook.command` → `hook.root.command`。`hook.root.handler_type` を確認してからハンドラ固有フィールドを読む（`command` ハンドラ以外に `.command` は無い）
- 影響範囲: `openai_codex.types` / `generated.v2_all` の `HookMetadata` を直接使っている（hook 一覧 API の応答を読む）コードのみ。便宜 API しか使っていなければ影響なし

#### 4d. その他の生成モデルの変更（参考）[差分: クラス一覧の grep]

スレッド添付（`ThreadAttachment*`、`thread/attachment/add|list|remove`）、`ThreadRevertParams`、`ThreadItemsListParams`、`PluginReconcile*`、Realtime/BEM 関連（`ThreadRealtimeBemItemPresentation` 等）、Guardian の `WriteStdinGuardianApprovalReviewAction`、`ComputerUse*` の macOS/Windows 分割、`GetAccountRateLimitsParams` / `GetAccountTokenUsageParams`、`ResponseUsageMetadata`、`ConfigurationReasoning` などが追加・変更されている。これらは低レベル API（`CodexClient.request()` 等）で app-server を直接叩く場合にだけ関係する。個別の追加・削除クラスの網羅的な確認は未実施（→ 末尾「未調査項目」）。

#### 4e. turn-start 応答前に届いた完了イベントの保持（バグ修正）[差分: `_message_router.py`, `client.py`; PR #44400]

- 0.147.0 の `MessageRouter.route_notification` は、まだ `register_turn` されていない turn_id の `turn/completed` を**破棄**していた（`if notification.method == "turn/completed": pending.pop(...); return`）。`turn/start` の応答より先にターン完了通知が届く（極端に短いターンや即時エラー）と、`TurnHandle.stream()` が完了イベントを受け取れず待ち続ける可能性があった [推測: コードからの読み取り]
- 0.154.0 では `CodexClient._start_turn()` が `router.pending_turn(thread_id)` コンテキスト内で `turn/start` を送り、その時点以降のイベントをすべて `_TurnState.events` に蓄積する。応答後に `prepare_turn()` で handle 用の購読を「リクエスト送信時点のカーソル」から開始するため、応答前に届いた `turn/completed` やトランスポート失敗（`fail_all` で `_pending_turn_requests[thread_id] = exc`）も取りこぼさない
- 利用者への影響: 挙動が正しくなる方向のみ。API 変更なし

---

## 移行時の注意（リリースノート "Check these migrations when upgrading"）

### A. `HookMetadata` は `.root` 経由でアクセス

上記 4c を参照。

### B. 一部の通知が型付きに

上記 4b を参照。

### C. ターン購読モデルの変更（#44086 → #44400）— 手で組み立てた／後から参加した TurnHandle は「参加時点以降」のイベントのみ受け取る

- 分類: **挙動変更（内部再設計に伴うもの）**。`thread.turn()` の戻り値をそのまま使う通常の使い方には影響しない

変更前（0.147.0）[差分]:

- `MessageRouter` は turn_id ごとに 1 本の `queue.Queue` を持ち、`register_turn()` されるまでのイベントを `_pending_turn_notifications` に溜めて、`stream()` 開始時に再生（replay）していた。turn_id ごとに購読者は 1 つ
- `TurnHandle(client, thread_id, id)` を手で作って `stream()` すれば、ターン開始直後から溜まっていたイベントを（`turn/completed` を除き）受け取れた

中間状態（#44086 時点、リリースには含まれない）[PR, `9fd29dfd8c` での api-reference.md の書き換え]: 参加した handle には完了済みアイテムと最新 usage を再生し、両 handle が完全な結果を得る設計だった。#44400 でこの再生を取り除いた。

変更後（0.154.0）[差分: `_message_router.py`, `api.py`]:

- `_TurnState` がイベントを連番で保持し、`_TurnSubscription` が購読者ごとのカーソルを持つ。全購読者が読み終えたイベントは `_prune_turn_events` で解放される。購読は `weakref.finalize` で GC 時にも解放される
- `thread.turn(...)` が返す handle: `turn/start` **リクエスト送信時点**からのイベントを受け取る（4e の仕組み）
- それ以外（`TurnHandle(client, thread_id, turn_id)` を直接構築、`ExternalMessage` で進行中ターンに参加した handle、低レベル `register_turn_notifications()`）: 購読を作った時点以降のイベントのみ。以前のイベントは再生されないので、`run()` で集めた `TurnResult` は部分的になり得る。保存済み履歴は `thread.read(include_turns=True)` で取得する
- ターン完了後に handle を作って `next()` すると `TransportClosedError("Turn is no longer streaming")` が送出される
- `AsyncCodexClient.turn_start` は専用の `ThreadPoolExecutor`（`codex-turn-start`）で実行され、awaiting 側がキャンセルされた場合は完了後に購読を閉じてリークを防ぐ

利用者への影響: `TurnHandle` を手動構築して後追いストリーミングしていたコードは結果が欠ける可能性がある。同じターンを複数の消費者で読む用途は、`thread.turn()` の戻り値を共有するか、`thread.read(include_turns=True)` で補完する。

### D. ランタイム互換性チェック（`CodexError` の新しい発生条件）[差分: `_runtime_requirements.py`, `client.py`]

- `MINIMUM_RUNTIME_VERSION = "0.151.0"`。`CodexClient.request()` は `turn/start` の `toolOutput` / `turnTrigger` / `serviceTierForTurn`、`thread/resume` / `thread/fork` の `excludeTurns` が非 None のとき、`initialize` 応答の `serverInfo.version`（無ければ `userAgent` から抽出）を `packaging.version.Version` で比較し、不足なら送信前に `CodexError("turn/start with toolOutput: Codex CLI 0.151.0 or newer is required; ... Configure CodexConfig.codex_bin with a supported CLI.")` を投げる
- ローカルビルドなどバージョンが `0.0.0` の CLI は、`codex generate-json-schema --experimental` を一度だけ実行して `TurnStartParams` 等のスキーマにフィールドがあるかを遅延検査する（`CheckoutCapabilities`、`launch_args_override` 指定時は検査不能としてエラー）
- CLI の alpha 表記 `0.154.0-alpha.1.2` は PEP 440 の `0.154.0a1.post2` に正規化して比較
- 利用者への影響: `pydantic` に加えて **`packaging>=26.2` が依存に追加**。新オプションを使わない限りチェックは走らないので、既存コードは古い `codex_bin` でも従来通り動く

---

## リリースノート外の変更

### E. リリース・CI 基盤（#44053, #44061, #44067）— 利用者への直接影響なし

- https://github.com/openai/codex/pull/44053: Python SDK テストを TypeScript SDK と同じ Bazel ビルドの CLI に対して実行。`CODEX_EXEC_PATH` → ローカル debug ビルド → インストール済みランタイムの順で探索。wheel を新規環境にインストールして動かすスモークテスト（`tests/installed_sdk_smoke.py`）を追加
- https://github.com/openai/codex/pull/44061: SDK のビルドをランタイム公開の前に行い、片方だけ公開される事故を防ぐ。`stage-sdk --codex-version` でランタイム依存を SDK バージョンと独立に指定可能
- https://github.com/openai/codex/pull/44067: 安定版 CLI リリース（`release` ジョブ成功、prerelease でない）を起点に Python SDK とランタイムを同じバージョンで自動公開するダウンストリームワークフロー。`sdk/python/RELEASING.md` を新設し、FAQ / getting-started の記述を「安定版 CLI リリースが SDK を同バージョンで公開する。CLI の prerelease では Python パッケージは公開されない。独立した SDK beta は手動で別番号を付けて公開できる」に更新
- 0.154.0 自身はタグメッセージとコミットメッセージから「manual release」（手動リリース）として作られている [差分: `9fd29dfd8c`]。この手動リリースコミットでは `openai-codex-cli-bin` の pin を 0.153.4 → 0.154.0 に上げ、api-reference.md の「参加した handle」の説明を #44400 の挙動に合わせて修正している

### F. `release_version.py`（内部）

`normalize_codex_version()` により Codex のリリースタグ（`rust-v0.154.0` 形式）も受け付けるようになった。CI 用スクリプトで利用者影響なし [差分, PR #44061]。

### G. コードフォーマット（#42109, #41925）

リポジトリ全体の Python スクリプト整形・Rust フォーマッタ探索のテスト。`sdk/python` 配下にも触れているが動作変更なし。

---

## 変更分類まとめ

| 分類 | 項目 |
| --- | --- |
| 破壊的変更 | `HookMetadata` の RootModel 化（`.root` 経由に）— 生成型を直接使う場合のみ。`TurnHandle.steer()` の引数型が `Input \| str` に（`ExternalMessage` 不可、型ヒント上の変更） |
| 挙動変更 | 手動構築／後追い TurnHandle は参加時点以降のイベントのみ（再生なし）。完了後の購読は `TransportClosedError`。以前 `UnknownNotification` だった 12 通知が型付きに |
| 新機能 | `ExternalMessage`、`include_turns`、`turn_service_tier`、`source`、`ReasoningEffort.max` / `ultra`、型付き通知（thread attachments、realtime、project、queue、revert、auth recovery、strict review、MCP event stream） |
| バグ修正 | turn/start 応答前の完了イベント／トランスポート失敗の取りこぼし防止。async turn_start キャンセル時の購読リーク防止 |
| 非推奨化 | SDK API 上の明示的な非推奨化は無し。プロトコル側の `excludeTurns` の説明で「ページネーションされたスレッドの全履歴ハイドレーションは非推奨」と記載 |
| 内部変更 | スキーマからの型生成、ランタイム互換性チェック機構、リリース/CI ワークフロー、`packaging` 依存追加、CLI pin 0.147.0 → 0.154.0 |

## 未調査項目

- `generated/v2_all.py` の追加・削除クラスの網羅的な一覧（本ノートでは主要なものを抜粋。`ThreadRealtime*`、`PluginShare*` など一部は削除→再追加として diff に現れており、実質的な変更かどうかは個別確認していない）
- `ExternalMessage` 参加時の app-server（Rust）側の挙動（`toolOutput` を進行中ターンへどう合流させるか）は Python SDK 側の差分・ドキュメントのみに基づく
- `max` / `ultra` を実際に受け付けるモデルの一覧
