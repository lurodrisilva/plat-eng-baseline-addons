# Platform origin certificate alerting

ADR-0026 accepted a certificate whose renewal budget is not ours, and made
alerting a **condition** of that acceptance rather than a follow-up. This
document is the Azure half; the cluster half is `templates/certcheck.yaml`.

## Why this is not routine monitoring

Entra refuses a non-localhost `http` redirect URI, so the certificate is in the
**sign-in path**. A failed renewal does not degrade the portal — it takes
sign-in **down**. And because `cloudapp.azure.com` is absent from the Public
Suffix List, Let's Encrypt scopes the per-registered-domain budget to
`azure.com`, shared with every Azure tenant, with an override only Microsoft can
raise. So the renewal can fail for reasons that have nothing to do with this
platform, and the only useful response is to notice early: Let's Encrypt renews
at two-thirds of lifetime, giving roughly a **30-day** window in which the
recovery is waiting rather than acting.

## The split

The CronJob **reports**; the alert rules **decide**. Every threshold lives in
the KQL below, so changing "warn at 21 days" is an alert-rule edit, not a chart
release, and there is exactly one place to read the current policy from.

Each run emits one line:

```text
PLATFORM_ORIGIN_CERT_CHECK name=platform-origin-tls ready=True notAfter=2026-10-25T12:01:04Z
```

That prefix is the contract between the chart and the rules. Changing it in one
place without the other **turns the alerting off silently** — which is the exact
failure this document exists to prevent.

### Which table, and why the query unions two of them

`aks-test` writes the **legacy `ContainerLog`** table, whose message column is
`LogEntry` (a string). `ContainerLogV2` exists in the schema but holds **0
rows** here. AKS is migrating clusters to V2, where the column is `LogMessage`
and is **dynamic**, not string — passing it straight to `extract()` fails with
`extract(): argument #3 expected to be a string expression`.

Both queries therefore union the two and normalise to one `Msg` column. That
costs nothing today and means the eventual V2 migration does not silently take
the alerting offline, which is precisely the class of failure this whole slice
exists to prevent.

## Rule 1 — the certificate is unhealthy or expiring

`platform-origin-cert-unhealthy`, severity 1.

Fires when the Certificate is not Ready, or expires within 21 days. 21 sits
*inside* the ~30-day renewal window: an alert at 7 days would leave no room to
do anything but watch.

```kusto
union isfuzzy=true (ContainerLog | extend Msg = tostring(LogEntry)), (ContainerLogV2 | extend Msg = tostring(LogMessage))
| where Msg has "PLATFORM_ORIGIN_CERT_CHECK"
| extend ready    = extract(@"ready=(\S+)", 1, Msg)
| extend notAfter = todatetime(extract(@"notAfter=(\S+)", 1, Msg))
| extend daysLeft = datetime_diff('day', notAfter, now())
| where ready != "True" or daysLeft < 21
| project TimeGenerated, ready, notAfter, daysLeft
```

Condition `count > 0`, evaluated every 6 hours over a 24-hour window.

## Rule 2 — the check itself stopped running

`platform-origin-cert-check-silent`, severity 2.

This is the one that matters most and the one most easily left out. Rule 1 can
only fire on a line that exists; if the CronJob stops being scheduled, is
pruned, or its image stops pulling, Rule 1 goes quiet — and quiet is
indistinguishable from healthy. **A detector that fails silently is worse than
no detector**, because it also removes the suspicion that would otherwise prompt
someone to look by hand.

```kusto
union isfuzzy=true (ContainerLog | extend Msg = tostring(LogEntry)), (ContainerLogV2 | extend Msg = tostring(LogMessage))
| where Msg has "PLATFORM_ORIGIN_CERT_CHECK"
| summarize heartbeats = count()
```

Condition **`min 'heartbeats' < 1`** — a metric measurement, not a row count.

The obvious form, appending `| where heartbeats == 0` and alerting on "results
> 0", was tried first and **returns no rows at all** through this union, so the
rule would never have fired — the exact silent failure it is meant to catch.
`summarize` with no `by` always emits one row, so measuring the value is
reliable where counting rows is not. Do not "simplify" this back.

With a 6-hourly schedule, a 24-hour window tolerates one missed run before
firing.

## Provisioning

Created with `az monitor action-group create` (action group
`platform-origin-alerts`, email) and `az monitor scheduled-query create` against
the Log Analytics workspace Container Insights already wires to the cluster
(`aks-test-workspace` in `aks-test-rg`).

**Not** in Terraform: `plat-eng-aks-foundation` has no alerting module, and
inventing one for two rules would put the origin's alerting in a repo whose
`terraform apply` is not automated — so it would look managed while nothing
reconciled it. Recorded here to be reproducible rather than folklore.

## Verifying it end to end

The rules cannot be proven by the absence of a page. To check the pipeline is
actually connected, run the job by hand and confirm the line reaches the
workspace:

```bash
kubectl create job -n control-plane-system certcheck-manual \
  --from=cronjob/platform-origin-cert-check
kubectl logs -n control-plane-system job/certcheck-manual
# PLATFORM_ORIGIN_CERT_CHECK name=platform-origin-tls ready=True notAfter=...

# then, allowing a few minutes for ingestion:
az monitor log-analytics query -w <customerId> --analytics-query \
  'ContainerLog | where LogEntry has "PLATFORM_ORIGIN_CERT_CHECK" | take 5'
```

If the second command returns nothing while the first prints the line, the alert
rules are decorative.

## What is deliberately not alerted

- **The public endpoint's reachability.** That belongs to an availability test
  against the origin, and until P5/P6 attach routes the origin correctly answers
  404, so such a test would need to assert a status that is about to change.
- **cert-manager pod health.** A broken controller shows up here as a
  Certificate that stops being Ready, which is the symptom that actually
  matters; alerting on both would fire twice for one fault.
