# Resource Limit Predictable Demands
## Intro

- **Version**: `kubepattern.it/v1`  
- **id**: `resource-limit-predictable-demands`  
- **Name**: *Resource Limit Predictable Demands*  
- **Type**: `FOUNDATIONAL`  
- **severity**: `WARNING`  
- **category**: `architecture`  
- **docUrl**: [Kubernetes — Resource limits](https://kubernetes.io/docs/concepts/workloads/pods/resource-limit-predictable-demands/)  
- **gitUrl**: [Repository pattern](https://github.com/GabrieleGroppo/kubepattern-registry/tree/main/doc/resource-limit-pattern.md)

## Problem
Omettere i **resource limits** (CPU e memoria) sui Pod Kubernetes introduce seri rischi di stabilità e imprevedibilità nel cluster. Senza vincoli, un Pod può **consumare risorse in modo incontrollato**: può saturare la memoria del nodo, innescando OOMKill o l'eviction di altri workload (come nel caso di un *memory leak*), oppure monopolizzare la CPU, causando degrado delle prestazioni e latenza per le altre applicazioni co-locate (effetto *noisy neighbour*). Questa indeterminatezza rende il comportamento delle applicazioni **imprevedibile**, complica enormemente il *capacity planning* e rende fragile l'intero bilanciamento del cluster. Di conseguenza, Kubernetes assegna a questi Pod una classe **QoS inferiore** (*Burstable* o *BestEffort*), rendendoli i primi a essere terminati in caso di pressione sulle risorse. Tale comportamento erratico, infine, compromette l'efficacia dell'**autoscaling** e può generare picchi di costo incontrollati negli ambienti cloud.

## Solution

La soluzione consiste nell'impostare sistematicamente **`requests`** e **`limits`** di risorse (CPU e memoria) per ogni container all'interno della specifica del Pod.

  * **`requests`**: Specifica la quantità di risorse *minima garantita* al container. Questo valore è cruciale per lo **scheduler** di Kubernetes, che lo usa per decidere su quale nodo posizionare il Pod, assicurando che il nodo abbia capacità sufficiente.
  * **`limits`**: Specifica il *tetto massimo* di risorse che il container può consumare. Se un container supera il limite di memoria, viene terminato (OOMKilled). Se supera il limite di CPU, viene "throttled" (rallentato), ma non terminato.

Questo approccio trasforma i workload da imprevedibili a prevedibili, permettendo a Kubernetes di gestirli efficacemente e di proteggere gli altri workload.

L'impostazione di `requests` e `limits` determina la **Classe QoS (Quality of Service)** del Pod:

1.  **Guaranteed**: Impostando `limits` e `requests` (sia per CPU che per memoria) allo **stesso valore**. Questa è la configurazione ideale per "demands prevedibili", in quanto il Pod riceve la massima priorità e viene terminato solo se supera i suoi stessi limiti, e comunque per ultimo in caso di pressione sul nodo.
2.  **Burstable**: Impostando `limits` **superiori** ai `requests`. Il Pod ha una garanzia di base (`requests`) ma può "sforare" (`burst`) fino al `limits` se le risorse del nodo sono disponibili. Questa è una configurazione comune e molto più stabile di `BestEffort`.
3.  **BestEffort**: Omettendo del tutto `requests` e `limits`. Questa è la configurazione che il pattern intende prevenire.

Un'implementazione concreta (idealmente `Guaranteed QoS`):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: predictable-pod
spec:
  containers:
  - name: my-container
    image: nginx
    resources:
      requests:
        memory: "256Mi"
        cpu: "500m"  # 0.5 CPU core
      limits:
        memory: "256Mi"
        cpu: "500m"
```

## Involved resources

I seguenti oggetti Kubernetes sono direttamente coinvolti nell'implementazione o nella gestione di questo pattern:

- **`Pod`**: La risorsa fondamentale dove vengono definite le specifiche del container.

## Metrics

Per identificare i Pod che *non* seguono questo pattern (e che quindi rappresentano un rischio), è necessario ispezionare i campi relativi alle risorse.

### Fields to Match

Il pattern è violato se, per qualsiasi `Container` all'interno di un `Pod`, uno o più dei seguenti campi **non** sono definiti (sono `null` o assenti):

  * `spec.containers[].resources.limits.cpu`
  * `spec.containers[].resources.limits.memory`

Sebbene l'assenza di `requests` sia anch'essa problematica (porta a QoS `BestEffort` anche se i `limits` sono presenti), l'assenza di `limits` è la violazione primaria che causa instabilità da "noisy neighbour".