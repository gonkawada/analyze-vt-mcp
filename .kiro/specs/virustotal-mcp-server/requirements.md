# 要件文書

## 概要

VirusTotal MCPサーバーは、VirusTotal APIを使用してセキュリティ分析機能を提供するModel Context Protocol (MCP) サーバーです。AIアシスタント（Claude等）に対して、URL、ファイル、IPアドレス、ドメインの包括的なセキュリティ分析機能を提供します。

## 用語集

- **MCP_Server**: Model Context Protocolに準拠したサーバー
- **VirusTotal_API**: VirusTotalが提供するセキュリティ分析API
- **Security_Analysis**: マルウェア検出と脅威インテリジェンス分析
- **Relationship_Data**: 分析対象に関連するセキュリティ情報
- **Transport_Layer**: MCPサーバーとクライアント間の通信方式
- **Rate_Limiter**: API呼び出し頻度制限機能

## 要件

### 要件1: URL セキュリティ分析

**ユーザーストーリー:** セキュリティアナリストとして、URLの安全性を分析したいので、包括的なURL分析レポートを取得できるようにしたい。

#### 受入基準

1. WHEN ユーザーがURLを指定してURL分析を要求した場合、THE MCP_Server SHALL VirusTotal_APIを使用してURLをスキャンし、分析結果を返す
2. WHEN URL分析が完了した場合、THE MCP_Server SHALL 検出統計（悪意のある、疑わしい、クリーン、未検出）を含む結果を返す
3. WHEN URL分析を実行する場合、THE MCP_Server SHALL 関連ファイル、接触ドメイン、接触IP、ダウンロードファイル、リダイレクト先、関連脅威アクターの関係データを自動取得する
4. WHEN 無効なURLが提供された場合、THE MCP_Server SHALL 適切なエラーメッセージを返す

### 要件2: ファイル セキュリティ分析

**ユーザーストーリー:** セキュリティアナリストとして、ファイルハッシュを使用してファイルの安全性を分析したいので、詳細なファイル分析レポートを取得できるようにしたい。

#### 受入基準

1. WHEN ユーザーがファイルハッシュ（MD5、SHA-1、SHA-256）を指定してファイル分析を要求した場合、THE MCP_Server SHALL VirusTotal_APIを使用してファイル分析結果を返す
2. WHEN ファイル分析を実行する場合、THE MCP_Server SHALL 動作、ドロップファイル、接触ドメイン、接触IP、埋め込みURL、関連脅威アクターの関係データを自動取得する
3. WHEN 存在しないハッシュが提供された場合、THE MCP_Server SHALL 適切なエラーメッセージを返す
4. THE MCP_Server SHALL MD5、SHA-1、SHA-256の全てのハッシュ形式をサポートする

### 要件3: IPアドレス セキュリティ分析

**ユーザーストーリー:** セキュリティアナリストとして、IPアドレスの評判と地理的情報を分析したいので、包括的なIP分析レポートを取得できるようにしたい。

#### 受入基準

1. WHEN ユーザーがIPアドレスを指定してIP分析を要求した場合、THE MCP_Server SHALL VirusTotal_APIを使用してIP分析結果を返す
2. WHEN IP分析を実行する場合、THE MCP_Server SHALL 通信ファイル、履歴SSL証明書、解決履歴、関連脅威アクターの関係データを自動取得する
3. WHEN 無効なIPアドレスが提供された場合、THE MCP_Server SHALL 適切なエラーメッセージを返す
4. THE MCP_Server SHALL IPv4とIPv6の両方のアドレス形式をサポートする

### 要件4: ドメイン セキュリティ分析

**ユーザーストーリー:** セキュリティアナリストとして、ドメインのDNS情報とWHOISデータを分析したいので、包括的なドメイン分析レポートを取得できるようにしたい。

#### 受入基準

1. WHEN ユーザーがドメイン名を指定してドメイン分析を要求した場合、THE MCP_Server SHALL VirusTotal_APIを使用してドメイン分析結果を返す
2. WHEN ドメイン分析を実行する場合、THE MCP_Server SHALL サブドメイン、履歴SSL証明書、解決履歴、関連脅威アクターの関係データを自動取得する
3. WHEN オプションで特定の関係タイプが指定された場合、THE MCP_Server SHALL 指定された関係データのみを取得する
4. WHEN 無効なドメイン名が提供された場合、THE MCP_Server SHALL 適切なエラーメッセージを返す

### 要件5: 詳細関係データ分析

**ユーザーストーリー:** セキュリティアナリストとして、特定の関係タイプについて詳細な調査を行いたいので、ページネーション機能付きで関係データを取得できるようにしたい。

#### 受入基準

1. WHEN ユーザーがURL、ファイル、IP、ドメインの特定の関係タイプを指定した場合、THE MCP_Server SHALL 該当する関係データを返す
2. WHEN 関係データの取得時にlimitパラメータが指定された場合、THE MCP_Server SHALL 指定された数（1-40）の結果を返す
3. WHEN 関係データの取得時にcursorパラメータが指定された場合、THE MCP_Server SHALL ページネーション機能を使用して続きのデータを返す
4. WHEN 存在しない関係タイプが指定された場合、THE MCP_Server SHALL 適切なエラーメッセージを返す

### 要件6: API認証とセキュリティ

**ユーザーストーリー:** システム管理者として、VirusTotal APIへの安全なアクセスを確保したいので、適切な認証機能を実装したい。

#### 受入基準

1. THE MCP_Server SHALL 環境変数からVirusTotal APIキーを読み込む
2. WHEN APIキーが設定されていない場合、THE MCP_Server SHALL 起動時にエラーを発生させる
3. WHEN VirusTotal APIへのリクエストを送信する場合、THE MCP_Server SHALL 適切なAPIキーをヘッダーに含める
4. WHEN API認証エラーが発生した場合、THE MCP_Server SHALL 適切なエラーメッセージを返す

### 要件7: レート制限とエラーハンドリング

**ユーザーストーリー:** システム管理者として、VirusTotal APIのレート制限を遵守し、適切なエラーハンドリングを行いたいので、堅牢なエラー処理機能を実装したい。

#### 受入基準

1. WHEN VirusTotal APIからレート制限エラーが返された場合、THE MCP_Server SHALL 適切なエラーメッセージを返す
2. WHEN ネットワークエラーが発生した場合、THE MCP_Server SHALL タイムアウト（30秒）を設定し、適切なエラーメッセージを返す
3. WHEN APIレスポンスが無効な場合、THE MCP_Server SHALL 適切なエラーメッセージを返す
4. THE MCP_Server SHALL 全てのHTTPエラーを適切にキャッチし、ユーザーフレンドリーなエラーメッセージに変換する

### 要件8: 複数トランスポート対応

**ユーザーストーリー:** 開発者として、異なる統合方法に対応したいので、複数のトランスポート方式をサポートしたい。

#### 受入基準

1. THE MCP_Server SHALL STDIOトランスポートをサポートする（Claude Desktop統合用）
2. THE MCP_Server SHALL SSEトランスポートをサポートする（Web統合用）
3. WHEN MCP_TRANSPORT環境変数が"stdio"に設定された場合、THE MCP_Server SHALL STDIOトランスポートを使用する
4. WHEN MCP_TRANSPORT環境変数が設定されていないか"sse"の場合、THE MCP_Server SHALL SSEトランスポートを使用し、ホスト0.0.0.0、ポート8000でリッスンする

### 要件9: データフォーマットと表示

**ユーザーストーリー:** セキュリティアナリストとして、分析結果を理解しやすい形式で受け取りたいので、構造化された読みやすいレポート形式で結果を取得したい。

#### 受入基準

1. WHEN 分析結果を返す場合、THE MCP_Server SHALL マークダウン形式で構造化されたレポートを生成する
2. WHEN 検出統計を表示する場合、THE MCP_Server SHALL 悪意のある、疑わしい、クリーン、未検出の各カテゴリの数を明確に表示する
3. WHEN 関係データを表示する場合、THE MCP_Server SHALL 各関係タイプごとに項目数と詳細リストを表示する
4. WHEN 関係項目を表示する場合、THE MCP_Server SHALL 項目タイプに応じて適切な形式（ドメイン、IP、ファイル、URL等）で表示する

### 要件10: 非同期処理とパフォーマンス

**ユーザーストーリー:** システム管理者として、効率的なAPI利用を実現したいので、非同期処理によるパフォーマンス最適化を行いたい。

#### 受入基準

1. THE MCP_Server SHALL 全てのVirusTotal API呼び出しを非同期で実行する
2. WHEN URL分析を実行する場合、THE MCP_Server SHALL スキャン送信後に適切な待機時間（3秒）を設ける
3. WHEN 複数の関係データを取得する場合、THE MCP_Server SHALL 各関係タイプを並行して取得する
4. WHEN 関係データの取得でエラーが発生した場合、THE MCP_Server SHALL 他の関係データの取得を継続する