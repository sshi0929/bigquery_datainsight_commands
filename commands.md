# BigQuery Data Insights (Knowledge Catalog) API 実行コマンド集

```bash
# ==============================================================================
# 0. パラメータ定義（環境に合わせて変更してください）
# ==============================================================================
export PROJECT_ID="<PROJECT_ID>"
export DATASET_ID="<DATASET_ID>"
export TABLE_ID="<TABLE_ID>"

# ※ スキャンを実行するリージョンを指定します
#   - 東京リージョンのデータセットの場合: "asia-northeast1"
#   - US マルチリージョンのデータセットの場合: "us-central1"
export LOCATION="<LOCATION>"

# ※ DATASCAN_ID は英小文字・数字・ハイフン(-)のみ使用可能です（アンダースコア "_" は不可）
#   例: "my-table-insight-scan"
export DATASCAN_ID="<DATASCAN_ID>"


# ==============================================================================
# 1. データスキャン（DATA_DOCUMENTATION）リソースの作成
# ------------------------------------------------------------------------------
# Schema Reference : https://docs.cloud.google.com/knowledge-catalog/docs/reference/rest/v1/projects.locations.dataScans#DataScan
# Spec Reference   : https://docs.cloud.google.com/knowledge-catalog/docs/reference/rest/v1/DataDocumentationSpec
# Method Reference : https://docs.cloud.google.com/knowledge-catalog/docs/reference/rest/v1/projects.locations.dataScans/create
# ※ envsubst で table_insight_request.json 内の環境変数 (${PROJECT_ID} 等) を自動展開して送信します
# ==============================================================================
envsubst < table_insight_request.json | curl -s -X POST \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  "https://dataplex.googleapis.com/v1/projects/${PROJECT_ID}/locations/${LOCATION}/dataScans?dataScanId=${DATASCAN_ID}" \
  -d @- | jq .


# ==============================================================================
# 2. 作成されたスキャンリソースの確認
# ==============================================================================
gcloud dataplex datascans list \
  --project="${PROJECT_ID}" \
  --location="${LOCATION}"


# ==============================================================================
# 3. スキャンジョブの実行（オンデマンド実行）
# ------------------------------------------------------------------------------
# Method Reference: https://docs.cloud.google.com/knowledge-catalog/docs/reference/rest/v1/projects.locations.dataScans/run
# ==============================================================================
curl -s -X POST \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  "https://dataplex.googleapis.com/v1/projects/${PROJECT_ID}/locations/${LOCATION}/dataScans/${DATASCAN_ID}:run" | jq .


# ==============================================================================
# 4. スキャンジョブの実行ステータス確認（SUCCEEDED になるまで待機）
# ------------------------------------------------------------------------------
# Method Reference: https://docs.cloud.google.com/knowledge-catalog/docs/reference/rest/v1/projects.locations.dataScans.jobs/list
# ==============================================================================
gcloud dataplex datascans jobs list \
  --datascan="${DATASCAN_ID}" \
  --project="${PROJECT_ID}" \
  --location="${LOCATION}"


# ==============================================================================
# 5. スキャン結果を対象 BigQuery テーブルに紐付け（コンソール GUI「分析情報」タブ表示用）
#    ※ この時点ではテーブル・カラム説明文は「ドラフト（提案）」状態として表示されます
# ------------------------------------------------------------------------------
# Guide Reference: https://docs.cloud.google.com/bigquery/docs/generate-table-insights
# ==============================================================================
bq update \
  --set_label "dataplex-data-documentation-published-scan:${DATASCAN_ID}" \
  --set_label "dataplex-data-documentation-published-project:${PROJECT_ID}" \
  --set_label "dataplex-data-documentation-published-location:${LOCATION}" \
  "${PROJECT_ID}:${DATASET_ID}.${TABLE_ID}"


# ==============================================================================
# 6. 生成された説明文（テーブル概要・全カラム説明）を BigQuery メタデータへ本登録
#    （コンソール GUI 上の「Save to details」「Save to schema」を一括実行する処理）
# ------------------------------------------------------------------------------
# Result Schema Reference : https://docs.cloud.google.com/knowledge-catalog/docs/reference/rest/v1/DataDocumentationResult#TableResult
# BigQuery Patch Reference: https://cloud.google.com/bigquery/docs/reference/rest/v2/tables/patch
# ==============================================================================

# 6a. スキャン結果からテーブル説明・カラム説明（ネストされた RECORD カラム含む）を抽出し、既存スキーマとマージした JSON を作成
jq -n \
  --argjson scan "$(curl -s -H "Authorization: Bearer $(gcloud auth print-access-token)" "https://dataplex.googleapis.com/v1/projects/${PROJECT_ID}/locations/${LOCATION}/dataScans/${DATASCAN_ID}?view=FULL")" \
  --argjson table "$(bq show --format=json "${PROJECT_ID}:${DATASET_ID}.${TABLE_ID}")" '
  def merge_fields($orig; $gen):
    [ $orig[] | . as $o |
      (($gen // []) | map(select(.name == $o.name)) | .[0]) as $g |
      (if $g != null and ($g.description // "") != "" then .description = $g.description else . end) |
      (if ($o.fields != null) and ($g.fields != null) then .fields = merge_fields($o.fields; $g.fields) else . end)
    ];
  {
    description: ($scan.dataDocumentationResult.tableResult.overview // $table.description),
    schema: {
      fields: merge_fields($table.schema.fields; $scan.dataDocumentationResult.tableResult.schema.fields)
    }
  }
' > approved_table_metadata_patch.json

# 6b. 作成した JSON パッチを BigQuery テーブルに適用して本登録
curl -s -X PATCH \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  "https://bigquery.googleapis.com/bigquery/v2/projects/${PROJECT_ID}/datasets/${DATASET_ID}/tables/${TABLE_ID}" \
  -d @approved_table_metadata_patch.json \
  | jq '{description: .description, updatedColumns: [.schema.fields[] | {name: .name, description: .description}]}'
```
