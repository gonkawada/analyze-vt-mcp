# VirusTotal MCP サーバー

[VirusTotal API](https://www.virustotal.com/)を使用した包括的なセキュリティ分析のためのModel Context Protocol (MCP) サーバーです。FastMCPとPythonで構築されており、ClaudeなどのAIアシスタントに強力なマルウェア検出と脅威インテリジェンス機能を提供します。

## 概要

このMCPサーバーは、VirusTotalの広範なセキュリティデータベースと統合し、AIアシスタントがURL、ファイル、IPアドレス、ドメインに対して包括的なセキュリティ分析を実行できるようにします。サーバーは関係データを自動的に取得し、単一のリクエストで完全なセキュリティコンテキストを提供します。

## 機能

- **包括的なセキュリティ分析**: 関係データの自動取得による完全な脅威分析
- **URL分析**: 接触ドメイン、ダウンロードファイル、脅威アクターを含むセキュリティレポート
- **ファイル分析**: 動作、ドロップファイル、ネットワーク接続を含む詳細なファイルハッシュ分析
- **IP分析**: 地理的位置情報、評判データ、履歴情報
- **ドメイン分析**: DNSレコード、WHOISデータ、SSL証明書、サブドメイン
- **詳細な関係クエリ**: 深い調査のための特定の関係タイプへのページネーション対応アクセス
- **レート制限対応**: VirusTotal APIの制限を遵守
- **複数トランスポート対応**: 異なる統合ニーズに対応するSSE、STDIO、StreamableHTTPトランスポート

## クイックスタート

### 前提条件

- Python 3.8以上 または Docker
- [uv](https://astral.sh/uv) パッケージマネージャー（ローカル開発用）
- VirusTotal APIキー（[こちらから取得](https://www.virustotal.com/gui/my-apikey)）

### インストール

#### オプション1: Docker（推奨）

1. **クローンとセットアップ:**
   ```bash
   git clone https://github.com/barvhaim/virustotal-mcp-server.git
   cd virustotal-mcp-server
   ```

2. **APIキーの設定:**
   ```bash
   echo "VIRUSTOTAL_API_KEY=your_api_key_here" > .env
   ```

3. **Docker Composeで実行:**
   ```bash
   docker-compose up -d
   ```

4. **またはDockerで直接実行:**
   ```bash
   docker build -t virustotal-mcp .
   docker run -d --name virustotal-mcp -p 8000:8000 --env-file .env virustotal-mcp
   ```

#### オプション2: ローカル開発

1. **クローンとセットアップ:**
   ```bash
   git clone https://github.com/barvhaim/virustotal-mcp-server.git
   cd virustotal-mcp-server
   uv sync
   ```

2. **APIキーの設定:**
   ```bash
   echo "VIRUSTOTAL_API_KEY=your_api_key_here" > .env
   ```

3. **サーバーの実行:**
   ```bash
   # SSEトランスポート（Web対応、デフォルト）
   uv run main.py
   
   # STDIOトランスポート（Claude Desktop用）
   MCP_TRANSPORT=stdio uv run main.py
   
   # StreamableHTTPトランスポート（HTTPストリーミング用）
   MCP_TRANSPORT=streamable-http uv run main.py
   
   # カスタムホストとポート（SSEまたはStreamableHTTP用）
   MCP_HOST=127.0.0.1 MCP_PORT=9000 uv run main.py
   ```

## 利用可能なツール

### レポートツール（関係データ自動取得付き）

#### 1. URLレポートツール
- **名前**: `get_url_report`
- **説明**: セキュリティスキャン結果と主要な関係データを含む包括的なURL分析を取得
- **パラメータ**:
  - `url`（必須）: 分析するURL
- **自動取得される関係データ**: 通信ファイル、接触ドメイン/IP、ダウンロードファイル、リダイレクト、脅威アクター

#### 2. ファイルレポートツール
- **名前**: `get_file_report` 
- **説明**: ハッシュ（MD5/SHA-1/SHA-256）を使用した包括的なファイル分析を取得
- **パラメータ**:
  - `hash`（必須）: 分析するファイルハッシュ
- **自動取得される関係データ**: 動作、ドロップファイル、接触ドメイン/IP、埋め込みURL、脅威アクター

#### 3. IPレポートツール
- **名前**: `get_ip_report`
- **説明**: 地理的位置情報と評判を含む包括的なIPアドレス分析を取得
- **パラメータ**:
  - `ip`（必須）: 分析するIPアドレス
- **自動取得される関係データ**: 通信ファイル、履歴SSL証明書、解決履歴、脅威アクター

#### 4. ドメインレポートツール
- **名前**: `get_domain_report`
- **説明**: DNSとWHOISデータを含む包括的なドメイン分析を取得
- **パラメータ**:
  - `domain`（必須）: 分析するドメイン名
  - `relationships`（オプション）: 含める特定の関係データ
- **自動取得される関係データ**: サブドメイン、履歴SSL証明書、解決履歴、脅威アクター

### 関係ツール（詳細分析用）

#### 1. URL関係ツール
- **名前**: `get_url_relationship`
- **説明**: ページネーション対応でURLの特定の関係タイプをクエリ
- **パラメータ**:
  - `url`（必須）: 分析するURL
  - `relationship`（必須）: 関係タイプ（analyses、communicating_files、contacted_domainsなど）
  - `limit`（オプション、1-40、デフォルト: 10）: 結果の数
  - `cursor`（オプション）: ページネーションカーソル

#### 2. ファイル関係ツール
- **名前**: `get_file_relationship`
- **説明**: ページネーション対応でファイルの特定の関係タイプをクエリ
- **パラメータ**:
  - `hash`（必須）: ファイルハッシュ
  - `relationship`（必須）: 関係タイプ（behaviours、dropped_files、contacted_domainsなど）
  - `limit`（オプション、1-40、デフォルト: 10）: 結果の数
  - `cursor`（オプション）: ページネーションカーソル

#### 3. IP関係ツール
- **名前**: `get_ip_relationship`
- **説明**: ページネーション対応でIPの特定の関係タイプをクエリ
- **パラメータ**:
  - `ip`（必須）: IPアドレス
  - `relationship`（必須）: 関係タイプ（communicating_files、resolutionsなど）
  - `limit`（オプション、1-40、デフォルト: 10）: 結果の数
  - `cursor`（オプション）: ページネーションカーソル

#### 4. ドメイン関係ツール
- **名前**: `get_domain_relationship`
- **説明**: ページネーション対応でドメインの特定の関係タイプをクエリ
- **パラメータ**:
  - `domain`（必須）: ドメイン名
  - `relationship`（必須）: 関係タイプ（subdomains、historical_ssl_certificatesなど）
  - `limit`（オプション、1-40、デフォルト: 10）: 結果の数
  - `cursor`（オプション）: ページネーションカーソル

## Claude Desktop統合

このサーバーをClaude Desktopに接続するには、`claude_desktop_config.json`に以下を追加してください：

**設定ファイルの場所:**
- **macOS**: `~/Library/Application Support/Claude/claude_desktop_config.json`
- **Linux**: `~/.config/claude/claude_desktop_config.json`
- **Windows**: `%APPDATA%\\Claude\\claude_desktop_config.json`

```json
{
  "mcpServers": {
    "virustotal": {
      "command": "uv",
      "args": [
        "--directory",
        "/absolute/path/to/virustotal-mcp-server",
        "run",
        "main.py"
      ],
      "env": {
        "VIRUSTOTAL_API_KEY": "your_api_key_here",
        "MCP_TRANSPORT": "stdio"
      }
    }
  }
}
```

## トランスポートオプション

このサーバーは3つのトランスポートモードをサポートしています：

### 1. STDIOトランスポート
- **用途**: Claude Desktop統合、コマンドラインツール
- **設定**: `MCP_TRANSPORT=stdio`を設定
- **通信方式**: 標準入出力ストリーム

### 2. SSEトランスポート（デフォルト）
- **用途**: Webアプリケーション、ブラウザベースのクライアント
- **設定**: `MCP_TRANSPORT=sse`を設定、または未設定
- **通信方式**: HTTP上のServer-Sent Events
- **デフォルトエンドポイント**: `http://0.0.0.0:8000`

### 3. StreamableHTTPトランスポート
- **用途**: HTTPベースのストリーミングアプリケーション、カスタム統合
- **設定**: `MCP_TRANSPORT=streamable-http`を設定
- **通信方式**: チャンク転送エンコーディングを使用したHTTPストリーミング
- **デフォルトエンドポイント**: `http://0.0.0.0:8000`

### 環境変数

- `MCP_TRANSPORT`: トランスポートモード（`stdio`、`sse`、または`streamable-http`）。デフォルト: `sse`
- `MCP_HOST`: SSE/StreamableHTTP用のホストアドレス。デフォルト: `0.0.0.0`
- `MCP_PORT`: SSE/StreamableHTTP用のポート番号。デフォルト: `8000`
- `VIRUSTOTAL_API_KEY`: VirusTotal APIキー（必須）

## リソース

- **FastMCP ドキュメント**: [github.com/jlowin/fastmcp](https://github.com/jlowin/fastmcp)
- **MCP 仕様**: [modelcontextprotocol.io](https://modelcontextprotocol.io)
- **VirusTotal API**: [developers.virustotal.com](https://developers.virustotal.com)
- **uv パッケージマネージャー**: [astral.sh/uv](https://astral.sh/uv)
- **Claude Desktop**: [claude.ai](https://claude.ai)

## バージョン履歴

- **v1.0.0**: 包括的なVirusTotal統合による初回リリース
  - 8つのセキュリティ分析ツール
  - 関係データの自動取得
  - SSE、STDIO、StreamableHTTPトランスポート対応
  - レート制限対応
  - 完全なエラーハンドリング
