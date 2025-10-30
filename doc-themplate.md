
-----

# `[Nome del Pattern]`

## ℹ️ Intro

  * **Version**: `kubepattern.it/v1`
  * **id**: `[id-del-pattern-es: resource-limit]`
  * **Name**: `[Nome leggibile del pattern, es: Resource Limit Predictable Demands]`
  * **Type**: `[Tipologia, es: FOUNDATIONAL, BEHAVIORAL, STRUCTURAL]`
  * **Severity**: `[Gravità della violazione, es: CRITICAL, WARNING, INFO]`
  * **Category**: `[Area di impatto, es: architecture, security, reliability, cost-optimization]`
  * **docUrl**: `https://kubernetes.io/docs/home/`
  * **gitUrl**: `https://learn.microsoft.com/it-it/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/infrastructure-persistence-layer-design`

-----

## ❗ Problema (Perché)

[Descrivere in modo chiaro e conciso il problema che questo pattern risolve. Qual è l'**anti-pattern** (il modo sbagliato di fare le cose)? Quali sono i **rischi concreti** (es. instabilità, costi imprevisti, falle di sicurezza) se questo pattern non viene applicato?

Usare esempi specifici, come "memory leak", "noisy neighbour" o "privilege escalation", per rendere il rischio tangibile. L'obiettivo di questa sezione è rispondere alla domanda: "Perché dovrei preoccuparmi di questo?"]

-----

## ✅ Soluzione (Come)

[Descrivere la soluzione tecnica e concettuale. Spiegare i componenti chiave della soluzione (es. cosa sono `requests` e `limits`, o cos'è un `NetworkPolicy`).

Spiegare *perché* questa soluzione mitiga il problema descritto in precedenza. Se rilevante, spiegare concetti correlati che la soluzione abilita o influenza (es. Classi QoS, principio del "least privilege", ecc.).]

-----

## 🛠️ Esempio di Implementazione

[Fornire uno o più blocchi di codice YAML minimi ma completi che dimostrano l'implementazione **corretta** del pattern. Questo serve come riferimento "copia-e-incolla" per l'utente.

Assicurarsi di commentare le righe chiave che implementano il pattern.]

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-con-pattern-corretto
spec:
  containers:
  - name: mio-container
    image: nginx
    # Esempio di commento: queste righe implementano il pattern
    resources:
      requests:
        memory: "256Mi"
        cpu: "500m"
      limits:
        memory: "256Mi"
        cpu: "500m"
```

-----

## 🔗 Risorse Coinvolte

[Elencare gli oggetti API di Kubernetes rilevanti per questo pattern e descrivere brevemente il loro ruolo specifico.]

  * **`Kind`**: [Breve descrizione del suo ruolo nel pattern (es. `Pod`: la risorsa dove il pattern viene applicato).]
  * **`Kind`**: [Breve descrizione (es. `LimitRange`: una policy che può imporre questo pattern a livello di Namespace).]
  * **`Kind`**: [Breve descrizione (es. `Deployment`: il controller che gestisce i Pod a cui applicare il pattern).]

-----

## 📊 Metriche di Violazione (Come trovarlo)

[Spiegare come identificare programmaticamente le risorse che *violano* questo pattern (ovvero, che implementano l'anti-pattern). Questo è cruciale per gli strumenti di scansione.]

### Fields to Match

[Elencare i campi YAML specifici, usando la notazione `path.to.field`, che devono essere ispezionati per rilevare la violazione.]

Il pattern è violato se, per qualsiasi container, uno o più dei seguenti campi **non** sono definiti (sono `null` o assenti):

  * `spec.containers[].resources.limits.cpu`
  * `spec.containers[].resources.limits.memory`

### Logica di Rilevamento (Opzionale)

[Se la logica è più complessa del semplice controllo di un campo nullo (es. richiede il confronto di più campi), descriverla qui.]

  * Esempio: "La violazione si verifica SE `spec.containers[].securityContext.privileged` è `true`."
  * Esempio: "La violazione si verifica SE `spec.containers[].resources.requests.cpu` \> `spec.containers[].resources.limits.cpu`."

-----

## ✨ Benefici e Considerazioni

[Riassumere i vantaggi e gli eventuali "trade-off" da considerare prima di applicare ciecamente il pattern.]

### Benefici

  * [Beneficio 1, es: Migliore stabilità e prevedibilità del cluster.]
  * [Beneficio 2, es: Riduzione dei costi grazie a un bin-packing più efficiente.]
  * [Beneficio 3, es: Maggiore sicurezza e isolamento dei workload.]

### Considerazioni (Trade-offs)

  * [Considerazione 1, es: Richiede un'analisi di "rightsizing" delle applicazioni; impostare limiti troppo bassi può causare OOMKilled.]
  * [Considerazione 2, es: Può essere complesso da applicare a workload legacy o di cui non si conosce il profilo di consumo.]