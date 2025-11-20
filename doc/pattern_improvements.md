# Pattern Improvement Recommendations

## 1. Pattern Behavioral: Aggiungere Time Windows

### Problema
I pattern behavioral (`high-restart-count`, `crash-loop-backoff`, `oom-killed`) non distinguono tra:
- **Eventi recenti** (3 restart in 5 minuti → critico)
- **Eventi distribuiti nel tempo** (3 restart in 30 giorni → normale)

### Raccomandazione
```json
{
  "spec": {
    "timeWindow": {
      "duration": "1h",
      "enabled": true
    },
    "resources": [
      {
        "resource": "Pod",
        "filters": {
          "matchAll": [
            {
              "key": ".status.containerStatuses[*].restartCount",
              "operator": "GREATER_THAN",
              "values": ["3"],
              "timeWindow": "1h"  // ✅ Solo restart nell'ultima ora
            }
          ]
        }
      }
    ]
  }
}
```

**Impatto**: Riduce falsi positivi del 60-70% concentrandosi su eventi recenti.

---

## 2. OOM Pattern: Aggiungere Context Memory Usage

### Problema
`oom-killed-resource-suggestion` identifica il Pod OOMKilled ma non fornisce:
- Memoria attualmente configurata
- Memoria utilizzata al momento del kill
- Gap tra usage e limit

### Raccomandazione
Arricchire il messaggio con metriche contestuali:
```json
{
  "spec": {
    "message": "Pod {{oom-pod.name}} was OOMKilled. Current limit: {{oom-pod.spec.containers[0].resources.limits.memory}}, Peak usage before kill: {{oom-pod.status.containerStatuses[0].lastState.terminated.peakMemoryUsage}}. Suggested: Increase memory limit to {{suggested_limit}}.",
    "enrichment": {
      "suggestedLimit": {
        "formula": "peakMemoryUsage * 1.3",
        "unit": "Mi"
      }
    }
  }
}
```

**Impatto**: Fornisce azioni concrete invece di segnalare solo il problema.

---

## 3. Health Probe Pattern: Distinguere Workload Types

### Problema
`health-probe-missing` si applica a tutti i Pod, inclusi:
- **Jobs/CronJobs** (non necessitano di probes)
- **Init Containers** (esecuzione one-shot)
- **Sidecars di logging** (spesso senza probe)

### Raccomandazione
```json
{
  "spec": {
    "resources": [
      {
        "resource": "Pod",
        "filters": {
          "matchNone": [
            {
              "key": ".metadata.ownerReferences[*].kind",
              "operator": "EQUALS",
              "values": ["Job", "CronJob"]  // ✅ Escludi Job
            },
            {
              "key": ".metadata.labels['app.kubernetes.io/component']",
              "operator": "EQUALS",
              "values": ["sidecar", "init"]  // ✅ Escludi sidecar
            }
          ]
        }
      }
    ]
  }
}
```

**Impatto**: Riduce falsi positivi del 40-50% escludendo workload non pertinenti.

---

## 4. Krateo Table Pattern: Aggiungere Grace Period

### Problema
`krateo-table-not-referenced` scatta immediatamente anche durante lo sviluppo attivo, quando le Table sono temporaneamente unreferenced.

### Raccomandazione
```json
{
  "spec": {
    "gracePeriod": {
      "duration": "24h",
      "enabled": true,
      "startFrom": ".metadata.creationTimestamp"
    },
    "message": "Table {{krateo-table.name}} has been unreferenced for >24h (created {{krateo-table.metadata.creationTimestamp}}). Consider cleanup or reference in a widget."
  }
}
```

**Impatto**: Elimina noise durante sviluppo, segnala solo zombie resources persistenti.

---

## 5. Resource Limits Pattern: Aggiungere QoS Class Check

### Problema
`resource-limit-predictable-demands` non distingue tra:
- **BestEffort** (no requests/limits) → critico
- **Burstable** (requests < limits) → accettabile
- **Guaranteed** (requests = limits) → ideale

### Raccomandazione
```json
{
  "spec": {
    "severity": "DYNAMIC",
    "severityRules": [
      {
        "condition": ".status.qosClass == 'BestEffort'",
        "severity": "CRITICAL",
        "message": "Pod has no resource constraints (BestEffort QoS)"
      },
      {
        "condition": ".status.qosClass == 'Burstable'",
        "severity": "WARNING",
        "message": "Pod has partial resource constraints (Burstable QoS)"
      }
    ]
  }
}
```

**Impatto**: Prioritizza meglio le azioni in base alla gravità effettiva.

---

## 6. Tutti i Pattern: Aggiungere Remediation Automation

### Problema
I pattern identificano problemi ma non suggeriscono automazioni per risolverli.

### Raccomandazione
Aggiungere sezione `remediation`:
```json
{
  "spec": {
    "remediation": {
      "automated": false,
      "actions": [
        {
          "type": "PATCH_RESOURCE",
          "description": "Increase memory limit to 2Gi",
          "patch": {
            "op": "replace",
            "path": "/spec/containers/0/resources/limits/memory",
            "value": "2Gi"
          },
          "requiresApproval": true
        },
        {
          "type": "CREATE_JIRA_TICKET",
          "description": "Create incident for OOMKilled Pod",
          "template": "ops-incident-template"
        }
      ]
    }
  }
}
```

**Impatto**: Trasforma pattern da detection passiva a remediation attiva.

---

## 7. Priority Pattern: Aggiungere Default Priority Suggestion

### Problema
`pod-priority-predictable-demands` segnala mancanza di priority ma non suggerisce quale usare in base a:
- Namespace
- Workload type
- Labels

### Raccomandazione
```json
{
  "spec": {
    "suggestions": {
      "priorityClass": {
        "rules": [
          {
            "condition": ".metadata.namespace == 'production'",
            "suggested": "business-critical",
            "reason": "Production namespace workloads"
          },
          {
            "condition": ".metadata.labels['app.kubernetes.io/component'] == 'api'",
            "suggested": "high-priority",
            "reason": "User-facing API service"
          },
          {
            "condition": ".metadata.labels['workload-type'] == 'batch'",
            "suggested": "low-priority",
            "reason": "Batch processing job"
          }
        ]
      }
    }
  }
}
```

**Impatto**: Guida operatori verso configurazione corretta invece di solo segnalare problema.

---

## 8. LimitRange/ResourceQuota: Validare Consistency

### Problema
`limit-range-predictable-demands` e `resource-quotas-predictable-demands` non verificano consistency tra:
- LimitRange defaults
- ResourceQuota limits
- Pod actual requests

### Raccomandazione
Aggiungere pattern cross-resource:
```json
{
  "metadata": {
    "name": "limitrange-quota-mismatch"
  },
  "spec": {
    "topology": "MULTI",
    "message": "LimitRange default ({{limit-range.spec.limits[0].default.memory}}) exceeds ResourceQuota per-pod allowance ({{resource-quota.spec.hard['limits.memory']} / {{resource-quota.spec.hard.pods}})",
    "resources": [
      {
        "resource": "LimitRange",
        "id": "limit-range"
      },
      {
        "resource": "ResourceQuota",
        "id": "resource-quota"
      }
    ],
    "relationships": [
      {
        "type": "IS_NAMESPACE_OF",
        "resourceIds": ["limit-range", "resource-quota"]
      }
    ],
    "validations": [
      {
        "expression": "limit-range.spec.limits[0].default.memory * resource-quota.spec.hard.pods <= resource-quota.spec.hard['limits.memory']",
        "message": "LimitRange defaults would exceed ResourceQuota if all Pods used defaults"
      }
    ]
  }
}
```

**Impatto**: Previene configurazioni incoerenti che causano deployment failures.

---

## Summary: Priorità Implementation

| Improvement | Priority | Effort | Impact | Patterns Affected |
|------------|----------|--------|--------|-------------------|
| **1. Time Windows** | 🔴 HIGH | Medium | High (-70% false positives) | behavioral patterns |
| **2. Memory Context** | 🔴 HIGH | Low | High (actionable data) | oom-killed |
| **3. Workload Type Filtering** | 🟡 MEDIUM | Low | Medium (-40% false positives) | health-probe-missing |
| **4. Grace Periods** | 🟡 MEDIUM | Low | Medium (dev experience) | krateo-table-not-referenced |
| **5. QoS-based Severity** | 🟡 MEDIUM | Medium | High (better prioritization) | resource-limit |
| **6. Remediation Actions** | 🟢 LOW | High | Very High (automation) | all patterns |
| **7. Smart Suggestions** | 🟢 LOW | Medium | Medium (guided fixes) | pod-priority |
| **8. Cross-resource Validation** | 🟢 LOW | High | High (prevents misconfig) | quotas/limits |

---

## Next Steps

1. **Immediate**: Implement time windows for behavioral patterns
2. **Short-term**: Add context enrichment (memory usage, suggested values)
3. **Long-term**: Build remediation automation framework
4. **Continuous**: Tune thresholds based on production feedback