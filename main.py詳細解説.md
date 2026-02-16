# main.py 詳細解説ドキュメント

## 概要

このドキュメントは、VirusTotal MCPサーバーのメインファイル `main.py` について、ライン単位での詳細な解説を提供します。このファイルは、VirusTotal APIを使用してセキュリティ分析機能を提供するModel Context Protocol (MCP) サーバーの実装です。

## ファイル構造と全体概要

- **総行数**: 約400行
- **主要機能**: 8つのセキュリティ分析ツール（URL、ファイル、IP、ドメイン分析）
- **使用技術**: FastMCP、httpx、asyncio
- **対応形式**: STDIO、SSEトランスポート

---

## ライン単位詳細解説

### 1-3行: シバン行とドキュメント文字列
```python
#!/usr/bin/env python3
"""VirusTotal MCP Server - VirusTotal Model Context Protocol server."""
```

**解説**:
- **1行目**: Unix系システムでのPython3実行を指定するシバン行
- **2行目**: ファイル全体の説明を提供するドキュメント文字列
- **目的**: スクリプトの実行可能性とドキュメント化

### 4-8行: 必要ライブラリのインポート
```python
import asyncio
import base64
import os
from typing import Any, Dict, List, Optional
```

**解説**:
- **asyncio**: 非同期処理のためのPython標準ライブラリ
- **base64**: URLエンコーディングに使用
- **os**: 環境変数の読み取りに使用
- **typing**: 型ヒントの提供（コードの可読性と保守性向上）

### 9-12行: 外部ライブラリのインポート
```python
import httpx
from dotenv import load_dotenv
from fastmcp import FastMCP
```

**解説**:
- **httpx**: 非同期HTTP クライアントライブラリ（requests の非同期版）
- **dotenv**: .envファイルから環境変数を読み込む
- **FastMCP**: Model Context Protocol サーバーの実装フレームワーク

### 14行: 環境変数の読み込み
```python
load_dotenv()
```

**解説**:
- .envファイルから環境変数を読み込み
- 開発環境での設定管理を簡素化
- 本番環境では通常、システム環境変数を使用

### 16-20行: 設定値の定義
```python
# Configuration
API_KEY = os.getenv("VIRUSTOTAL_API_KEY")
if not API_KEY:
    raise ValueError("VIRUSTOTAL_API_KEY environment variable is required")
```

**解説**:
- **16行目**: 設定セクションのコメント
- **17行目**: 環境変数からVirusTotal APIキーを取得
- **18-19行目**: APIキーが設定されていない場合のエラーハンドリング
- **目的**: 必須設定の検証とアプリケーション起動時の早期エラー検出

### 21行: VirusTotal API ベースURL
```python
VT_BASE_URL = "https://www.virustotal.com/api/v3"
```

**解説**:
- VirusTotal API v3のベースURLを定数として定義
- 設定の一元管理とメンテナンス性の向上

### 23-24行: FastMCPサーバーの初期化
```python
# Initialize FastMCP server
mcp = FastMCP("VirusTotal MCP Server")
```

**解説**:
- MCPサーバーインスタンスの作成
- サーバー名を"VirusTotal MCP Server"として設定
- このインスタンスがツール登録とサーバー実行を管理

### 26-29行: HTTPクライアントの設定
```python
# HTTP client for VirusTotal API
client = httpx.AsyncClient(
    base_url=VT_BASE_URL, headers={"x-apikey": API_KEY}, timeout=30.0
)
```

**解説**:
- **26行目**: HTTPクライアントセクションのコメント
- **27行目**: 非同期HTTPクライアントの作成
- **28行目**: ベースURLとAPIキーヘッダーの設定
- **29行目**: 30秒のタイムアウト設定（ネットワーク問題への対応）

### 32-35行: URL エンコーディング関数
```python
def encode_url_for_vt(url: str) -> str:
    """Encode URL for VirusTotal API."""
    return base64.urlsafe_b64encode(url.encode()).decode().rstrip("=")
```

**解説**:
- **32行目**: 関数定義（URLをVirusTotal API用にエンコード）
- **33行目**: 関数の説明ドキュメント
- **34行目**: 
  - `url.encode()`: 文字列をバイト列に変換
  - `base64.urlsafe_b64encode()`: URL安全なBase64エンコーディング
  - `.decode()`: バイト列を文字列に戻す
  - `.rstrip("=")`: 末尾のパディング文字を削除（VirusTotal API仕様）

### 37-48行: VirusTotal API クエリ関数
```python
async def query_virustotal(
    endpoint: str, method: str = "GET", data: Optional[Dict] = None
) -> Dict[str, Any]:
    """Query VirusTotal API."""
    try:
        if method.upper() == "POST":
            response = await client.post(endpoint, data=data)
        else:
            response = await client.get(endpoint)

        response.raise_for_status()
        return response.json()
    except httpx.HTTPError as exc:
        raise ValueError(f"VirusTotal API error: {str(exc)}") from exc
```

**解説**:
- **37-39行目**: 非同期関数定義（エンドポイント、HTTPメソッド、データを受け取る）
- **40行目**: 関数の説明ドキュメント
- **41行目**: 例外処理の開始
- **42-45行目**: HTTPメソッドに応じたリクエスト送信
  - POST: データ付きでPOSTリクエスト
  - その他: GETリクエスト
- **47行目**: HTTPステータスコードのチェック（4xx、5xxでエラー発生）
- **48行目**: レスポンスをJSON形式で返す
- **49-50行目**: HTTPエラーをValueErrorに変換（統一的なエラーハンドリング）

### 52-69行: 関係項目フォーマット関数
```python
def format_relationship_item(item: Dict[str, Any]) -> str:
    """Format a single relationship item for display."""
    if item.get("type") == "domain":
        return f"  - Domain: {item.get('id', 'Unknown')}"
    if item.get("type") == "ip_address":
        return f"  - IP: {item.get('id', 'Unknown')}"
    if item.get("type") == "file":
        return f"  - File: {item.get('id', 'Unknown')}"
    if item.get("type") == "url":
        return f"  - URL: {item.get('id', 'Unknown')}"

    if "attributes" in item:
        attrs = item["attributes"]
        if "host_name" in attrs and "ip_address" in attrs:
            date_str = attrs.get("date", "unknown date")
            return (
                f"  - {attrs['host_name']} → "
                f"{attrs['ip_address']} (resolved {date_str})"
            )
        if "certificate_id" in attrs:
            cert_id = attrs["certificate_id"]
            validity = attrs.get("validity", {})
            not_before = validity.get("not_before", "unknown")
            not_after = validity.get("not_after", "unknown")
            return f"  - SSL Cert: {cert_id} (valid {not_before} - {not_after})"

        item_type = item.get("type", "Unknown")
        item_id = item.get("id", str(attrs)[:50])
        return f"  - {item_type}: {item_id}"

    item_type = item.get("type", "Unknown")
    item_id = item.get("id", "Unknown")
    return f"  - {item_type}: {item_id}"
```

**解説**:
- **52-53行目**: 関数定義と説明（関係項目を表示用にフォーマット）
- **54-61行目**: 基本的な項目タイプの処理（domain、ip_address、file、url）
- **63-78行目**: 属性情報がある場合の詳細処理
  - **65-69行目**: DNS解決情報の処理
  - **70-75行目**: SSL証明書情報の処理
  - **77-78行目**: その他の属性情報の処理
- **80-82行目**: フォールバック処理（最小限の情報表示）

### 85-118行: スキャン結果フォーマット関数
```python
def format_scan_results(data: Dict[str, Any], scan_type: str) -> str:
    """Format scan results for display."""
    output = [f"# {scan_type.title()} Analysis Report\n"]

    # Basic info
    if "attributes" in data:
        attrs = data["attributes"]
        if "last_analysis_stats" in attrs:
            stats = attrs["last_analysis_stats"]
            output.append("**Detection Summary:**")
            output.append(f"- Malicious: {stats.get('malicious', 0)}")
            output.append(f"- Suspicious: {stats.get('suspicious', 0)}")
            output.append(f"- Clean: {stats.get('harmless', 0)}")
            output.append(f"- Undetected: {stats.get('undetected', 0)}")
            output.append("")

    # Relationships
    if "relationships" in data:
        output.append("**Relationship Data:**")
        for rel_type, rel_data in data["relationships"].items():
            if "data" in rel_data:
                items = rel_data["data"]
                if isinstance(items, list) and len(items) > 0:
                    output.append(
                        f"- {rel_type.replace('_', ' ').title()}: {len(items)} items"
                    )
                    for item in items:
                        output.append(format_relationship_item(item))
                elif items:
                    output.append(f"- {rel_type.replace('_', ' ').title()}: 1 item")
        output.append("")

    return "\n".join(output)
```

**解説**:
- **85-86行目**: 関数定義と説明（スキャン結果を表示用にフォーマット）
- **87行目**: 出力リストの初期化（マークダウンヘッダー付き）
- **89-99行目**: 基本情報の処理
  - **91-92行目**: 属性情報の取得
  - **93-99行目**: 検出統計の表示（悪意のある、疑わしい、クリーン、未検出）
- **101-115行目**: 関係データの処理
  - **102行目**: 関係データセクションのヘッダー
  - **103行目**: 各関係タイプのループ処理
  - **105-112行目**: 関係項目の表示（リスト形式と単一項目の処理）
  - **113行目**: 単一項目の場合の処理
- **117行目**: 全ての出力を改行で結合して返す

### 121-162行: URL レポートツール
```python
@mcp.tool()
async def get_url_report(url: str) -> str:
    """Get comprehensive URL analysis report with security results and relationships.

    Args:
        url: The URL to analyze

    Returns:
        str: Formatted analysis report with detection summary and relationships
    """
    encoded_url = encode_url_for_vt(url)

    # Submit URL for scanning
    scan_data = await query_virustotal("/urls", "POST", {"url": url})
    analysis_id = scan_data["data"]["id"]

    # Wait for analysis
    await asyncio.sleep(3)

    # Get analysis results
    analysis = await query_virustotal(f"/analyses/{analysis_id}")

    # Fetch key relationships
    relationships = {}
    default_rels = [
        "communicating_files",
        "contacted_domains",
        "contacted_ips",
        "downloaded_files",
        "redirects_to",
        "related_threat_actors",
    ]

    for rel_type in default_rels:
        try:
            rel_data = await query_virustotal(f"/urls/{encoded_url}/{rel_type}")
            relationships[rel_type] = rel_data
        except (httpx.HTTPError, KeyError, ValueError):
            continue

    result_data = {
        "attributes": analysis["data"]["attributes"],
        "relationships": relationships,
        "url": url,
    }

    return format_scan_results(result_data, "URL")
```

**解説**:
- **121行目**: MCPツールとしての登録デコレータ
- **122行目**: 非同期関数定義
- **123-129行目**: 関数の詳細ドキュメント（引数、戻り値の説明）
- **131行目**: URLをVirusTotal API用にエンコード
- **133-135行目**: URLスキャンの送信とanalysis IDの取得
- **137-138行目**: 分析完了待機（3秒）
- **140-141行目**: 分析結果の取得
- **143-152行目**: デフォルト関係データの定義
- **154-159行目**: 各関係データの取得（エラー時は継続）
- **161-166行目**: 結果データの構築
- **168行目**: フォーマットされたレポートの返却

### 171-207行: ファイル レポートツール
```python
@mcp.tool()
async def get_file_report(file_hash: str) -> str:
    """Get a comprehensive file analysis report using its hash.

    Args:
        file_hash: MD5, SHA-1 or SHA-256 hash of the file

    Returns:
        str: Formatted analysis report with detection summary and relationships
    """

    # Get file report
    file_data = await query_virustotal(f"/files/{file_hash}")

    # Fetch key relationships
    relationships = {}
    default_rels = [
        "behaviours",
        "dropped_files",
        "contacted_domains",
        "contacted_ips",
        "embedded_urls",
        "related_threat_actors",
    ]

    for rel_type in default_rels:
        try:
            rel_data = await query_virustotal(f"/files/{file_hash}/{rel_type}")
            relationships[rel_type] = rel_data
        except (httpx.HTTPError, KeyError, ValueError):
            continue

    result_data = {
        "attributes": file_data["data"]["attributes"],
        "relationships": relationships,
        "hash": file_hash,
    }

    return format_scan_results(result_data, "File")
```

**解説**:
- **171行目**: MCPツールとしての登録
- **172行目**: 非同期関数定義（ファイルハッシュを受け取る）
- **173-179行目**: 関数ドキュメント（MD5、SHA-1、SHA-256対応を明記）
- **181-182行目**: ファイルレポートの直接取得（URLと異なりスキャン不要）
- **184-193行目**: ファイル固有の関係データ定義
  - behaviours: ファイルの動作
  - dropped_files: ドロップされたファイル
  - embedded_urls: 埋め込まれたURL
- **195-200行目**: 関係データの取得処理
- **202-207行目**: 結果データの構築と返却

### 210-246行: IP レポートツール
```python
@mcp.tool()
async def get_ip_report(ip: str) -> str:
    """Get a comprehensive IP address analysis report.

    Args:
        ip: IP address to analyze

    Returns:
        str: Formatted analysis report with detection summary and relationships
    """

    # Get IP report
    ip_data = await query_virustotal(f"/ip_addresses/{ip}")

    # Fetch key relationships
    relationships = {}
    default_rels = [
        "communicating_files",
        "historical_ssl_certificates",
        "resolutions",
        "related_threat_actors",
    ]

    for rel_type in default_rels:
        try:
            rel_data = await query_virustotal(f"/ip_addresses/{ip}/{rel_type}")
            relationships[rel_type] = rel_data
        except (httpx.HTTPError, KeyError, ValueError):
            continue

    result_data = {
        "attributes": ip_data["data"]["attributes"],
        "relationships": relationships,
        "ip": ip,
    }

    return format_scan_results(result_data, "IP")
```

**解説**:
- **210-218行目**: IP分析ツールの定義とドキュメント
- **220-221行目**: IPアドレスレポートの取得
- **223-230行目**: IP固有の関係データ定義
  - communicating_files: 通信したファイル
  - historical_ssl_certificates: 履歴SSL証明書
  - resolutions: DNS解決履歴
- **232-237行目**: 関係データの取得処理
- **239-244行目**: 結果データの構築と返却

### 249-288行: ドメイン レポートツール
```python
@mcp.tool()
async def get_domain_report(
    domain: str, relationships: Optional[List[str]] = None
) -> str:
    """Get a comprehensive domain analysis report.

    Args:
        domain: Domain name to analyze
        relationships: Optional list of specific relationships to include

    Returns:
        str: Formatted analysis report with detection summary and relationships
    """

    # Get domain report
    domain_data = await query_virustotal(f"/domains/{domain}")

    # Fetch key relationships
    rel_data = {}
    default_rels = relationships or [
        "subdomains",
        "historical_ssl_certificates",
        "resolutions",
        "related_threat_actors",
    ]

    for rel_type in default_rels:
        try:
            rel_response = await query_virustotal(f"/domains/{domain}/{rel_type}")
            rel_data[rel_type] = rel_response
        except (httpx.HTTPError, KeyError, ValueError):
            continue

    result_data = {
        "attributes": domain_data["data"]["attributes"],
        "relationships": rel_data,
        "domain": domain,
    }

    return format_scan_results(result_data, "Domain")
```

**解説**:
- **249-251行目**: ドメイン分析ツールの定義（オプション引数付き）
- **252-259行目**: 関数ドキュメント（関係データフィルタリング機能を説明）
- **261-262行目**: ドメインレポートの取得
- **264-271行目**: 関係データの設定（引数で指定されたものまたはデフォルト）
- **273-278行目**: 関係データの取得処理
- **280-285行目**: 結果データの構築と返却

### 291-320行: URL 関係データクエリツール
```python
@mcp.tool()
async def get_url_relationship(
    url: str, relationship: str, limit: int = 10, cursor: Optional[str] = None
) -> str:
    """Query a specific relationship type for a URL with pagination support.

    Args:
        url: The URL to get relationships for
        relationship: Type of relationship to query
        limit: Maximum number of related objects to retrieve (1-40)
        cursor: Continuation cursor for pagination

    Returns:
        str: Formatted relationship data
    """
    encoded_url = encode_url_for_vt(url)

    params = {"limit": limit}
    if cursor:
        params["cursor"] = cursor

    endpoint = f"/urls/{encoded_url}/{relationship}"
    if params:
        param_str = "&".join([f"{k}={v}" for k, v in params.items()])
        endpoint = f"{endpoint}?{param_str}"

    rel_data = await query_virustotal(endpoint)

    result_data = {"relationships": {relationship: rel_data}, "url": url}

    return format_scan_results(result_data, f"URL {relationship}")
```

**解説**:
- **291-293行目**: URL関係データクエリツールの定義（ページネーション対応）
- **294-303行目**: 詳細ドキュメント（limit範囲とcursor機能を説明）
- **305行目**: URLエンコーディング
- **307-310行目**: クエリパラメータの構築
- **312-315行目**: エンドポイントURLの構築（パラメータ付き）
- **317行目**: 関係データの取得
- **319行目**: 結果データの構築
- **321行目**: フォーマットされた結果の返却

### 324-353行: ファイル 関係データクエリツール
```python
@mcp.tool()
async def get_file_relationship(
    file_hash: str, relationship: str, limit: int = 10, cursor: Optional[str] = None
) -> str:
    """Query a specific relationship type for a file with pagination support.

    Args:
        file_hash: MD5, SHA-1 or SHA-256 hash of the file
        relationship: Type of relationship to query
        limit: Maximum number of related objects to retrieve (1-40)
        cursor: Continuation cursor for pagination

    Returns:
        str: Formatted relationship data
    """

    params = {"limit": limit}
    if cursor:
        params["cursor"] = cursor

    endpoint = f"/files/{file_hash}/{relationship}"
    if params:
        param_str = "&".join([f"{k}={v}" for k, v in params.items()])
        endpoint = f"{endpoint}?{param_str}"

    rel_data = await query_virustotal(endpoint)

    result_data = {"relationships": {relationship: rel_data}, "hash": file_hash}

    return format_scan_results(result_data, f"File {relationship}")
```

**解説**:
- **324-326行目**: ファイル関係データクエリツールの定義
- **327-335行目**: 関数ドキュメント
- **337-340行目**: クエリパラメータの構築
- **342-345行目**: エンドポイントURLの構築
- **347行目**: 関係データの取得
- **349行目**: 結果データの構築（ハッシュ情報付き）
- **351行目**: フォーマットされた結果の返却

### 356-385行: IP 関係データクエリツール
```python
@mcp.tool()
async def get_ip_relationship(
    ip: str, relationship: str, limit: int = 10, cursor: Optional[str] = None
) -> str:
    """Query a specific relationship type for an IP address with pagination support.

    Args:
        ip: IP address to analyze
        relationship: Type of relationship to query
        limit: Maximum number of related objects to retrieve (1-40)
        cursor: Continuation cursor for pagination

    Returns:
        str: Formatted relationship data
    """

    params = {"limit": limit}
    if cursor:
        params["cursor"] = cursor

    endpoint = f"/ip_addresses/{ip}/{relationship}"
    if params:
        param_str = "&".join([f"{k}={v}" for k, v in params.items()])
        endpoint = f"{endpoint}?{param_str}"

    rel_data = await query_virustotal(endpoint)

    result_data = {"relationships": {relationship: rel_data}, "ip": ip}

    return format_scan_results(result_data, f"IP {relationship}")
```

**解説**:
- **356-358行目**: IP関係データクエリツールの定義
- **359-367行目**: 関数ドキュメント
- **369-372行目**: クエリパラメータの構築
- **374-377行目**: エンドポイントURL構築（IP用）
- **379行目**: 関係データの取得
- **381行目**: 結果データの構築（IP情報付き）
- **383行目**: フォーマットされた結果の返却

### 388-417行: ドメイン 関係データクエリツール
```python
@mcp.tool()
async def get_domain_relationship(
    domain: str, relationship: str, limit: int = 10, cursor: Optional[str] = None
) -> str:
    """Query a specific relationship type for a domain with pagination support.

    Args:
        domain: Domain name to analyze
        relationship: Type of relationship to query
        limit: Maximum number of related objects to retrieve (1-40)
        cursor: Continuation cursor for pagination

    Returns:
        str: Formatted relationship data
    """

    params = {"limit": limit}
    if cursor:
        params["cursor"] = cursor

    endpoint = f"/domains/{domain}/{relationship}"
    if params:
        param_str = "&".join([f"{k}={v}" for k, v in params.items()])
        endpoint = f"{endpoint}?{param_str}"

    rel_data = await query_virustotal(endpoint)

    result_data = {"relationships": {relationship: rel_data}, "domain": domain}

    return format_scan_results(result_data, f"Domain {relationship}")
```

**解説**:
- **388-390行目**: ドメイン関係データクエリツールの定義
- **391-399行目**: 関数ドキュメント
- **401-404行目**: クエリパラメータの構築
- **406-409行目**: エンドポイントURL構築（ドメイン用）
- **411行目**: 関係データの取得
- **413行目**: 結果データの構築（ドメイン情報付き）
- **415行目**: フォーマットされた結果の返却

### 420-431行: メイン実行部分
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

**解説**:
- **420行目**: スクリプトが直接実行された場合の処理
- **421行目**: 環境変数からトランスポート方式を取得し小文字に変換（デフォルト: "sse"）
- **422行目**: 環境変数からホストアドレスを取得（デフォルト: "0.0.0.0"）
- **423行目**: 環境変数からポート番号を取得し整数に変換（デフォルト: 8000）
- **425-426行目**: STDIOトランスポートの場合の実行
- **427-428行目**: StreamableHTTPトランスポートの場合の実行（ホスト、ポート指定）
- **429-430行目**: SSEトランスポート（デフォルト）の場合の実行（ホスト、ポート指定）

**トランスポートモードの説明**:
- **STDIO**: 標準入出力を使用した通信（Claude Desktop統合用）
- **SSE**: Server-Sent Eventsを使用したHTTP通信（Webアプリケーション用）
- **StreamableHTTP**: チャンク転送エンコーディングを使用したHTTPストリーミング（カスタム統合用）

---

## 技術的特徴

### 1. 非同期処理の活用
- 全てのAPI呼び出しが非同期で実行
- 複数の関係データを並行取得
- パフォーマンスの最適化

### 2. エラーハンドリング
- 各関係データ取得でのエラー分離
- HTTPエラーの適切な変換
- 部分的失敗での継続処理

### 3. 柔軟な設定管理
- 環境変数による設定
- 複数トランスポート対応（STDIO、SSE、StreamableHTTP）
- デフォルト値の提供
- ホストとポートのカスタマイズ可能

### 4. 構造化されたレスポンス
- マークダウン形式の統一
- 階層的な情報表示
- 読みやすいフォーマット

### 5. 型安全性
- 型ヒントの活用
- Optional型の適切な使用
- 戻り値型の明確化

---

## まとめ

このmain.pyファイルは、VirusTotal APIを活用した包括的なセキュリティ分析機能を提供する、よく設計されたMCPサーバーの実装です。非同期処理、適切なエラーハンドリング、柔軟な設定管理により、堅牢で使いやすいセキュリティ分析ツールを実現しています。