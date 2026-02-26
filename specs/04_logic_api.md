# ロジック・API仕様書 (Draft)

## 1. 業務パイプライン (Pipelines)
各ロジックは `specs/05_architecture.md` で定義されたパイプライン層として実装され、複数のサービスをオーケストレートする。

### 1.1 部品インポートパイプライン (`ImportPipeline`)
- **入力**: `files: File[]`, `projectId: string`, `unitId: string`
- **実装方針**: 
    - 開発コスト削減およびデータ不整合防止のため、すべてのステップを **同期処理（直列実行）** で行う。
    - ユーザーはアップロードから本登録完了まで画面上で待機し、途中でバックグラウンド処理への移行は行わない。
- **ステップ**:
    1. **プレ登録 (Pending)**: 
        - ファイル名から `part_number` を抽出。
        - DBに `status: "PENDING"` で `Part` レコードを作成し、一意のIDを確保。
    2. **ストレージ保存**: 
        - 確保したIDをディレクトリ構造やファイル名に含めて Firebase Storage へアップロード（例: `parts/{id}/file.stl`）。
    3. **本登録 (Activate)**: 
        - アップロード成功を受けて、DBの `stl_url` を更新し、`status: "ACTIVE"` に変更。
    4. **クリーンアップ (非同期/エラー時)**:
        - 失敗時は必要に応じて `PENDING` レコードを削除、またはリトライト対象とする。

### 1.2 製造オーダーパイプライン (`OrderPipeline`)
- **入力**: `partId: string`, `machineId: string`, `quantity: number`
- **ステップ**:
    1. **事前チェック**: 部品の存在確認と基本情報の取得。指定数以上の空きBoxがあるか確認。
    2. **ドメイン操作 (トランザクション内)**:
        - **空きBox確保**: 
            - `Box` テーブルから `status: "AVAILABLE"` かつ最小IDのレコードを、`quantity` 分取得。
            - **排他制御**: `SELECT FOR UPDATE` 等を用いて、取得対象のBoxレコードをロックする。
        - **Boxステータス更新**: 取得したBoxのステータスを `"OCCUPIED"` に更新。
        - **個体生成**: 取得した `box_id`, `machine_id` と紐付けた `PartItem` レコードを `quantity` 分一括生成（初期ステータス: `READY`）。
        - **履歴記録**: 各個体の初期履歴を `HistoryService` で記録。

### 1.3 ステータス遷移パイプライン (`StatusUpdatePipeline`)
- **入力**: `itemId: string`, `newStatus: string`
- **ステップ**:
    1. 遷移ルールチェック（`dccs/SN-923-stats.md` に基づくバリデーション。戻り遷移や造形失敗等も考慮）。
    2. `ProductionControlService` により `PartItem` のステータスを更新。
    3. `HistoryService` により遷移ログを保存。
    4. ステータスが `SHIPPED` または `DISCARD` の場合、`StorageLogisticsService` を通じて `Box` を解放。

### 1.4 ダウンロードURL発行パイプライン (`DownloadPipeline`)
- **入力**: `partId: string`
- **ステップ**:
    1. 部品のアクセス権限（存在確認）チェック。
    2. `StorageService` を経由して一時的な閲覧URLを取得。

## 2. バリデーションルール
- STLファイルのバリデーション（サイズ、拡張子）。
- ステータス遷移の整合性チェック（例: 完了済みから未着手への不用意な戻りを制限など）。
