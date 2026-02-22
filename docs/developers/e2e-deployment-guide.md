# End-to-End Deployment Guide: Identity Hub + EDC Connectors

This guide documents a complete, working deployment of the Tractus-X dataspace with Identity Hub as the DCP wallet, including the full data exchange lifecycle: catalog request, contract negotiation, transfer process, and data access.

## Architecture

A single-node K3s cluster running all components:

| Component | Version | Role |
|-----------|---------|------|
| EDC Provider | v0.11.0 | Data provider connector (control plane + data plane) |
| EDC Consumer | v0.11.0 | Data consumer connector (control plane + data plane) |
| Identity Hub | v0.1.1 | Multi-tenant DCP wallet (DID, STS, Credentials, Accounts) |
| Issuer Service | v0.1.1 | Credential issuance authority |
| BDRS Server | v0.5.1 | BPN-to-DID resolution directory |

Each component has its own PostgreSQL (non-persistent) and HashiCorp Vault (dev mode).

### Participants

| Role | BPN | DID |
|------|-----|-----|
| Provider | `BPNL00000003AYRE` | `did:web:identity-hub.example.com:BPNL00000003AYRE` |
| Consumer | `BPNL00000003AZQP` | `did:web:identity-hub.example.com:BPNL00000003AZQP` |
| Issuer | `BPNL00000003CRHK` | `did:web:identity-hub.example.com:BPNL00000003CRHK` |

## Helm Chart Fixes Required

The upstream Helm charts (Identity Hub + Issuer Service) have several bugs and missing features that block a working deployment. These are fixed in the `fix/helm-chart-bugs-and-missing-features` branch:

### Bug Fixes

| Chart | Issue | Fix |
|-------|-------|-----|
| Identity Hub | HPA uses `targetCPUUtilizationPercentage` for memory metric | Changed to `targetMemoryUtilizationPercentage` |
| Issuer Service | Same HPA bug | Same fix |
| Identity Hub | PostgreSQL CPU limit set to `500Mi` (memory unit) | Changed to `500m` |
| Issuer Service | Same PostgreSQL bug | Same fix |
| Issuer Service | Log file path is `/app/logs/identityhub.log` (copy-paste) | Changed to `/app/logs/issuerservice.log` |
| Issuer Service | Sets empty `JAVA_TOOL_OPTIONS` when debug is disabled and `useSVE` is false | Removed the unnecessary else branch |

### Missing Features

| Chart | Feature | Description |
|-------|---------|-------------|
| Identity Hub | `hostAliases` | Required for internal DID resolution — deployment template had no support for `hostAliases`, which are needed to resolve `did:web` DIDs without leaving the cluster |
| Issuer Service | `hostAliases` | Same — needed for the Issuer to resolve DIDs internally |
| Identity Hub | Application config in ConfigMap | `edc.ih.iam.id`, `edc.iam.did.web.use.https`, `edc.iam.accesstoken.jti.validation` were missing from the ConfigMap template — had to be set as raw env vars |
| Identity Hub | Accounts auth alias | `web.http.accounts.auth.alias` was missing from ConfigMap |

All fixes apply to both the standard and `-memory` chart variants.

## Critical Configuration Patterns

### 1. DID Resolution Without External TLS

The biggest deployment challenge is `did:web` resolution. By specification, `did:web` resolves via HTTPS, but in a single-cluster deployment this creates circular dependencies and TLS issues.

**Solution — four-part fix applied to ALL Java components:**

```yaml
# 1. hostAliases: resolve *.example.com to the cluster node IP
hostAliases:
  - ip: "10.10.20.35"
    hostnames:
      - "identity-hub.example.com"

# 2. Use HTTP instead of HTTPS for did:web
env:
  EDC_IAM_DID_WEB_USE_HTTPS: "false"

# 3. Ingress: don't redirect HTTP to HTTPS
annotations:
  nginx.ingress.kubernetes.io/ssl-redirect: "false"

# 4. Fallback: add CA certs to JVM truststore
customCaCerts:
  letsencrypt-r10.pem: |
    -----BEGIN CERTIFICATE-----
    ...
    -----END CERTIFICATE-----
```

### 2. STS Configuration (Single-Step Mode)

EDC has two STS flows. The DIM 2-step flow omits the `audience` claim, which breaks Identity Hub. The `dim.url` **must be empty**:

```yaml
iatp:
  sts:
    dim:
      url: ""              # MUST be empty — forces RemoteSecureTokenService
    oauth:
      token_url: "http://identityhub:8087/api/sts/token"
      client:
        id: "did:web:identity-hub.example.com:{BPN}"
        secret_alias: "sts-oauth-client-secret"
```

### 3. Credential Service URL

The base64 segment must encode the **BPN** (not the DID), because Identity Hub looks up participants by `participant_context_id`:

```yaml
env:
  TX_EDC_IAM_IATP_CREDENTIALSERVICE_URL: >-
    http://identityhub:8083/api/credentials/v1/participants/{base64(BPN)}
```

### 4. Verifiable Credentials

VCs must include Catena-X specific claims in `credentialSubject`:

- **MembershipCredential**: `memberOf: "Catena-X"`
- **DataExchangeGovernanceCredential**: `contractVersion`, `useCase`, `group`, `contractTemplate`

The `contractVersion` field is critical — `FrameworkAgreementConstraintFunction` checks for it during policy evaluation. Without it, contract negotiation fails.

VC `@context` must include `https://w3id.org/catenax/credentials/v1.0.0`.

### 5. Data Plane Token Signer

The data plane's `privatekey_alias` for token signing **must not contain `#`** characters. The Vault HTTP client treats `#` as a URL fragment and strips everything after it:

```yaml
# Bad — Vault lookup fails silently
privatekey_alias: "did:web:identity-hub.example.com:BPN#key-1"

# Good
privatekey_alias: "token-signer-key"
```

### 6. Participant Activation Workaround

Identity Hub v0.1.1 creates participants in state `CREATED` (1) even when `"active": true` is passed. Activation must be done via SQL:

```sql
UPDATE participant_context SET state = 2 WHERE participant_id = '{BPN}';
```

## E2E Data Exchange Flow

Once all components are deployed and configured, the full data exchange lifecycle works:

```
1. Catalog Request       → Consumer queries Provider's available datasets
2. Contract Negotiation  → Consumer agrees to Provider's access policy
3. Transfer Process      → Consumer initiates HttpData-PULL
4. EDR Retrieval         → Consumer gets data plane endpoint + access token
5. Data Access           → Consumer fetches actual data from Provider
```

### Step 1: Catalog Request

```bash
curl -X POST "http://edc-consumer:8081/management/v3/catalog/request" \
  -H "Content-Type: application/json" \
  -H "X-Api-Key: password" \
  -d '{
    "@context": {"@vocab": "https://w3id.org/edc/v0.0.1/ns/"},
    "counterPartyAddress": "http://edc-provider:8084/api/v1/dsp",
    "counterPartyId": "did:web:identity-hub.example.com:PROVIDER_BPN",
    "protocol": "dataspace-protocol-http"
  }'
```

> **Important**: `counterPartyId` is mandatory. Without the Provider's DID, the Consumer cannot set the `audience` claim in the STS token request.

### Step 2: Contract Negotiation

Extract the offer from the catalog and initiate negotiation. The policy must include `odrl:target` and `odrl:assigner` as JSON-LD `@id` references:

```bash
# Extract offer from catalog response, then:
NEGOTIATION_POLICY=$(echo "$OFFER" | jq \
  --arg target "$ASSET_ID" \
  --arg assigner "$PROVIDER_BPN" '
  . + {
    "@context": "http://www.w3.org/ns/odrl.jsonld",
    "odrl:target": {"@id": $target},
    "odrl:assigner": {"@id": $assigner}
  }')

curl -X POST "http://edc-consumer:8081/management/v3/contractnegotiations" \
  -H "Content-Type: application/json" \
  -H "X-Api-Key: password" \
  -d '{
    "@context": {"@vocab": "https://w3id.org/edc/v0.0.1/ns/"},
    "@type": "ContractRequest",
    "counterPartyAddress": "http://edc-provider:8084/api/v1/dsp",
    "counterPartyId": "did:web:identity-hub.example.com:PROVIDER_BPN",
    "protocol": "dataspace-protocol-http",
    "policy": ... // the negotiation policy above
  }'
```

Poll `GET /management/v3/contractnegotiations/{id}` until state is `FINALIZED`, then extract `contractAgreementId`.

### Step 3: Transfer Process

```bash
curl -X POST "http://edc-consumer:8081/management/v3/transferprocesses" \
  -H "Content-Type: application/json" \
  -H "X-Api-Key: password" \
  -d '{
    "@context": {"@vocab": "https://w3id.org/edc/v0.0.1/ns/"},
    "@type": "TransferRequest",
    "counterPartyAddress": "http://edc-provider:8084/api/v1/dsp",
    "counterPartyId": "did:web:identity-hub.example.com:PROVIDER_BPN",
    "protocol": "dataspace-protocol-http",
    "contractId": "CONTRACT_AGREEMENT_ID",
    "transferType": "HttpData-PULL"
  }'
```

Poll `GET /management/v3/transferprocesses/{id}` until state is `STARTED`.

### Step 4: EDR Retrieval

```bash
curl "http://edc-consumer:8081/management/v3/edrs/{transferProcessId}/dataaddress" \
  -H "X-Api-Key: password"
```

Returns `endpoint` (data plane URL) and `authorization` (bearer token).

### Step 5: Data Access

```bash
curl "$ENDPOINT" -H "Authorization: $AUTHORIZATION"
```

Returns the actual data from the Provider's backend data source.

## Issues Found During E2E Testing

| # | Issue | Impact | Resolution |
|---|-------|--------|------------|
| 1 | `hostAliases` missing from IH/Issuer Helm charts | Pods can't resolve DIDs internally | Added to deployment templates |
| 2 | ConfigMap missing app-level settings | Must set as env vars manually | Added `edc.ih.iam.id`, `didweb.https`, `jtivalidation` |
| 3 | HPA references wrong metric variable | Memory autoscaling uses CPU target | Fixed variable name |
| 4 | PostgreSQL CPU limit uses memory unit | Invalid resource spec | Changed `500Mi` to `500m` |
| 5 | `dim.url` must be empty for STS | DIM 2-step omits audience claim | Set to empty string |
| 6 | `counterPartyId` required in catalog request | Consumer can't determine STS audience | Must pass Provider DID |
| 7 | Credential `@context` must be Catena-X specific | VC validation fails | Use `w3id.org/catenax/credentials/v1.0.0` |
| 8 | `contractVersion` required in VC claims | FrameworkAgreement policy rejects negotiation | Add to credentialSubject |
| 9 | Vault alias with `#` breaks lookup | Data plane can't sign tokens | Use simple alias name |
| 10 | Participant activation bug in IH v0.1.1 | Participants stuck in CREATED state | Activate via SQL |
| 11 | Issuer Service log path is wrong | Logs go to `identityhub.log` | Fixed to `issuerservice.log` |

## Repository

The full deployment configuration (Helm values, scripts, ingress) is maintained at:
https://github.com/felipebustillo/tractus-x

---

## NOTICE

This work is licensed under the [CC-BY-4.0](https://creativecommons.org/licenses/by/4.0/legalcode).

* SPDX-License-Identifier: CC-BY-4.0
* SPDX-FileCopyrightText: 2026 Contributors to the Eclipse Foundation
* Source URL: <https://github.com/eclipse-tractusx/tractusx-identityhub>
