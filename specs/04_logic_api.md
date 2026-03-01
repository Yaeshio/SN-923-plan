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
- **入力**: `itemId: string`, `newStatus: string`, `reason_code?: string`, `comment?: string`
- **ステップ**:
    1. **遷移ルール・ガード条件チェック**:
        - `dccs/SN-923-stats.md` の状態遷移図に反する遷移を禁止。
        - **工程スキップ禁止**: 工程順序をスキップする遷移（例: `PRINTING` から `INSPECTION`）をガード条件で弾く。
    2. **リソース再確保（例外系リカバリ）**:
        - 戻り遷移やリカバリ遷移（`SHIPPED/DISCARD/CANCELLED` からの復帰）において、対象の `PartItem` に紐づく `Box` が既に解放されている、あるいは現在 `"AVAILABLE"` 状態である場合、`OrderPipeline` のロジックと同様に利用可能な Box を再割り当てする。
    3. **ドメイン更新 (トランザクション内)**:
        - `ProductionControlService` により `PartItem` のステータスを更新。
        - ステータスが `SHIPPED`, `DISCARD`, または `CANCELLED` に遷移する場合、`StorageLogisticsService` を通じて紐付いている `Box` を解放する。
    4. **履歴記録**:
        - `HistoryService` により、`reason_code` と `comment` を含む詳細な履歴を保存。

### 1.4 ダウンロードURL発行パイプライン (`DownloadPipeline`)
- **入力**: `partId: string`
- **ステップ**:
    1. 部品のアクセス権限（存在確認）チェック。
    2. `StorageService` を経由して一時的な閲覧URLを取得。

## 2. バリデーションルール
- STLファイルのバリデーション（サイズ、拡張子）。
- ステータス遷移の整合性チェック:
    - `dccs/SN-923-stats.md` の定義に基づいた順方向・逆方向（戻り）の許可。
    - **工程スキップの禁止**: 前後の定義を逸脱した遷移（例: `PRINTING` から `INSPECTION` へ直接飛ぶ）を禁止。
    - 完了ステータス (`SHIPPED/DISCARD/CANCELLED`) からのリカバリ時は、`reason_code` の指定を必須とする。
