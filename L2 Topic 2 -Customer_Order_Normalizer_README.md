# Customer Order Normalizer - Assessment Documentation and README

**Assessment:** Advanced Expressions & Dynamic Data Mapping | **Status:** Implementation blueprint; execution evidence to be attached | **Date:** 2026-10-22

## 1. Objective
Build an n8n workflow that normalizes incoming order fields, calculates business dates, classifies order priority and customer tier, and generates an API-ready nested invoice object. This document is a reproducible implementation guide, not a claim that the workflow has passed testing.

## 2. Architecture

```text
Mock_Data -> Clean_Strings -> Format_Dates -> Categorize_Order -> Map_Output
```

- **Mock_Data:** Code node emitting representative raw orders.
- **Clean_Strings:** Edit Fields node; normalize ID/name/email and amount.
- **Format_Dates:** Edit Fields node; Luxon purchase timestamp and SLA date.
- **Categorize_Order:** Edit Fields node; priority and tier expressions.
- **Map_Output:** Code node; dynamic cross-node references and nested invoice output.

## 3. Example input

```json
{
  "orderId": " ord-1001 ",
  "customerName": "  aNNa   lee ",
  "customerEmail": " Anna.Lee@Example.COM ",
  "amount": 1500,
  "purchaseDate": "2026-10-22T10:30:00+05:30"
}
```

## 4. Node configuration and expressions

### Mock_Data
Use a Code node (Run Once for All Items):

```javascript
return [{ json: {
  orderId: ' ord-1001 ', customerName: '  aNNa   lee ',
  customerEmail: ' Anna.Lee@Example.COM ', amount: 1500,
  purchaseDate: '2026-10-22T10:30:00+05:30'
}}];
```

### Clean_Strings
Use Edit Fields in manual mapping mode; **Include Other Input Fields = ON**. Create:

| Output field | Expression |
| --- | --- |
| cleanOrderId | `{{ String($json.orderId ?? '').trim().toUpperCase() }}` |
| cleanName | `{{ String($json.customerName ?? '').trim().replace(/\s+/g, ' ').toLowerCase().replace(/\b\w/g, c => c.toUpperCase()) }}` |
| cleanEmail | `{{ String($json.customerEmail ?? '').trim().toLowerCase() }}` |
| emailDomain | `{{ /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(String($json.customerEmail ?? '').trim()) ? String($json.customerEmail).trim().toLowerCase().split('@')[1] : '' }}` |
| cleanAmount | `{{ Number($json.amount ?? 0) }}` |

Validate required fields and reject invalid records in a production deployment; a fallback empty string alone does not make an invalid email valid.

### Format_Dates
Use Edit Fields; **Include Other Input Fields = ON**. Date values use the timezone encoded in the input; choose a fixed business timezone if required by your assessor.

| Output field | Expression |
| --- | --- |
| formattedPurchaseDate | `{{ DateTime.fromISO($json.purchaseDate ?? '').isValid ? DateTime.fromISO($json.purchaseDate).toFormat('yyyy-MM-dd HH:mm:ss') : '' }}` |
| slaDueDate | `{{ DateTime.fromISO($json.purchaseDate ?? '').isValid ? DateTime.fromISO($json.purchaseDate).plus({ days: 3 }).toFormat('yyyy-MM-dd HH:mm:ss') : '' }}` |

### Categorize_Order
Use Edit Fields; **Include Other Input Fields = ON**. The source assessment does not specify classification thresholds. **Assumption for demonstration only:** priority HIGH when amount >= 1000, MEDIUM when amount >= 500, else LOW; tier GOLD when amount >= 2000, SILVER when amount >= 1000, else BRONZE. Confirm actual rubric thresholds before submission.

| Output field | Expression |
| --- | --- |
| priority | `{{ Number($json.cleanAmount ?? 0) >= 1000 ? 'HIGH' : Number($json.cleanAmount ?? 0) >= 500 ? 'MEDIUM' : 'LOW' }}` |
| customerTier | `{{ Number($json.cleanAmount ?? 0) >= 2000 ? 'GOLD' : Number($json.cleanAmount ?? 0) >= 1000 ? 'SILVER' : 'BRONZE' }}` |

### Map_Output
Use a Code node (Run Once for All Items). This version assumes one mock order and references earlier named nodes. For multiple records, use paired-item-safe mapping rather than `.first()`.

```javascript
const raw = $('Mock_Data').first().json;
const clean = $('Clean_Strings').first().json;
const dates = $('Format_Dates').first().json;
const category = $('Categorize_Order').first().json;
return [{json: {invoice: {
  orderId: clean.cleanOrderId,
  customer: {name: clean.cleanName, emailDomain: clean.emailDomain},
  purchase: {purchaseDate: dates.formattedPurchaseDate, slaDueDate: dates.slaDueDate},
  amount: clean.cleanAmount,
  priority: category.priority,
  customerTier: category.customerTier
}}}];
```

## 5. Expected output for example input

```json
{
  "invoice": {
    "orderId": "ORD-1001",
    "customer": {"name": "Anna Lee", "emailDomain": "example.com"},
    "purchase": {
      "purchaseDate": "2026-10-22 10:30:00",
      "slaDueDate": "2026-10-25 10:30:00"
    },
    "amount": 1500,
    "priority": "HIGH",
    "customerTier": "SILVER"
  }
}
```

## 6. Data mapping design

| Source | Destination | Transformation |
| --- | --- | --- |
| orderId | invoice.orderId | trim, uppercase |
| customerName | invoice.customer.name | trim, collapse spaces, title case |
| customerEmail | invoice.customer.emailDomain | trim, validate, lowercase, extract domain |
| purchaseDate | invoice.purchase.purchaseDate | Luxon format |
| purchaseDate | invoice.purchase.slaDueDate | Luxon plus 3 calendar days |
| amount | invoice.amount | number conversion |
| amount | invoice.priority | assumed threshold rules |
| amount | invoice.customerTier | assumed threshold rules |

## 7. Testing and evidence matrix

| Test ID | Scenario | Expected result | Status |
| --- | --- | --- | --- |
| TC-01 | Extra whitespace and mixed-case name | Anna Lee | Not yet evidenced |
| TC-02 | Spaced order ID | ORD-1001 | Not yet evidenced |
| TC-03 | Valid email | example.com | Not yet evidenced |
| TC-04 | Missing or invalid email | Empty domain; flag/reject in production | Not yet evidenced |
| TC-05 | Valid ISO date | YYYY-MM-DD HH:mm:ss | Not yet evidenced |
| TC-06 | Purchase date plus 3 days | Correct SLA date | Not yet evidenced |
| TC-07 | Amount 1500 | HIGH priority; SILVER tier (assumed rules) | Not yet evidenced |
| TC-08 | Null/missing fields | No expression crash; validation policy enforced | Not yet evidenced |
| TC-09 | Final Map_Output | Nested invoice matches schema | Not yet evidenced |

Capture screenshots of each node's configuration/output, a successful full execution, and final invoice JSON. Record actual pass/fail after running tests.

## 8. Annotations and troubleshooting

Add sticky notes for each stage with **Owner**, **Input schema**, **Output schema**, **Transformation**, and **Failure behavior**. Keep node names exactly as shown in the architecture. Common issues: mismatched field capitalization, missing `Include Other Input Fields`, invalid ISO dates, and referencing `.first()` in multi-item workflows. Never hardcode test output values into Map_Output.

## 9. Production readiness and limitations

- [ ] Confirm business-approved priority/tier thresholds.
- [ ] Decide and document timezone and daylight-saving behavior.
- [ ] Validate required fields, email format, amount and dates before mapping.
- [ ] Test nulls, invalid dates, boundary amounts and multi-order batches.
- [ ] Verify no sensitive customer data or credentials are committed publicly.
- [ ] Export actual n8n workflow JSON and import-test it.
- [ ] Attach execution screenshots and results.
- [ ] Record Loom/YouTube demo and verify accessibility.
- [ ] Publish a new public GitHub repository with README, workflow export and evidence.

## 10. Repository and LMS submission

Suggested structure:

```text
customer-order-normalizer/
  README.md
  workflow/Customer_Order_Normalizer.json
  docs/Customer_Order_Normalizer_Assessment.pdf
  evidence/ (screenshots and test results)
```

In n8n, save workflow, export/download JSON, then commit it to GitHub. Add a video URL and a public repository URL in the LMS. If the LMS rejects a previously used repository, create a distinct public repository for this assessment. The PDF and README are documentation only; they do not replace the required exported workflow JSON, video or real evidence.

**Suggested LMS description:** Built a Customer Order Normalizer using n8n string expressions, Luxon date calculations, conditional classification, and cross-node invoice mapping. Test evidence and workflow export are available in the linked repository. Classification thresholds are demonstration assumptions pending confirmation against assessment-specific business rules.
