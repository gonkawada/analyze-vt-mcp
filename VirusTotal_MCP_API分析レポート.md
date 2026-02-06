# VirusTotal MCP サーバー API分析レポート

## 概要

このレポートは、VirusTotal MCPサーバーにおけるAPI問い合わせの詳細分析を行い、MCPサーバーへの入力値以外のデータ使用とPOSTメソッドの使用について包括的に調査した結果をまとめています。

## 1. 入力値以外のデータ使用分析

### 1.1 環境変数の使用

**VIRUSTOTAL_API_KEY**
- **場所**: `main.py` 16-18行
- **用途**: VirusTotal API認証
- **ソース**: 環境変数 (`os.getenv("VIRUSTOTAL_API_KEY")`)
- **セキュリティ影響**: 高 - APIキーが外部に漏洩するリスク
- **検証**: 起動時に必須チェック実施

```python
API_KEY = os.getenv("VIRUSTOTAL_API_KEY")
if not API_KEY:
    raise ValueError("VIRUSTOTAL_API_KEY environment variable is required")
```

**MCP_TRANSPORT**
- **場所**: `main.py` 423行
- **用途**: トランスポート方式選択（STDIO/SSE）
- **ソース**: 環境変数 (`os.getenv("MCP_TRANSPORT", "sse")`)
- **デフォルト値**: "sse"
- **セキュリティ影響**: 低 - 設定値のみ

### 1.2 ハードコードされた定数

**VT_BASE_URL**
- **値**: `"https://www.virustotal.com/api/v3"`
- **場所**: `main.py` 20行
- **用途**: VirusTotal APIのベースURL
- **変更可能性**: コード修正が必要

**タイムアウト値**
- **値**: `30.0`秒
- **場所**: `main.py` 27行 (`httpx.AsyncClient`の設定)
- **用途**: HTTP リクエストタイムアウト
- **影響**: ネットワーク遅延時の動作に影響

**待機時間**
- **値**: `3`秒
- **場所**: `main.py` 138行 (`await asyncio.sleep(3)`)
- **用途**: URL分析完了待機
- **根拠**: VirusTotal API仕様に基づく推奨値

**ポート番号**
- **値**: `8000`
- **場所**: `main.py` 426行
- **用途**: SSEトランスポートのリスニングポート
- **設定**: ハードコード（環境変数化推奨）

### 1.3 デフォルト関係データ配列

各分析ツールで使用される関係データタイプが事前定義されています：

**URL分析用**
```python
default_rels = [
    "communicating_files",
    "contacted_domains", 
    "contacted_ips",
    "downloaded_files",
    "redirects_to",
    "related_threat_actors",
]
```

**ファイル分析用**
```python
default_rels = [
    "behaviours",
    "dropped_files",
    "contacted_domains",
    "contacted_ips",
    "embedded_urls", 
    "related_threat_actors",
]
```

**IP分析用**
```python
default_rels = [
    "communicating_files",
    "historical_ssl_certificates",
    "resolutions",
    "related_threat_actors",
]
```

**ドメイン分析用**
```python
default_rels = [
    "subdomains",
    "historical_ssl_certificates",
    "resolutions", 
    "related_threat_actors",
]
```

## 2. POSTメソッド詳細分析

### 2.1 POSTメソッド使用箇所

VirusTotal MCPサーバーでは、**URL分析機能のみ**でPOSTメソッドが使用されています。

**使用場所**: `get_url_report`関数（`main.py` 135行）

```python
# Submit URL for scanning
scan_data = await query_virustotal("/urls", "POST", {"url": url})
```

### 2.2 POSTリクエストの詳細

#### 2.2.1 エンドポイント
- **URL**: `https://www.virustotal.com/api/v3/urls`
- **メソッド**: POST
- **目的**: URLスキャンの開始

#### 2.2.2 リクエストデータ

**データソース**: MCPクライアントからの入力
```python
data = {"url": url}  # urlはMCPツール呼び出し時の引数
```

**データ形式**: 
- **Content-Type**: `application/x-www-form-urlencoded`（httpxのデフォルト）
- **ペイロード**: `url=<分析対象URL>`

#### 2.2.3 認証情報

**ヘッダー**: 
```python
headers = {"x-apikey": API_KEY}
```

**APIキーソース**: 環境変数 `VIRUSTOTAL_API_KEY`

#### 2.2.4 レスポンス処理

**期待されるレスポンス構造**:
```json
{
  "data": {
    "type": "analysis",
    "id": "<analysis_id>",
    "links": {
      "self": "https://www.virustotal.com/api/v3/analyses/<analysis_id>"
    }
  }
}
```

**処理フロー**:
1. POSTリクエスト送信
2. `analysis_id`の抽出: `scan_data["data"]["id"]`
3. 3秒待機（分析完了待ち）
4. 分析結果取得（GETリクエスト）

### 2.3 POSTメソッドが使用されない他の機能

以下の機能は全てGETメソッドを使用：

- **ファイル分析** (`get_file_report`): 既存ハッシュの情報取得
- **IP分析** (`get_ip_report`): 既存IPの情報取得  
- **ドメイン分析** (`get_domain_report`): 既存ドメインの情報取得
- **関係データクエリ** (4つの`*_relationship`関数): 既存データの関係情報取得

### 2.4 POSTメソッド使用の理由

**URL分析でPOSTが必要な理由**:
1. **新規スキャン開始**: URLは動的コンテンツのため、リアルタイムスキャンが必要
2. **VirusTotal API仕様**: `/urls`エンドポイントはPOSTでスキャン開始を要求
3. **非冪等性**: 同じURLでも時間により結果が変わる可能性

**他の分析でGETを使用する理由**:
1. **既存データ取得**: ハッシュ、IP、ドメインは既存の分析結果を取得
2. **冪等性**: 同じ入力に対して同じ結果が期待される
3. **キャッシュ効率**: GETリクエストはキャッシュが可能

## 3. セキュリティ考慮事項

### 3.1 機密データの取り扱い

**APIキー**:
- ✅ 環境変数から取得（ハードコード回避）
- ✅ 起動時検証実施
- ⚠️ ログ出力時の漏洩リスク要注意

**分析対象データ**:
- ✅ ユーザー入力のみ使用
- ✅ 外部データソースなし
- ✅ 入力検証実装済み

### 3.2 データ流出リスク

**低リスク要因**:
- 入力値以外の機密データ使用なし
- 環境変数による設定管理
- 適切なエラーハンドリング

**注意点**:
- APIキーの環境変数設定が必須
- ネットワーク通信の暗号化（HTTPS使用）

## 4. 改善提案

### 4.1 設定の外部化

**現在ハードコードされている値の環境変数化**:
```python
# 推奨改善
TIMEOUT = float(os.getenv("VT_TIMEOUT", "30.0"))
SCAN_WAIT_TIME = int(os.getenv("VT_SCAN_WAIT", "3"))
SERVER_PORT = int(os.getenv("MCP_PORT", "8000"))
SERVER_HOST = os.getenv("MCP_HOST", "0.0.0.0")
```

### 4.2 設定検証の強化

**起動時設定検証**:
```python
def validate_config():
    if not API_KEY:
        raise ValueError("VIRUSTOTAL_API_KEY is required")
    if not API_KEY.startswith("vt-"):
        raise ValueError("Invalid VirusTotal API key format")
    # その他の検証...
```

### 4.3 ログ出力の改善

**機密データ除外**:
```python
# APIキーをログから除外
sanitized_headers = {k: "***" if "apikey" in k.lower() else v 
                    for k, v in headers.items()}
```

## 5. まとめ

### 5.1 入力値以外のデータ使用状況

**適切な使用**:
- 環境変数による設定管理
- 必要最小限のハードコード値
- セキュリティを考慮した実装

**改善の余地**:
- 一部設定値の環境変数化
- 設定検証の強化

### 5.2 POSTメソッド使用状況

**適切な使用**:
- URL分析のみでPOST使用
- VirusTotal API仕様に準拠
- 適切なデータ送信

**セキュリティ**:
- 入力データのみ送信
- 機密情報の適切な管理
- HTTPS通信による暗号化

### 5.3 総合評価

VirusTotal MCPサーバーは、入力値以外のデータ使用とPOSTメソッドの使用において、**セキュリティと機能性のバランスが適切に取れた実装**となっています。機密データの取り扱いも適切で、外部データソースへの不正アクセスリスクは最小限に抑えられています。