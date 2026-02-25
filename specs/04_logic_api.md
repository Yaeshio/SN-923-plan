# ロジック・API仕様書 (Draft)

## 1. 業務パイプライン (Pipelines)
各ロジックは `specs/05_architecture.md` で定義されたパイプライン層として実装され、複数のサービスをオーケストレートする。

### 1.1 部品インポートパイプライン (`ImportPipeline`)
- **入力**: `files: File[]`, `projectId: string`
- **ステップ**:
    1. ファイル名から `part_number` を抽出（解析ユーティリティ）。
    2. 既存部品の重複チェック。
    3. `DesignRegistryService` により `Part` レコードを作成。
    4. `StorageService` によりファイルをアップロードし、URLを取得。
    5. `Part` レコードにURLを紐付け更新。

### 1.2 製造オーダーパイプライン (`OrderPipeline`)
- **入力**: `partId: string`, `quantity: number`
- **ステップ**:
    1. 部品の存在確認と基本情報の取得。
    2. 指定個数分の `PartItem` レコードを一括生成（トランザクション有）。
    3. 各個体の初期履歴を `HistoryService` で記録。

### 1.3 ステータス遷移パイプライン (`StatusUpdatePipeline`)
- **入力**: `itemId: string`, `newStatus: string`
- **ステップ**:
    1. 遷移ルールチェック（不正な戻り等のバリデーション）。
    2. `ProductionControlService` により `PartItem` のステータスを更新。
    3. `HistoryService` により遷移ログを保存。
    4. ステータスが特定の状態（例: 出荷済、廃棄等）の場合、`StorageLogisticsService` を通じて `Box` を解放。

### 1.4 ダウンロードURL発行パイプライン (`DownloadPipeline`)
- **入力**: `partId: string`
- **ステップ**:
    1. 部品のアクセス権限（存在確認）チェック。
    2. `StorageService` を経由して一時的な閲覧URLを取得。

## 2. バリデーションルール
- STLファイルのバリデーション（サイズ、拡張子）。
- ステータス遷移の整合性チェック（例: 完了済みから未着手への不用意な戻りを制限など）。
