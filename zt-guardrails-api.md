# ZeroTrusted.ai Guardrails API

Claude Code skill for integrating ZeroTrusted.ai's guardrails API into any application.
Covers PII detection, anonymization, deanonymization, compliance checking, hallucination
detection, and audit logging.

---

## 1. Architecture Overview

### Data Flow

```
User input (prompt, file, or form data)
  -> Detect PII (detect-sensitive-keywords)
    -> If PII found:
       -> Anonymize (anonymize-sensitive-keywords)
         -> Send anonymized text to LLM
           -> Receive LLM response (contains anonymized placeholders)
             -> Deanonymize (deanonymize-sensitive-keywords)
               -> Return original values in response to user
    -> If no PII:
       -> Send directly to LLM
  -> Post-response checks:
     -> Hallucination / Reliability check (evaluate-reliability)
     -> Compliance check (check-compliance)
     -> Audit log (log-event)
```

### Environment Variables

```bash
# Required — your ZeroTrusted.ai API key (get from https://dev.zerotrusted.ai)
ZT_GUARDRAILS_API_KEY=zt-your-api-key-here

# Required for hallucination check — your LLM provider key, encrypted by ZeroTrusted.ai
ZT_ENCRYPTED_PROVIDER_KEY=your-encrypted-key

# Optional — override default API base URLs
ZT_GUARDRAILS_URL=https://dev-guardrails.zerotrusted.ai/api/v3
ZT_AGENTS_URL=https://dev-agents.zerotrusted.ai/api/v3
ZT_HALLUCINATION_URL=https://dev-agents.zerotrusted.ai/api/v1/responses/evaluate-reliability
```

### Configuration Module

Create a shared config file that reads from environment variables:

```typescript
// src/lib/zerotrusted.ts

export const ZT_GUARDRAILS_URL = process.env.ZT_GUARDRAILS_URL
  || "https://dev-guardrails.zerotrusted.ai/api/v3";

export const ZT_AGENTS_URL = process.env.ZT_AGENTS_URL
  || "https://dev-agents.zerotrusted.ai/api/v3";

export const ZT_HALLUCINATION_URL = process.env.ZT_HALLUCINATION_URL
  || "https://dev-agents.zerotrusted.ai/api/v1/responses/evaluate-reliability";

export const ZT_API_KEY = process.env.ZT_GUARDRAILS_API_KEY || "";

export const ZT_ENCRYPTED_PROVIDER_KEY = process.env.ZT_ENCRYPTED_PROVIDER_KEY || "";

// Comprehensive PII entity types for detection/anonymization
export const PII_ENTITY_TYPES = [
  "email", "email address", "gmail", "person", "organization",
  "phone number", "address", "passport number", "credit card number",
  "social security number", "health insurance id number", "itin",
  "date time", "us passport_number", "date", "time",
  "crypto currency number", "url", "date of birth",
  "mobile phone number", "bank account number", "medication", "cpf",
  "driver's license number", "tax identification number",
  "medical condition", "identity card number", "national id number",
  "ip address", "iban", "credit card expiration date", "username",
  "health insurance number", "registration number", "student id number",
  "insurance number", "flight number", "landline phone number",
  "blood type", "cvv", "reservation number", "digital signature",
  "social media handle", "license plate number", "cnpj", "postal code",
  "serial number", "vehicle registration number", "credit card brand",
  "fax number", "visa number", "insurance company",
  "identity document number", "transaction number",
  "national health insurance number", "cvc",
  "birth certificate number", "train ticket number",
  "passport expiration date", "social_security_number", "medical license",
].join(", ");
```

> **IMPORTANT**: Never hardcode API keys. Always read from `process.env`.

---

## 2. Authentication

All ZeroTrusted.ai API endpoints use the `X-API-Key` header:

```typescript
headers: { "X-API-Key": process.env.ZT_GUARDRAILS_API_KEY }
```

---

## 3. API Endpoints Reference

| Method | Base | Path | Purpose |
|--------|------|------|---------|
| POST | Guardrails | `/detect-sensitive-keywords` | Detect PII in text |
| POST | Guardrails | `/anonymize-sensitive-keywords` | Anonymize PII with fake values |
| POST | Guardrails | `/deanonymize-sensitive-keywords` | Restore original values from mappings |
| POST | Agents | `/responses/check-compliance` | Check against compliance frameworks |
| POST | Agents | `/api/v1/responses/evaluate-reliability` | Hallucination / reliability scoring |
| POST | Agents | `/audits/log-event` | Log an audit event |
| POST | Agents | `/chat` | Chat with model through guardrails |
| POST | Agents | `/anonymize-sensitive-keywords` | Anonymize with model-assisted detection |

---

## 4. PII Detection

Scans text and returns detected PII entities with their types.

### Request

```typescript
async function detectPII(text: string): Promise<{
  has_pii: boolean;
  detections: { entity_type: string; text: string }[];
}> {
  const formData = new FormData();
  formData.append("user_prompt", text);
  formData.append("pii_entity_types", PII_ENTITY_TYPES);

  const res = await fetch(`${ZT_GUARDRAILS_URL}/detect-sensitive-keywords`, {
    method: "POST",
    headers: { "X-API-Key": ZT_API_KEY },
    body: formData,
  });

  if (!res.ok) throw new Error(`PII detection failed: ${res.status}`);

  const data = await res.json();
  const piiEntities: [string, string][] = data?.privacy_result?.pii_entities || [];

  return {
    has_pii: piiEntities.length > 0,
    detections: piiEntities.map(([text, entityType]) => ({
      entity_type: entityType,
      text,
    })),
  };
}
```

### Response Format

```json
{
  "privacy_result": {
    "pii_entities": [
      ["john.doe@example.com", "EMAIL_ADDRESS"],
      ["John Doe", "PERSON"],
      ["123 Main Street", "ADDRESS"]
    ],
    "processing_stats": {
      "entities_detected": 3
    }
  }
}
```

### Common PII Entity Types Returned

| Entity Type | Example |
|------------|---------|
| `PERSON` | John Doe |
| `EMAIL_ADDRESS` | john@example.com |
| `PHONE_NUMBER` | +1-555-0100 |
| `CREDIT_CARD` | 4111 1111 1111 1111 |
| `US_SSN` | 123-45-6789 |
| `ADDRESS` | 123 Main St, Anytown |
| `DATE_TIME` | 12/24/1990 |
| `IP_ADDRESS` | 192.168.1.1 |
| `IBAN` | DE89 3704 0044 0532 0130 00 |

---

## 5. Anonymization

Replaces detected PII with realistic fake values and returns a mapping for later deanonymization.

### Request

```typescript
async function anonymize(text: string): Promise<{
  anonymized_text: string;
  original_text: string;
  mappings: { original: string; anonymized: string }[];
}> {
  const formData = new FormData();
  formData.append("user_prompt", text);
  formData.append("pii_entity_types", PII_ENTITY_TYPES);
  formData.append("response_language", "EN");

  const res = await fetch(`${ZT_GUARDRAILS_URL}/anonymize-sensitive-keywords`, {
    method: "POST",
    headers: { "X-API-Key": ZT_API_KEY },
    body: formData,
  });

  if (!res.ok) throw new Error(`Anonymization failed: ${res.status}`);

  const data = await res.json();
  const privacyResult = data?.privacy_result;

  return {
    anonymized_text: privacyResult?.processed_text || text,
    original_text: privacyResult?.original_text || text,
    mappings: (privacyResult?.anonymized_keywords_mapping || []).map(
      (m: { original: string; anonymized: string }) => ({
        original: m.original,
        anonymized: m.anonymized,
      })
    ),
  };
}
```

### Response Format

```json
{
  "privacy_result": {
    "processed_text": "My name is Jennifer Benjamin and my email is sarah.wilson@example.net",
    "original_text": "My name is John Doe and my email is john.doe@example.com",
    "anonymized_keywords_mapping": [
      { "original": "John Doe", "anonymized": "Jennifer Benjamin" },
      { "original": "john.doe@example.com", "anonymized": "sarah.wilson@example.net" }
    ]
  }
}
```

### Optional Parameters

| Field | Type | Description |
|-------|------|-------------|
| `anonymize_keywords` | string | Additional custom keywords to anonymize |
| `safeguard_keywords` | string | Keywords that should trigger a warning |
| `preserve_keywords` | string | Keywords to NOT anonymize |
| `uploaded_file` | string | File content to include in anonymization |

---

## 6. Deanonymization

Reverses anonymization using the stored mappings.

### Request

```typescript
async function deanonymize(
  anonymizedText: string,
  mappings: { original: string; anonymized: string }[]
): Promise<string> {
  const formData = new FormData();
  formData.append("user_prompt", anonymizedText);
  formData.append("anonymized_keywords_mapping", JSON.stringify(mappings));

  const res = await fetch(`${ZT_GUARDRAILS_URL}/deanonymize-sensitive-keywords`, {
    method: "POST",
    headers: { "X-API-Key": ZT_API_KEY },
    body: formData,
  });

  if (!res.ok) throw new Error(`Deanonymization failed: ${res.status}`);

  const data = await res.json();
  return data?.privacy_result?.processed_text || anonymizedText;
}
```

### Mappings Format

The `anonymized_keywords_mapping` field expects a JSON array:

```json
[
  { "original": "John Doe", "anonymized": "Jennifer Benjamin" },
  { "original": "12/24", "anonymized": "Dec-17-1975" }
]
```

---

## 7. Hallucination / Reliability Check

Evaluates whether an AI response is reliable or potentially hallucinated.

### Request

```typescript
async function checkReliability(
  userPrompt: string,
  aiResponse: string,
  model: string = "gpt-4.1"
): Promise<{ reliable: boolean; score: number; explanation: string }> {
  const res = await fetch(`${ZT_HALLUCINATION_URL}?service=openai`, {
    method: "POST",
    headers: {
      "X-API-Key": ZT_API_KEY,
      "content-type": "application/json",
    },
    body: JSON.stringify({
      provider_api_key: ZT_ENCRYPTED_PROVIDER_KEY,
      evaluator_model: model,
      candidate_responses: [
        { model: "assistant", response: aiResponse },
      ],
      user_prompt: userPrompt,
      is_provider_api_key_encrypted: true,
      response_language: "EN",
    }),
  });

  if (!res.ok) throw new Error(`Reliability check failed: ${res.status}`);

  const data = await res.json();
  let score = 50;
  let explanation = "";

  if (data.success && data.data) {
    const parsed = typeof data.data === "string" ? JSON.parse(data.data) : data.data;
    const result = Object.values(parsed).find(
      (v: unknown) => typeof v === "object" && v !== null && "score" in (v as Record<string, unknown>)
    ) as { score: string; explanation?: string } | undefined;

    if (result) {
      score = parseInt(result.score, 10) || 50;
      explanation = result.explanation || "";
    }
  }

  return { reliable: score >= 70, score, explanation };
}
```

### Response Format

```json
{
  "success": true,
  "data": "{\"assistant\":{\"rank\":\"1\",\"score\":\"92\",\"explanation\":\"The response accurately...\"}}"
}
```

> **Note**: `data` is a JSON string that needs to be parsed. Score is 0-100 where >= 70 is considered reliable.

---

## 8. Compliance Check

Evaluates detected PII against regulatory compliance frameworks.

### Request

```typescript
async function checkCompliance(
  detectedEntities: [string, string][],
  anonymizationMethod: string,
  originalPrompt: string,
  frameworks: string[] = ["gdpr", "ccpa", "hipaa", "pci_dss"]
): Promise<unknown> {
  const res = await fetch(`${ZT_AGENTS_URL}/responses/check-compliance`, {
    method: "POST",
    headers: {
      "X-API-Key": ZT_API_KEY,
      "content-type": "application/json",
    },
    body: JSON.stringify({
      enable_compliance_report: true,
      detected_pii_entities: [detectedEntities, anonymizationMethod, originalPrompt],
      compliance_frameworks: frameworks,
      file_pii_entities: [],
      source_filename: "",
    }),
  });

  if (!res.ok) throw new Error(`Compliance check failed: ${res.status}`);
  return res.json();
}
```

### Supported Frameworks

| Framework | Key |
|-----------|-----|
| GDPR | `gdpr` |
| CCPA | `ccpa` |
| HIPAA | `hipaa` |
| HITECH | `hitech` |
| HITRUST | `hitrust` |
| PCI DSS | `pci_dss` |
| GLBA | `glba` |
| LGPD | `lgpd` |
| APPI | `appi` |

---

## 9. Audit Logging

Logs a complete interaction event for audit trail and compliance.

### Request

```typescript
async function logAuditEvent(event: {
  prompt: string;
  llm_model: string;
  llm_output: string;
  privacy_level: number;
  anonymization_checked: boolean;
  sanitized_pii_count: number;
  highlighted_original_prompt: string;
  highlighted_anonymized_prompt: string;
  pii_list_str: string;
}): Promise<void> {
  await fetch(`${ZT_AGENTS_URL}/audits/log-event`, {
    method: "POST",
    headers: {
      "X-API-Key": ZT_API_KEY,
      "content-type": "application/json",
    },
    body: JSON.stringify({
      ...event,
      unique_id: Date.now(),
      user_ip: "",
      user_location_detail: "{}",
      browser_detail: "{}",
      device_type: "Server",
      file_pii_array_str: "[]",
    }),
  });
}
```

---

## 10. Model-Assisted Anonymization

Uses an LLM to enhance PII detection accuracy before anonymizing.

### Request

```typescript
async function anonymizeWithModel(
  text: string,
  modelName: string = "gpt-3.5-turbo"
): Promise<unknown> {
  const formData = new FormData();
  formData.append("user_prompt", text);
  formData.append("pii_entity_types", PII_ENTITY_TYPES);
  formData.append("provider_api_key", ZT_ENCRYPTED_PROVIDER_KEY);
  formData.append("model_name", modelName);
  formData.append("is_provider_api_key_encrypted", "true");
  formData.append("enable_streaming", "false");
  formData.append("task", "anonymize");
  formData.append("use_context_aware_processing", "false");

  const res = await fetch(`${ZT_AGENTS_URL}/anonymize-sensitive-keywords`, {
    method: "POST",
    headers: { "X-API-Key": ZT_API_KEY },
    body: formData,
  });

  if (!res.ok) throw new Error(`Model-assisted anonymization failed: ${res.status}`);
  return res.json();
}
```

---

## 11. Complete Guardrails Pipeline

End-to-end flow combining detection, anonymization, LLM call, and deanonymization:

```typescript
async function guardrailedLLMCall(
  userPrompt: string,
  callLLM: (prompt: string) => Promise<string>
): Promise<{ response: string; wasAnonymized: boolean; compliance?: unknown }> {
  // Step 1: Detect PII
  const detection = await detectPII(userPrompt);

  if (!detection.has_pii) {
    // No PII — send directly
    const response = await callLLM(userPrompt);
    return { response, wasAnonymized: false };
  }

  // Step 2: Anonymize
  const { anonymized_text, mappings } = await anonymize(userPrompt);

  // Step 3: Send anonymized text to LLM
  const llmResponse = await callLLM(anonymized_text);

  // Step 4: Deanonymize the LLM response
  const response = await deanonymize(llmResponse, mappings);

  // Step 5 (optional): Check reliability
  const reliability = await checkReliability(userPrompt, response);

  // Step 6 (optional): Check compliance
  const compliance = await checkCompliance(
    detection.detections.map((d) => [d.text, d.entity_type] as [string, string]),
    "Fictionalize",
    userPrompt
  );

  return { response, wasAnonymized: true, compliance };
}
```

---

## 12. Error Handling Best Practices

```typescript
// Retry wrapper for intermittent API failures
async function withRetry<T>(
  fn: () => Promise<T>,
  retries = 2,
  label = "ZeroTrusted API"
): Promise<T> {
  for (let attempt = 1; attempt <= retries; attempt++) {
    try {
      return await fn();
    } catch (err) {
      if (attempt === retries) throw err;
      console.warn(`[${label}] Attempt ${attempt} failed, retrying...`);
    }
  }
  throw new Error("Unreachable");
}

// Usage
const result = await withRetry(() => anonymize(text), 2, "Anonymize");
```

### Payload Size Limits

- Truncate user prompts to ~10KB before sending to detect/anonymize endpoints
- Truncate AI responses to ~3KB before sending to hallucination check
- Large payloads can cause 502 gateway timeouts

```typescript
const scanText = text.length > 10000 ? text.slice(0, 10000) : text;
```

---

## 13. Framework Examples

### Next.js API Route

```typescript
// src/app/api/guardrails/detect/route.ts
import { NextRequest } from "next/server";

export async function POST(request: NextRequest) {
  const { text } = await request.json();
  if (!text) return Response.json({ error: "text required" }, { status: 400 });

  const formData = new FormData();
  formData.append("user_prompt", text.slice(0, 10000));
  formData.append("pii_entity_types", PII_ENTITY_TYPES);

  const res = await fetch(
    `${process.env.ZT_GUARDRAILS_URL}/detect-sensitive-keywords`,
    {
      method: "POST",
      headers: { "X-API-Key": process.env.ZT_GUARDRAILS_API_KEY! },
      body: formData,
    }
  );

  if (!res.ok) return Response.json({ error: "Detection failed" }, { status: 502 });
  return Response.json(await res.json());
}
```

### Express.js

```typescript
import express from "express";

const app = express();
app.use(express.json());

app.post("/api/detect-pii", async (req, res) => {
  const { text } = req.body;
  const formData = new FormData();
  formData.append("user_prompt", text);
  formData.append("pii_entity_types", PII_ENTITY_TYPES);

  const response = await fetch(
    `${process.env.ZT_GUARDRAILS_URL}/detect-sensitive-keywords`,
    {
      method: "POST",
      headers: { "X-API-Key": process.env.ZT_GUARDRAILS_API_KEY! },
      body: formData,
    }
  );

  res.json(await response.json());
});
```

### Python (requests)

```python
import os
import requests

ZT_API_KEY = os.environ["ZT_GUARDRAILS_API_KEY"]
ZT_URL = os.environ.get("ZT_GUARDRAILS_URL", "https://dev-guardrails.zerotrusted.ai/api/v3")

def detect_pii(text: str) -> dict:
    response = requests.post(
        f"{ZT_URL}/detect-sensitive-keywords",
        headers={"X-API-Key": ZT_API_KEY},
        data={
            "user_prompt": text,
            "pii_entity_types": "email, person, phone number, credit card number, social security number",
        },
    )
    response.raise_for_status()
    return response.json()

def anonymize(text: str) -> dict:
    response = requests.post(
        f"{ZT_URL}/anonymize-sensitive-keywords",
        headers={"X-API-Key": ZT_API_KEY},
        data={
            "user_prompt": text,
            "pii_entity_types": "email, person, phone number, credit card number",
            "response_language": "EN",
        },
    )
    response.raise_for_status()
    return response.json()

def deanonymize(text: str, mappings: list[dict]) -> str:
    import json
    response = requests.post(
        f"{ZT_URL}/deanonymize-sensitive-keywords",
        headers={"X-API-Key": ZT_API_KEY},
        data={
            "user_prompt": text,
            "anonymized_keywords_mapping": json.dumps(mappings),
        },
    )
    response.raise_for_status()
    return response.json()["privacy_result"]["processed_text"]
```
