# Threat Intelligence: IP Reputation Enrichment

**Status:** Documented. **Not implemented, not validated.**

## 1. Detection objective

Automatically enrich alerts containing an external IP address with reputation and ownership context, so the analyst begins triage with the context already attached rather than spending the first ten minutes on lookups.

**Enrichment informs a decision; it does not make one.** This is the central discipline of this document.

## 2. Data source

| Source | Provides |
|---|---|
| Public IP reputation services | Reputation score, report history, category |
| Geolocation and ASN lookup | Country, network owner |
| Local lab history | Whether this address has been seen before |

API keys for any intelligence source are held in environment variables and **never committed**. See [`../../SECURITY.md`](../../SECURITY.md).

## 3. Enrichment logic

Triggered by Shuffle on any alert containing an external IP:

```
Alert → extract IP → validate it is external and routable
     → skip RFC1918, loopback, and known-good lab addresses
     → query reputation service
     → query geolocation / ASN
     → check local history
     → attach result to alert → present to analyst
```

Private, reserved and lab-internal addresses are filtered before any external query — both to avoid pointless API calls and to avoid disclosing internal addressing to a third-party service.

## 4. Expected output

```
IP:            <address>
Reputation:    <score / verdict>
Reports:       <count>, most recent <date>
Category:      <e.g. scanning, brute force, C2 — as reported>
Country / ASN: <country> / <network owner>
Seen in lab:   Yes / No  (first seen <date>)
Source:        <service>, queried <timestamp> UTC
```

The query timestamp matters. Reputation changes; an assessment is only valid as of when it was made.

## 5. Analyst use

Reputation **adjusts confidence**; it never determines the outcome.

| Situation | Correct reading |
|---|---|
| Poor reputation + suspicious behaviour | Confidence raised. Investigate as likely malicious. |
| Poor reputation, no corroborating behaviour | Note it. Not an incident by itself. |
| Clean reputation + clearly suspicious behaviour | **Trust the behaviour.** New infrastructure has no history. |
| Clean reputation, no suspicious behaviour | Routine. |

**A clean reputation is not exoneration.** Attacker infrastructure is often newly provisioned and unreported. Behaviour observed in your own environment is stronger evidence than any external feed.

## 6. Severity considerations

Reputation may raise severity by one level where it corroborates observed behaviour. It should not, on its own, raise an alert to High or drive a blocking action.

## 7. False-positive considerations

Reputation data is noisy, and understanding why prevents bad decisions:

- **Shared hosting and cloud providers** — one abusive tenant taints an address that later serves legitimate services
- **NAT and CGNAT** — an entire ISP's customer base behind one address; a single bad actor tarnishes it for everyone
- **Stale reports** — an address reported two years ago may have changed hands repeatedly
- **VPN and Tor exit nodes** — frequently flagged, routinely used legitimately
- **Security scanners and research crawlers** — reported constantly, benign
- **Geolocation inaccuracy** — country attribution is approximate and trivially defeated by a VPN

Country of origin is the weakest indicator available. Treating it as evidence produces confident, wrong conclusions.

## 8. Response

| Action | Approval |
|---|---|
| Attach enrichment to alert | None — automatic |
| Record in investigation notes | None |
| Use to adjust analyst confidence | None |
| **Block the address** | **Required** |
| **Add to a permanent blocklist** | **Required** |

**Automatic blocking on reputation alone is explicitly not implemented, and that is a design decision, not an omission.** A single reputation hit against a shared or NAT address can block a large number of legitimate users. The approval gate exists precisely for this.

## 9. Validation

- Query a known-benign address (a public DNS resolver) — expect clean.
- Query an address from a public test list — expect a reported result.
- Query an RFC1918 address — expect it to be filtered before any external query.
- Disable the API key — **confirm enrichment fails gracefully and the alert still reaches the analyst.**

That last case matters most: enrichment failing must never suppress an alert. An enrichment dependency that silently swallows alerts is worse than no enrichment.

Recorded in [`../testing/test-cases.md`](../testing/test-cases.md).

## 10. Evidence to capture

The address · full enrichment response · service queried and query timestamp · the alert it was attached to · the analyst's assessment and reasoning.

Record the reasoning, not only the verdict. A future analyst needs to know *why* the call was made.

## 11. Operational notes

- Respect API rate limits; cache results for a short period
- Handle service unavailability gracefully — never block the alert pipeline on an external dependency
- Do not submit internal addresses, hostnames or any client-identifying data to a third-party service
- Review source quality periodically; a feed producing consistently misleading results is worse than none
