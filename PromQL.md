## PromQL

### Prometheus data model

Tout ce que vous devez savoir est très bien expliqué dans la [Prometheus documentation](https://prometheus.io/docs/concepts/data_model/).

À la base, le modèle de données représente les séries chronologiques
sous la forme d'un nom de métrique suivi d'un ensemble de noms et de
valeurs d'étiquette. Les échantillons sont des valeurs numériques
associées à une série temporelle à un horodatage donné.

Voici la représentation d'une métrique avec différentes étiquettes:
```
process_resident_memory_bytes{instance="127.0.0.1:9100",job="node_exporter"}
process_resident_memory_bytes{instance="127.0.0.1:9090",job="prometheus"}
```

Le type d'une métrique peut être l'un des suivants:
* Counter (`node_cpu_seconds_total`)
* Gauge (`node_memory_MemFree_bytes`)
* Histogram (`prometheus_http_request_duration_seconds`)
* Summary (`go_gc_duration_seconds`)

L'histogramme et le Summary sont liés et offrent différents compromis. La principale différence du point de vue de l'utilisateur est que les histogrammes peuvent être agrégés alors que les Summary ne le peuvent pas (en général).

*Exercice: visitez à nouveau le point de terminaison des métriques de Prometheus et vérifiez les différents types de métriques.*

### Prometheus query language (PromQL)

PromQL est un langage puissant qui permet de découper et de regrouper les données selon les besoins.

Une expression peut vous obtenir toutes les séries temporelles pour un nom de métrique donné:

```
node_cpu_seconds_total
```

Ajoutez un sélecteur d'étiquettes pour filtrer un processeur particulier:
```
node_cpu_seconds_total{cpu="0"}
```
Mais obtenir le nombre total de secondes qu'un processeur a passé dans
chaque mode n'est pas vraiment significatif.
Et si vous souhaitez obtenir le pourcentage de temps passé?
C'est là que nous devons introduire des vecteurs et
des fonctions de plage (`rate ()` en particulier).

Jusqu'à présent, nous n'avons utilisé que des sélecteurs de
vecteurs instantanés qui renvoient un seul échantillon pour chaque
série temporelle correspondant aux sélecteurs d'étiquettes donnés.
Avec les sélecteurs de vecteurs de plage, Prometheus renvoie la liste
des échantillons dans la plage de temps donnée pour chaque série de temps.

```
node_cpu_seconds_total{cpu="0"}[5m]
```

Encore une fois, cette expression n'est pas très utile en elle-même
mais c'est là que `rate ()` vient à la sauver:

```
rate(node_cpu_seconds_total{cpu="0"}[5m])
```

*Exercice: écrivez une requête qui renvoie le pourcentage de temps processeur inactif (indice: PromQL prend en charge les opérateurs arithmétiques).*

<details>
  <summary>Solution</summary>

```
100 * rate(node_cpu_seconds_total{cpu="0",mode="idle"}[5m])
```
</details>

### More examples

Vous pouvez agréger les valeurs métriques par dimensions arbitraires
en utilisant «by» ou «without»:
```
sum(node_scrape_collector_duration_seconds) without (collector)
```

Ces opérateurs d'agrégation sont familiers si vous connaissez déjà SQL.
PromQL a également `min()`, `max()`, `avg()`, `count()` et [plus](https://prometheus.io/docs/prometheus/latest/querying/operators/# opérateurs d'agrégation).

*Exercice: écrivez une requête qui retourne les 5 collecteurs prenant le plus de temps lors des grattages.*

<details>
  <summary>Solution</summary>

```
topk(5, node_scrape_collector_duration_seconds)
```
</details>

PromQL prend également en charge la correspondance vectorielle pour
les opérations binaires et arithméthiques.
Allons-y générer des requêtes invalides à Prometheus:
```
for i in {1..10}; do \
  curl localhost:9090/api/v1/query; \
  curl localhost:9090/static/notfound; \
  sleep 5
done
```

Et maintenant, nous pouvons demander à Prometheus le pourcentage de requêtes HTTP qui ont renvoyé un code d'état 400.
```
100 * sum(rate(prometheus_http_requests_total{code="400"}[5m])) / sum(rate(prometheus_http_requests_total[5m]))
```

*Exercice: modifiez la requête pour calculer le pourcentage de requêtes HTTP ayant renvoyé un code d'état entre 400 et 499.*

<details>
  <summary>Solution</summary>

```
100 * sum(rate(prometheus_http_requests_total{code=~"4.."}[5m])) / sum(rate(prometheus_http_requests_total[5m]))
```
</details>

Calculons maintenant [quantiles](https://en.wikipedia.org/wiki/Quantile).
La méthode dépend du fait que la métrique est un summary ou un histogramme.
Les summary peuvent être reconnus par leur étiquette `quantile`
tandis que les métriques d'histogramme ont une étiquette `le` qui représente le compartiment de l'histogramme (le = inférieur ou égal).

Voici un exemple de métrique récapitulative.

```
go_gc_duration_seconds
```

Prometheus expose des métriques d'histogramme mesurant la taille des réponses HTTP.
```
prometheus_http_response_size_bytes_bucket{handler="/api/v1/query"}
```

La fonction `histogram_quantile()` appliquée aux métriques d'histogramme renvoie une estimation du n-ième quantile.

```
histogram_quantile(0.9, rate(prometheus_http_response_size_bytes_bucket{handler="/api/v1/query"}[5m]))
```

Les summary et les histogrammes suivent également la somme et le nombre d'échantillons observés qui peuvent être utilisés pour calculer les valeurs moyennes:
```
sum(rate(prometheus_http_request_duration_seconds_sum[5m])) by (job, handler)
 /
sum(rate(prometheus_http_request_duration_seconds_count[5m])) by (job, handler)
```

*Exercice: excluez les valeurs NaN.*

<details>
  <summary>Solution</summary>

```
sum(rate(prometheus_http_request_duration_seconds_sum[5m])) by (job, handler)
 /
( sum(rate(prometheus_http_request_duration_seconds_count[5m])) by (job, handler) > 0 )

```
</details>
