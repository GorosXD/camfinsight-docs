# JSONB shapes

The keys of every `jsonb` field. A field's table entry links here.

## documents.extraction

The `POST /ai/receipts/extract` response (ai-import-service), stored as received; finance-service only checks it is a JSON
object of at most 256 KB ([receipts](../specs/receipts/spec.md) RCT-D17, RCT-D18). Null for a file saved without a reading.

```json
{
  "review_status": "NEEDS_REVIEW",
  "engine": "<engine name>",
  "overall_confidence": 0.87,
  "vendor_name":    { "value": "Lucky Market", "confidence": 0.95, "source_text": "LUCKY MARKET" },
  "receipt_number": { "value": "A-1029", "confidence": 0.80, "source_text": "No: A-1029" },
  "receipt_date":   { "value": "2026-11-14", "confidence": 0.90, "source_text": "14/11/2026" },
  "currency":       { "value": "USD", "confidence": 0.99, "source_text": "$" },
  "subtotal":       { "value": "10.00", "confidence": 0.9, "source_text": "Sub total 10.00" },
  "tax_amount":     { "value": "1.00", "confidence": 0.9, "source_text": "VAT 1.00" },
  "total_amount":   { "value": "11.00", "confidence": 0.9, "source_text": "TOTAL 11.00" },
  "payment_method": { "value": "CASH", "confidence": 0.6, "source_text": "Cash" },
  "line_items": [
    { "description": { "value": "Rice 5kg", "confidence": 0.9, "source_text": "..." },
      "quantity": { "value": "1", "confidence": 0.9, "source_text": "..." },
      "unit_price": { "value": "10.00", "confidence": 0.9, "source_text": "..." },
      "line_total": { "value": "10.00", "confidence": 0.9, "source_text": "..." } }
  ],
  "issues": ["..."],
  "raw_lines": ["..."]
}
```

- Every field is optional; a field AI couldn't find is missing or has `value` null.
- `confidence` scale: **to be confirmed by the AI team** (0–1 in the example). Until then, clients must not assume a scale.
