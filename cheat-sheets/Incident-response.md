
# 🚨 Incident Response & Blameless Post-Mortem Template

Use this template to structure on-call incident triage and post-incident analysis.

---

## 1. Incident Summary
* **Incident Title:** 
* **Severity Level:** (SEV-1 / SEV-2 / SEV-3)
* **Incident Commander:** 
* **Communication Lead:** 
* **Impacted Service(s):** 
* **User Impact:** (e.g., "15% of checkout API calls failed with HTTP 500")
* **Time to Detect (MTTD):** 
* **Time to Mitigate (MTTR):** 

---

## 2. Timeline (UTC)
* `HH:MM` - Automated alert triggered in Datadog/Prometheus.
* `HH:MM` - Incident Commander page acknowledged.
* `HH:MM` - Initial triage identified database connection pool exhaustion.
* `HH:MM` - Mitigation applied (scaled read replicas, increased pool ceiling).
* `HH:MM` - Metrics normalized; error budget burn stopped.
* `HH:MM` - Incident officially resolved.

---

## 3. Root Cause Analysis (5 Whys)
1. **Why did the service fail?** Database connections were exhausted.
2. **Why were connections exhausted?** A new API deployment opened connections without closing them.
3. **Why were they not closed?** A newly merged pull request bypassed connection pooling logic.
4. **Why did tests pass?** Integration tests simulated low concurrency where pools did not leak.
5. **Why was there no canary gate?** Automated canary analysis was not enabled for this service pipeline.

---

## 4. Preventative Action Items
| Action Item | Type | Owner | Target Date |
| :--- | :--- | :--- | :--- |
| Enforce connection timeout limits in client configuration | Preventative | DevOps Team | YYYY-MM-DD |
| Add concurrency load testing to CI stage | Detection | QA / SRE | YYYY-MM-DD |
| Configure automated canary rollback on 5xx spike | Mitigation | SRE Genius | YYYY-MM-DD |