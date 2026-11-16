# DW CRM Operations

## Contract
DW consumes `shared-crm-identity.json` v0.2.0. DW owns opportunity analysis and decision state; CRM owns provider IDs and suppression state.

## Canonical records
Every DW record sent to CRM carries:
- `tenant_id`
- `canonical_id`
- `source_ref` with `source_type: research`, subtype, campaign ID, record ID, and provenance
- `external_refs` for notice IDs, search IDs, agency/prime keys, and legacy history keys

Canonical entities:
- organization: agency, prime, partner, or other company
- person: named government, prime, agency, or partner contact
- link: public source or profile URL
- evidence: observed claim tied to a source and section/page
- campaign: DW discovery or research campaign
- opportunity: RFQ/RFP/RFI or prime-landscape opportunity

## DW-specific state
Do not flatten or overwrite decision-support state when syncing:
`decision`, `posture`, `fit_score`, risk buckets, guardrails, gaps, inputs needed, sub/prime analysis, opportunity history, report references, and human-review state remain in DW.

## HubSpot mapping
Use the live shared properties:
- Company: `cintman_canonical_id`, `cintman_source_reference`, `cintman_campaign_id`
- Contact: `cintman_canonical_id`, `cintman_source_reference`, `cintman_campaign_id`

Provider object IDs remain under `crm_sync.external_ids`. Links, evidence, campaigns, and opportunities remain in DW unless a future CRM object mapping is explicitly approved.

## Handoff payload
```json
{
  "operation_key": "dw:<campaign_id>:<opportunity_id>:<action>",
  "mapping_version": "shared-crm-identity-0.2.0",
  "canonical_id": "opportunity_<tenant>_<stable-key>",
  "source_ref": {
    "source_type": "research",
    "source_subtype": "rfq | rfi | prime_landscape | agency | partner",
    "campaign_id": "campaign_<tenant>_<stable-key>",
    "record_id": "opportunity_<tenant>_<stable-key>",
    "provenance": [{"url": "https://...", "observed_at": "ISO-8601", "source_label": "SAM.gov"}]
  },
  "decision_state": "PENDING | GO | NO_GO | CONDITIONAL",
  "posture": "SUB | PRIME | BOTH | MONITOR",
  "crm_sync": {
    "company_external_id": null,
    "contact_external_ids": [],
    "status": "NOT_STARTED | VERIFIED | HOLD | PARTIAL"
  }
}
```

## Reconciliation
1. Read the current DW profile, opportunity record, and compact history first.
2. Match by provider ID, canonical ID, then corroborated identity.
3. HOLD ambiguous or conflicting records.
4. Never update NO-GO, excluded, or human-held opportunities.
5. Preserve existing user edits and decision history.
6. Read back every company, contact, association, canonical ID, source reference, and suppression state.
7. A company success with missing contacts or missing opportunity provenance is partial failure.
8. Missing CRM records are queued for explicit review; do not auto-create from discovery alone.
