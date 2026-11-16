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

## Contact handling
An organization may have multiple contacts in CRM. Contacts are not organization-singletons.

Each contact sync must preserve:
- `organization_canonical_id`
- `opportunity_canonical_id` when the person is tied to a specific notice
- `source_ref.record_id`
- `role_context` such as opportunity POC, agency contact, prime-vendor contact, federal department contact, state department contact, partner contact, or general
- the contact's own `person_<tenant>_<stable-key>`

Create or associate a contact when an authoritative source provides a named person and reliable contact path. Do not merge or suppress contacts solely because they share an organization, name, or email domain. Match by provider ID, canonical ID, or corroborated identity within the source and role context.

A contact can be associated with the organization and with one or more relevant opportunities. A missing contact is a partial sync result when the source provides a reliable named contact; it is not a failure when the source provides no contact path.

## DW-specific state
Do not flatten or overwrite decision-support state when syncing:
`decision`, `posture`, `fit_score`, risk buckets, guardrails, gaps, inputs needed, sub/prime analysis, opportunity history, report references, and human-review state remain in DW.

## HubSpot mapping
Use the live shared properties:
- Company: `cintman_canonical_id`, `cintman_source_reference`, `cintman_campaign_id`, `cintman_campaign_name`
- Contact: `cintman_canonical_id`, `cintman_source_reference`, `cintman_campaign_id`, `cintman_campaign_name`

`cintman_campaign_name` is the human-readable display label for saved views. `cintman_campaign_id` remains the stable matching key.

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
  "contact_context": {
    "organization_canonical_id": "org_<tenant>_<stable-key>",
    "opportunity_canonical_id": "opportunity_<tenant>_<stable-key>",
    "role_context": "opportunity_poc"
  },
  "crm_sync": {
    "company_external_id": null,
    "contact_external_ids": [],
    "status": "NOT_STARTED | VERIFIED | HOLD | PARTIAL"
  }
}
```

## Reconciliation
1. Read the current DW profile, opportunity record, and compact history first.
2. Match organizations by provider ID, canonical ID, then corroborated identity.
3. Match contacts by provider ID, canonical ID, then corroborated identity within organization, source record, and role context.
4. HOLD ambiguous or conflicting records.
5. Never update NO-GO, excluded, or human-held opportunities.
6. Preserve existing user edits and decision history.
7. Read back every company, contact, association, canonical ID, source reference, and suppression state.
8. A company success with missing contacts or missing opportunity provenance is partial failure when the source supplied those fields.
9. Missing CRM records are queued for explicit review; do not auto-create from discovery alone.
