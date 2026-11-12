Importálja mindkét JSON-t az n8n-be.

Először az docflow_error_handler-t importálja, és jegyezze fel az ID-ját.

A docflow workflow Settings → Error workflow mezőjébe ezt az ID-t állítsa be (most ERROR_WORKFLOW_ID a placeholder).

Hozza létre a saját credentialeket: Discord Webhook és Google Gemini API.

Hozza létre a docflow_audit_log data table-t ezekkel az oszlopokkal: timestamp, execution_id, record_id, stage, status, category, priority, http_status, confidence, needs_review, error_summary.

A HTTP Request node URL-jébe írja be a saját célcímét.
