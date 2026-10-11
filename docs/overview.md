# agent-gateway / agent-bridge 構想概要書

- 初版: 2026-10-10（ChatGPTと整理した初期案）
- 改訂: 2026-10-10（Claude Codeとのレビューを反映）
- 状態: 構想・方針整理。詳細設計・実装仕様は未確定
- 用途: Claude Codeで詳細設計・実装を進めるときの共有資料


## 1. 目的

複数のAIクライアント（Strata、Codex、Claude Codeなど）から、家庭内LANにある開発PCのファイル操作やテスト・ビルドなどを共通の仕組みで使えるようにする。

AIが動くPCと、ソースコードや開発ツールを持つPCを分離する。

将来の複数PC対応、MCPの追加、Wake-on-LAN（WoL）、未使用時の自動スリープを見据え、通信経路とコンポーネントの役割は初期段階から整理しておく。ただし初期実装は最小限にとどめ、将来の機能を先取りして複雑にしない。


## 2. 全体構成

```text
Strata / Codex / Claude Code など
        │
        │ MCP over HTTP（Bearer認証）
        ▼
NAS（Docker、常時稼働）
  agent-gateway
        │
        │ HTTP（MVPでは MCP HTTP をそのまま中継。Bearer認証）
        ▼
開発PC A（Windows。将来は Mac も）
  agent-bridge  ← ツールの許可リストによるフィルタはここ
        │
        │ stdio
        ▼
  公式 Filesystem MCP
        │
        ▼
  許可されたローカルのソースコード
```

将来は開発PC B、Cにもそれぞれagent-bridgeを置く。

AIクライアントからは原則agent-gatewayだけを接続先として見せる。クライアントごとに各PCや個別MCPの接続設定をたくさん持たせない。


## 3. コンポーネントの役割

### 3.1 agent-gateway（NAS上）

- AIクライアント向けの共通MCPエンドポイントを提供する。
- Docker上で動かす。既存のNAS MCP（vision-mcp / websearch-mcp）と同じ規約（`mcp-conventions`）に合わせる。
- 将来は、接続先PCの選択、状態確認、WoL、アイドル判定、スリープ制御を担う。
- 将来は、クライアントごとの権限（例: Strataは読み取りのみ、Claude Codeは書き込みも可）によるツールフィルタも担う。

**段階的な役割（決定事項）**

| 段階 | 方式 | 内容 |
|---|---|---|
| MVP | **A. 透過プロキシ** | MCP HTTPの要求をPC Aのbridgeへそのまま中継する。MCPの中身は解釈しない |
| WoL・複数PCに着手する時点 | **B. 集約型** | gateway自身がMCPサーバーとして振る舞い、各bridgeに対してはMCPクライアントになる |

B（集約型）でないと実現できないこと:

- PCがスリープ中でも、`initialize` と `tools/list` にgatewayが（保持しておいたツール定義で）応答し、`tools/call` が来たときだけPCを起こす。
- 複数PCのツールを、名前に接頭辞を付けるなどして1つの一覧にまとめる。
- クライアントごとのツールフィルタ。

**AからBへの切り替えでクライアント側の設定を変えない。** AIクライアントに設定するURL（例: `http://192.168.0.100:8100/mcp`）とトークンは、Bに切り替えても同じものを使えるようにする。

Bを一から作り込む前に、MCP SDKのプロキシ機能や既存のMCPゲートウェイ製品で賄えないか、切り替えの時点で調べる。

### 3.2 agent-bridge（各開発PC上）

- 開発PCに置く軽量なサービス。
- gatewayからの要求を受け、ローカルのstdio型MCPサーバーへ橋渡しする。
- MVPでは公式Filesystem MCPだけを起動・公開する。
- 将来はShell MCP、Git、Unity関連ツールなどを足せるように、MCPごとに設定を分けておく。
- 設定で、MCPごとに有効・無効を切り替えられるようにする。
- 最初はWindowsで動かす。将来はMacにも対応する（OSの違いはbridge内で吸収する）。
- Windowsで手軽にインストール・起動できることを重視する。将来は、必要なMCPだけを選んで入れられるワンクリックインストールを目指す。

**ツールフィルタ（決定事項）**

Filesystem MCPのツールのフィルタは、守る対象（PCのファイル）に一番近いbridgeで行う。gatewayに不具合・設定ミス・侵害があっても、bridgeが最後の砦として拒否できるようにするため。bridgeに直接つないで試すときにも制限が効く。

- **許可リスト方式**にする。Filesystem MCPを更新して新しいツール（例: 削除系）が増えても、許可していなければ自動的に使えない。
- **「隠す」と「拒否する」の両方**を行う。
  - `tools/list` の応答から、許可していないツールを消す。
  - `tools/call` で許可していないツールが呼ばれたらエラーを返す。
- この処理のためにbridgeはMCPメッセージの中身を読む必要がある。つまりbridgeは「完全な素通し」ではない。
- フィルタの処理は `shared/` に置き、将来gateway（B）のクライアント別フィルタでも同じものを使う。

設定のイメージ（形式は設計時に確定）:

```yaml
servers:
  filesystem:
    enabled: true
    allowed_dirs:
      - D:\Projects\MyGame
    allowed_tools:
      - list_directory
      - read_text_file
      - search_files
```

### 3.3 公式Filesystem MCP

- `@modelcontextprotocol/server-filesystem` を使う。
- ファイルの一覧・読み込み・書き込み・編集・検索を担当する。
- アクセスできるフォルダを明示的に限定する。
- 公式パッケージは改造しない。bridgeとは独立して更新できるようにする。
- `npx @latest` は使わず、**バージョンを固定**する。更新するときは、増減したツールを確認してから許可リストを見直す。

### 3.4 Shell MCP（将来追加）

- 候補は sonirico/mcp-shell。
- コマンド実行、テスト、ビルド、Gitの状態確認などを担当させたい。
- Filesystem MCPとは分ける。
- 任意のコマンドを実行できるのはFilesystemより強い権限なので、許可コマンド・作業ディレクトリ・認証・実行制限を別途設計する。
- 初期実装の対象外。


## 4. 初期実装のスコープ（MVP）

MVP（Minimum Viable Product）= 目的を果たせることを確かめられる、最小限の機能だけを持った最初の版。

実装するもの:

1. NASのDockerでagent-gatewayを起動できる。
2. Windowsの開発PC Aでagent-bridgeを簡単に起動できる。
3. agent-bridgeが公式Filesystem MCPを起動し、stdioとの通信を仲介できる。
4. agent-bridgeがツールを許可リストでフィルタできる（MVPは**読み取り系のツールだけ**を許可）。
5. StrataがNASのagent-gatewayへHTTPで接続できる。
6. agent-gatewayが開発PC Aへ要求を転送できる（透過プロキシ）。
7. StrataからFilesystem MCPのツールを認識し、許可フォルダの一覧取得・ファイル読み込みを確認できる。
8. 接続先URL、認証情報、許可フォルダ、許可ツールなどをコードを変えずに設定できる。
9. LAN内でも最低限の認証・アクセス制限を用意する（5章）。

後回しにするもの:

- 書き込み・編集ツールの解禁（フェーズ5）
- gatewayの集約型（B）への切り替え
- 複数開発PCの切り替え
- WoLによる起床、アイドル時のスリープ・休止
- Shell MCP、Git MCP、Unity関連MCP
- Mac版bridge
- Codex / Claude Codeとの正式な接続テスト
- 本格的なインストーラーGUI
- 複雑なワークフロー自動化、AIによるタスクの振り分け

最初の成功条件:

```text
Strata → agent-gateway → agent-bridge → 公式Filesystem MCP
```

という経路で、開発PC Aの許可済みフォルダについて、ファイル一覧の取得と読み込みが動くこと。


## 5. セキュリティ方針

### 5.1 利用範囲

- **当面は家庭内LAN専用**とする。インターネットへの公開や、外出先からの接続は扱わない。
- 必要になったら、そのときにTailscale（端末同士を同じLANのようにつなぐVPNサービス）などを検討する。

### 5.2 認証（2層）

| 区間 | 方式 |
|---|---|
| AIクライアント → gateway | Bearerトークン（例: `AGENT_GATEWAY_TOKEN`）。既存のNAS MCPと同じ方式 |
| gateway → bridge | 上とは**別の**Bearerトークン（例: `AGENT_BRIDGE_TOKEN`） |

- トークンは32文字以上。MCPごと・区間ごとに別の値にする。
- 秘密は `.env` などGit管理外のファイルに置き、`.env.example` を用意する。

### 5.3 アクセス制限

- bridgeの待ち受けポートは、Windowsファイアウォールで**NASのIP（192.168.0.100）からの接続だけ**を許可する。LAN全体に無防備に公開しない。
- gatewayはmcp-conventionsに従い、DNSリバインディング対策（Host / Originの検証）、リクエストサイズの上限、`MCP_BIND_IP` によるバインド先の制限を行う。
- bridgeにも同等のHost検証とサイズ上限を入れる。

### 5.4 許可フォルダ内の秘密情報

許可フォルダの中に `.env`、APIキー、`.git` 内の認証情報などがあると、AIがそれを読み、クラウドのAI（Codex / Claudeなど）に送ってしまうことがあり得る。これはClaude CodeやCodexをPC上で直接使う場合も同じ。

- **運用ルール: 許可フォルダには秘密情報を置かない。** APIキーなどはOSのキーチェーンやパスワードマネージャーに置き、リポジトリには値の入っていない `.env.example` だけを置く。
- 公式Filesystem MCPには除外の機能がない。bridgeでパスを除外する仕組みは、手間の割に抜けが出やすいのでMVPでは作らない。


## 6. 設計上の重要事項

- **MCP over HTTPは単なるJSONの横流しでは成立しないことがある。** 初期化、セッション（`Mcp-Session-Id` ヘッダ）、SSEによるストリーミング、切断、エラー処理の互換性を確認する。
- 透過プロキシ（A）でよくハマる点: SSE応答のバッファリング（無効にする）、`Mcp-Session-Id` などのヘッダの転送、タイムアウトの設定。
- mcp-conventionsでは `stateless_http=True` / `json_response=True` を使っている。gatewayとbridgeの間もこれに合わせられれば、セッション管理とストリーミングの問題を大きく減らせる。成り立つかどうかは設計時に確認する。
- **Strataのツール呼び出しは既定で60秒でタイムアウトする。** 中継による遅延や、将来のWoL起床待ち（数十秒〜1分以上かかり得る）はこの制約の中に収める必要がある。起床が間に合わない場合は「起動中なので少し待ってから再度呼ぶ」といった応答を返す方式を、B方式の設計時に検討する。
- 可能な限り、既存のMCP SDKや実績のあるライブラリを使い、プロトコルを独自に実装しすぎない。
- gatewayとbridgeは分ける。Filesystem MCPのソースは改造しない。
- コマンド実行を足すときはFilesystem以上の権限になるので、別途安全設計を行う。
- 複数PCが参加するときの権限の境界と接続先の選び方は、重要な設計課題（Bへの切り替え時に決める）。

### 6.1 WoL・スリープの検証項目（将来）

- NASのDockerコンテナからはブロードキャストが届かないことが多い。`network_mode: host` にするか、サブネット宛てのブロードキャストを使う。
- Windows側は「高速スタートアップ」の無効化と、NIC（LANアダプタ）のWoL設定が必要。
- Modern Standby（S0ix）のPCはWoLが不安定になりやすい。
- 基本的に有線LANが前提。
- スリープ・休止から復帰できるか、安定しているかは実機で確かめる。


## 7. リポジトリ構成

**モノレポ（1つのリポジトリにまとめる）に決定。** リポジトリ: `github.com/akiyakun/agent-gateway`

理由:

- フィルタ処理などの共通部品を `shared/` に1つだけ置き、gatewayとbridgeの両方から使える。
- 通信の仕様を変えるときに、gatewayとbridgeを1回のコミットで直せる。
- 一人で開発するので管理が楽。Claude Codeが両方のコードを同時に見て作業できる。
- Mac対応は `bridge/` の中で吸収すればよく、リポジトリを分ける理由にならない。

構成案（細部は設計時に確定）:

```text
agent-gateway/
├── gateway/   NAS Docker用（Dockerfile / docker-compose.yaml はここ）
├── bridge/    Windows / Mac用
├── shared/    両方で使う部品（ツールフィルタなど）
└── docs/      この概要書・設計書
```

- gateway部分は `mcp-conventions` の規約（Dockerの堅牢化、テストの方針など）に従う。ホスト側ポートは **8100**（mcp-conventionsのポート表に登録済み）。
- **実装言語はgateway / bridgeともPython（`mcp` SDK）に決定。** 理由:
  - `shared/` を両方で使うので、同じ言語にする必要がある。
  - 処理は中継とフィルタだけで、言語による速度差は問題にならない。
  - 既存のNAS MCPの規約・テストの型・SDKの実績をそのまま使える。
  - ユーザーが読めるので、Claude Codeが書いたコードを自分で確認できる。
  - 職場でもエンジニア以外を含めて広く使われており、Windowsへのインストールにも慣れている。
- bridgeのPCには、Filesystem MCPを動かすためのNode.jsも別に必要になる。


## 8. これから決めること（未確定）

| 項目 | 現時点の方向性 |
|---|---|
| gateway↔bridge間の通信方式 | MVPは MCP HTTP の透過中継（A）。細部は設計時 |
| bridgeのstdio↔HTTP変換 | MCP SDK を使い、フィルタを挟める形で実装する方向。既存ツールの流用可否は要調査 |
| 実装言語 | **決定: gateway / bridge とも Python**（7章） |
| 認証方式・秘密の管理 | 2層のBearer + `.env`（5章）。Windowsでの保管方法は要検討 |
| ポート番号・URLパス | gatewayはホスト側8100、`/mcp`。bridgeのポートは未定 |
| bridgeの起動方法 | タスクスケジューラの「ログオン時に実行」が有力。Windowsサービスはセッション0・別ユーザー環境で動くため、将来のUnityやGUI系ツールで問題が出やすい |
| Filesystem MCPのバージョン管理・更新方法 | バージョンを固定する。更新手順は要検討 |
| 書き込みの解禁時期 | MVPは読み取りのみ。フェーズ5で解禁 |
| セッション管理と再接続 | stateless にできるかを含めて設計時に確認 |
| 接続先PCの識別子・ルーティング | Bへの切り替え時に決める |

上記は、実装時に現行のMCP仕様と各製品の対応状況を確認して決める。

### Windows特有の注意点（bridge）

- Windowsでは `npx` を子プロセスとして起動するときに `cmd /c` が必要になることがある。
- bridgeを止めたときに子プロセス（node）が残らないよう、プロセスツリーごと終了させる。
- 許可フォルダのパス表記（`D:\...` と `D:/...`）の扱いを確認する。


## 9. 開発の進め方

まず通信経路が端から端まで通ることを優先する。

| フェーズ | 内容 |
|---|---|
| 1 | Windowsの開発PC Aで、agent-bridge + 公式Filesystem MCPを動かす。読み取り系ツールだけを許可するフィルタも入れる。MCP Inspectorで動作を確認する |
| 2 | **Strataからbridgeへ直接つなぎ**、ファイル一覧と読み込みを確認する（問題がbridge側かgateway側かを切り分けるため） |
| 3 | NASのDockerに最小のagent-gateway（透過プロキシ）を立て、PC Aへ中継する |
| 4 | Strataの接続先をgatewayに切り替え、ファイル一覧と読み込みを確認する（**MVP完成**） |
| 5 | 書き込み・編集の解禁、安全性の見直し、再接続、bridgeの起動方法（自動起動）を整える |
| 6以降 | 必要に応じて、Shell MCP、Mac版bridge、gatewayの集約型（B）への切り替え、複数PC、WoL・自動スリープ、Codex / Claude Codeとの連携を足す |

これは段階案で、正式な順序は設計時に調整してよい。


## 10. 将来の拡張構想

### 10.1 複数の開発PC

- PC A / B / Cにそれぞれagent-bridgeを置く。
- gatewayを集約型（B）にし、接続先PCの管理機能を足す。
- ツールの公開名（PCごとの接頭辞など）、接続先の識別方法、複数PCの競合の扱いは、その時点で設計する。

### 10.2 省電力運用

- gatewayが利用対象PCの状態を確認し、必要ならWoLで起こす。
- bridgeの準備ができるのを待ってから要求を中継する。
- 一定時間使われなければスリープ・休止に移る。
- PCがスリープ中だとMCPの初期化が失敗し得るので、gatewayがツール定義を保持して代わりに応答する（3.1）。
- 起床待ちとStrataの60秒タイムアウトとの兼ね合いを考える（6章）。

### 10.3 コーディングエージェントとの連携

- クラス設計や難しい実装には、必要に応じてCodex / Claude Codeを使う。
- ローカルのStrataは、軽い作業や判断に使う。
- 開発PC側のFilesystem / Shell / Unityなどを通じて、ファイル操作、コンパイル、テスト、結果の確認を行う。
- 将来は、複数のAIクライアントが同じagent-gatewayを使える状態を目指す。

### 10.4 その他のMCPとの共存

- NASでは別にwebsearch-mcpやvision-mcpなどを運用している。
- agent-gatewayは当面「開発PCの作業ツールを中継する」役割に絞る。
- 全MCPの統合管理までは初期実装に含めない。


## 11. この概要書の位置づけ

ユーザーとChatGPTで合意した初期案を、Claude Codeとのレビューで改訂した方向性の整理であり、完成した詳細設計書ではない。

次の段階では、この構成を前提に、Claude Codeがリポジトリの詳細構成・通信方式・MVPの設計案を提案し、ユーザーが確認してから実装する。
