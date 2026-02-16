# StreamableHTTP トランスポート機能追加

## 概要

VirusTotal MCPサーバーに、既存のSTDIOとSSEトランスポートに加えて、StreamableHTTPトランスポートのサポートを追加しました。

## ブランチ情報

- **ブランチ名**: `feature/streamable-http-transport`
- **ベースブランチ**: `main`
- **コミットID**: `2bc7b91`

## 変更内容

### 1. コード変更（main.py）

#### 追加された機能

**環境変数の追加**:
- `MCP_HOST`: ホストアドレスの設定（デフォルト: `0.0.0.0`）
- `MCP_PORT`: ポート番号の設定（デフォルト: `8000`）

**トランスポート選択ロジックの改善**:
```python
if __name__ == "__main__":
    transport = os.getenv("MCP_TRANSPORT", "sse").lower()
    host = os.getenv("MCP_HOST", "0.0.0.0")
    port = int(os.getenv("MCP_PORT", "8000"))
    
    if transport == "stdio":
        mcp.run(transport="stdio")
    elif transport == "streamable-http":
        mcp.run(transport="streamable-http", host=host, port=port)
    else:  # Default to SSE
        mcp.run(transport="sse", host=host, port=port)
```

**主な改善点**:
- トランスポート名の大文字小文字を区別しない（`.lower()`）
- ホストとポートを環境変数から取得可能
- StreamableHTTPトランスポートの追加
- より明確な条件分岐

### 2. ドキュメント更新

#### README.md
- 機能一覧にStreamableHTTP追加
- インストール手順にStreamableHTTP実行例追加
- トランスポートオプションセクション追加
  - 3つのトランスポートモードの詳細説明
  - 環境変数の説明
- バージョン履歴更新

#### README_JP.md
- README.mdと同様の内容を日本語で追加
- トランスポートオプションの詳細説明（日本語）
- 環境変数の説明（日本語）

#### main.py詳細解説.md
- メイン実行部分の解説を更新（420-431行）
- 新しいトランスポート選択ロジックの詳細説明
- 3つのトランスポートモードの説明追加
- 技術的特徴セクションの更新

#### .kiro/specs/virustotal-mcp-server/design.md
- アーキテクチャの特徴を更新（3つのトランスポート対応）
- 設定項目に`MCP_HOST`と`MCP_PORT`を追加

#### .kiro/specs/virustotal-mcp-server/requirements.md
- 要件8（複数トランスポート対応）を更新
- StreamableHTTPトランスポートの受入基準追加
- ホストとポート設定の受入基準追加

#### VirusTotal_MCP_API分析レポート.md
- 環境変数セクションに`MCP_HOST`と`MCP_PORT`追加
- ハードコード値セクションを更新（環境変数化済み）
- 設定の外部化セクションを更新

## トランスポートモード比較

### 1. STDIO トランスポート
- **用途**: Claude Desktop統合、コマンドラインツール
- **通信方式**: 標準入出力ストリーム
- **設定**: `MCP_TRANSPORT=stdio`
- **ネットワーク**: 不要

### 2. SSE トランスポート（デフォルト）
- **用途**: Webアプリケーション、ブラウザベースのクライアント
- **通信方式**: Server-Sent Events over HTTP
- **設定**: `MCP_TRANSPORT=sse` または未設定
- **エンドポイント**: `http://{MCP_HOST}:{MCP_PORT}`
- **デフォルト**: `http://0.0.0.0:8000`

### 3. StreamableHTTP トランスポート（新規追加）
- **用途**: HTTPベースのストリーミングアプリケーション、カスタム統合
- **通信方式**: HTTP streaming with chunked transfer encoding
- **設定**: `MCP_TRANSPORT=streamable-http`
- **エンドポイント**: `http://{MCP_HOST}:{MCP_PORT}`
- **デフォルト**: `http://0.0.0.0:8000`

## 使用例

### 基本的な使用

```bash
# SSEトランスポート（デフォルト）
uv run main.py

# STDIOトランスポート
MCP_TRANSPORT=stdio uv run main.py

# StreamableHTTPトランスポート
MCP_TRANSPORT=streamable-http uv run main.py
```

### カスタム設定

```bash
# カスタムホストとポート
MCP_HOST=127.0.0.1 MCP_PORT=9000 uv run main.py

# StreamableHTTPでカスタム設定
MCP_TRANSPORT=streamable-http MCP_HOST=localhost MCP_PORT=3000 uv run main.py

# 大文字小文字を区別しない
MCP_TRANSPORT=STREAMABLE-HTTP uv run main.py
```

### Docker環境での使用

```bash
# docker-compose.ymlに環境変数を追加
environment:
  - VIRUSTOTAL_API_KEY=${VIRUSTOTAL_API_KEY}
  - MCP_TRANSPORT=streamable-http
  - MCP_HOST=0.0.0.0
  - MCP_PORT=8000
```

## 環境変数一覧

| 変数名 | 説明 | デフォルト値 | 必須 |
|--------|------|--------------|------|
| `VIRUSTOTAL_API_KEY` | VirusTotal APIキー | なし | ✅ |
| `MCP_TRANSPORT` | トランスポートモード | `sse` | ❌ |
| `MCP_HOST` | ホストアドレス | `0.0.0.0` | ❌ |
| `MCP_PORT` | ポート番号 | `8000` | ❌ |

**MCP_TRANSPORT の有効な値**:
- `stdio`: 標準入出力
- `sse`: Server-Sent Events（デフォルト）
- `streamable-http`: HTTPストリーミング

## 互換性

### 後方互換性
- ✅ 既存の設定（STDIO、SSE）は変更なしで動作
- ✅ デフォルト動作は変更なし（SSEトランスポート）
- ✅ 既存の環境変数（`VIRUSTOTAL_API_KEY`、`MCP_TRANSPORT`）は引き続き使用可能

### 新機能
- ✅ StreamableHTTPトランスポートの追加
- ✅ ホストとポートのカスタマイズ
- ✅ 大文字小文字を区別しないトランスポート名

## テスト方法

### 1. STDIOトランスポートのテスト
```bash
MCP_TRANSPORT=stdio uv run main.py
# 標準入出力で通信を確認
```

### 2. SSEトランスポートのテスト
```bash
uv run main.py
# ブラウザで http://localhost:8000 にアクセス
```

### 3. StreamableHTTPトランスポートのテスト
```bash
MCP_TRANSPORT=streamable-http uv run main.py
# HTTPクライアントで http://localhost:8000 にアクセス
```

### 4. カスタム設定のテスト
```bash
MCP_TRANSPORT=streamable-http MCP_HOST=127.0.0.1 MCP_PORT=9000 uv run main.py
# HTTPクライアントで http://127.0.0.1:9000 にアクセス
```

## 今後の改善案

1. **設定検証の強化**
   - トランスポート名の検証
   - ホストアドレスの形式検証
   - ポート番号の範囲検証

2. **ログ機能の追加**
   - 起動時のトランスポート情報表示
   - 接続情報のログ出力

3. **ヘルスチェックエンドポイント**
   - HTTP系トランスポート用のヘルスチェック
   - `/health` エンドポイントの追加

4. **ドキュメントの拡充**
   - StreamableHTTPの詳細な使用例
   - トラブルシューティングガイド

## まとめ

StreamableHTTPトランスポートの追加により、VirusTotal MCPサーバーはより柔軟な統合オプションを提供できるようになりました。既存の機能との互換性を保ちながら、新しいユースケースに対応できる拡張性の高い実装となっています。

すべてのドキュメントが更新され、日本語ドキュメントも完全に対応しています。
