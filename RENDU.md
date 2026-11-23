# Rendu — Atelier Grafana

Nom / Prénom : SARAHLOUANES

Groupe : RS2

## Étape 2 — cAdvisor

Q1. À quoi ressemble une métrique Prometheus ? Repérez le nom, les labels et la valeur.

Ligne relevée sur cAdvisor :

```
container_memory_usage_bytes{id="/docker/c5c25c2053cf3e344b31fddd67e8e1630fe8b52f2d9c0f1f8be20c76d9a44d8c",image="quay.io/ocp-edge-qe/httpd",name="web1"} 7.20896e+06 1795442167882
```

Le nom est container_memory_usage_bytes : la mémoire utilisée, en octets. Les labels sont entre accolades (id, image, name="web1") : ils précisent de quel conteneur il s'agit et servent à filtrer. La valeur vient après l'accolade : 7.20896e+06, soit 7 208 960 octets (environ 7.2 Mo). Le dernier nombre est l'horodatage de la mesure, en millisecondes.

## Étape 3 — Prometheus

Vérification : dans Target health, les cibles prometheus et cadvisor sont UP. La requête container_memory_usage_bytes{name=~"web.*"} renvoie 3 séries, une par serveur web.

## Étape 7 — Faire bouger les courbes

Q2. Quel conteneur voit son CPU et son trafic réseau augmenter ? Pourquoi pas les deux autres ?

Seulement web1. Mesures pendant la boucle curl (moyenne sur 1 minute) :

```
web1   CPU 0.049 cœur   réseau reçu    83.0 Ko/s
web2   CPU 0.000 cœur   réseau reçu     0.0 Ko/s
web3   CPU 0.000 cœur   réseau reçu     0.0 Ko/s
```

La boucle n'interroge que le port 80, redirigé vers web1. Chaque conteneur a ses propres cgroups et sa propre interface réseau, que cAdvisor mesure séparément : web2 et web3 ne reçoivent aucune requête et restent au repos.

Q3. Après docker stop web2, que devient-il dans Grafana ? La cible cadvisor passe-t-elle à DOWN ?

Dans Grafana, les courbes de web2 s'interrompent, car cAdvisor cesse aussitôt de publier ses métriques. Les courbes calculées avec rate (CPU, réseau) disparaissent en moins d'une minute. La mémoire reste figée sur sa dernière valeur pendant environ 5 minutes : cAdvisor horodate lui-même ses mesures, et Prometheus garde alors la dernière valeur connue pendant 5 minutes avant de l'abandonner.

La cible cadvisor reste UP : la cible n'est pas web2 mais cAdvisor, qui tourne toujours et répond. Pour détecter un conteneur arrêté, il faut donc surveiller l'absence de ses métriques, pas l'état de la cible.

Observé : juste après l'arrêt, cAdvisor publiait 0 ligne pour web2 ; 60 s après, la cible cadvisor était UP, le CPU de web2 n'avait plus aucune valeur et sa mémoire avait encore sa dernière valeur, figée.

Q4. Et après docker start web2 ?

cAdvisor republie ses métriques et la courbe reprend : le CPU de web2 est revenu dans Prometheus en 16 s, le temps de collecter deux nouvelles mesures.

## Pour aller plus loin

Panneau personnalisé : tableau de bord « Mes serveurs web », avec le panneau « CPU des serveurs web » (requête sum by (name) (rate(container_cpu_usage_seconds_total{name=~"web.*"}[1m])), période Last 15 minutes).

Q5. Sans rate, quelle est la forme des courbes ? Pourquoi sont-elles inexploitables ?

container_cpu_usage_seconds_total est un compteur : il cumule les secondes de CPU depuis le démarrage du conteneur et ne fait qu'augmenter (web1 en était à 4.3 s). Les courbes sont des lignes qui montent sans cesse : leur hauteur dépend surtout de l'âge du conteneur, pas de sa charge actuelle, et elles repartent de zéro à chaque redémarrage. C'est la pente qui donne la charge : rate() la calcule, en secondes de CPU par seconde, c'est-à-dire en cœurs utilisés.

Q6. Pour ajouter web4 sur le port 83, faut-il modifier prometheus.yml ?

Non. Prometheus n'interroge que cAdvisor, et cAdvisor découvre seul tous les conteneurs Docker. web4 est apparu dans Prometheus 15 s après son lancement, sans toucher à la configuration, et la requête name=~"web.*" l'inclut. Il ne faudrait modifier prometheus.yml que pour ajouter une nouvelle cible à interroger, par exemple un autre exporter.

Q7. Créer une alerte

Non réalisée.

Q8. Tout décrire dans un docker-compose.yml

C'est ce que fait correction/docker-compose.yml : un service par commande docker run, avec extra_hosts à la place de --add-host et network_mode: bridge pour rester sur le réseau Docker par défaut. La pile démarre avec docker compose up -d et s'arrête avec docker compose down.
