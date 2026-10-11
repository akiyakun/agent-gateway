# agent-gateway

家庭内LANの開発PCにあるファイルや開発ツールを、複数のAIクライアント（Strata、Codex、Claude Codeなど）から共通の仕組みで使うためのMCP中継システムです。

- `gateway/` — NASのDockerで動く共通MCPエンドポイント
- `bridge/` — 各開発PCで動き、ローカルのstdio型MCP（公式Filesystem MCPなど）へ橋渡しする
- `shared/` — 両方で使う部品

構想・方針は [docs/overview.md](docs/overview.md) を参照してください（現在は設計前の段階です）。
