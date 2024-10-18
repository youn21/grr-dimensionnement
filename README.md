# TP Dimensionnement

[TOC]

## Quotas

Vérifiez les quotas de ressources (cpu/mémoire/pod) associés à votre namespace
```bash
kubectl get resourcequota
```

Ne garder qu'un seul projet pour rester dans la limite globale des ressources de la plateforme. Le but est le dimensionnement de votre déploiement grr.

Réduire la quantité de mémoire et de cpu utilisés par les autres workload du namespace.

## Générateur de trafic grafana/k6

L'opérateur grafana/k6 est installé sur la plateforme. Il propose de lancer dans le namespace une génération de charges. Le test, écrit en javascript, peut s'adapater à tout type d'application. Il décrit un scénario de charge avec des montées et des descentes du nombre de Virtual User. Voir documentation https://grafana.com/docs/k6/latest/.

Vous commencez par interroger la page login de votre GRR avec une montée en charge jusqu'à 1000 en 1 minute puis 13 minutes à pleine charge et une descente la dernière minute. Le test.js est passé sous forme de data d'une ConfigMap :

```bash
oc_user=$(kubectl auth whoami -o jsonpath='{.status.userInfo.username}')
cat <<EOF > k6-configmap.yml
apiVersion: v1
kind: ConfigMap
metadata:
  creationTimestamp: null
  name: k6-${oc_user}-test
data:
  test.js: |
    import http from 'k6/http';
    import { sleep } from 'k6';

    export const options = {
      // Key configurations for Stress in this section
      executor: 'ramping-arrival-rate', //Assure load increase if the system slows
      stages: [
        { duration: '1m', target: 1000 }, // traffic ramp-up from 1 to a higher 1000 users over 1 minutes.
        { duration: '13m', target: 1000 }, // stay at higher 1000 users for 13 minutes
        { duration: '1m', target: 0 }, // ramp-down to 0 users
      ],
    };

    export default function () {
      const url1 = 'https://grr-${oc_user}.apps.anf.math.cnrs.fr/login.php';

      const payload = JSON.stringify({
        email: 'ADMINISTRATEUR',
        password: 'admin',
      });

      const params = {
        headers: {
          'Content-Type': 'application/json',
        },
      };

      http.post(url1, payload, params);
      // sleep(1);
    };
EOF
kubectl apply -f k6-configmap.yml
```
L'opérateur définit le type de CustomRessource TestRun pour définir une campagne de test.

```bash
oc_user=$(kubectl auth whoami -o jsonpath='{.status.userInfo.username}')
cat <<EOF > k6-testrun.yml
apiVersion: k6.io/v1alpha1
kind: TestRun
metadata:
  name: k6-${oc_user}
spec:
# suppression du TestRun apres la fin du test
#  cleanup: 'post'
  parallelism: 1
  script:
    configMap:
      name: k6-${oc_user}-test
      file: test.js
  arguments: --out prometheus
  separate: false
  runner:
# utilisation d'une image d'extension k6 qui expose un Prometheus exporter sur le port TCP/5656
    image: ghcr.io/szkiba/xk6-prometheus:latest
    metadata:
      labels:
        k6: k6-${oc_user}
    securityContext:
      runAsNonRoot: true
    resources:
      limits:
        cpu: 200m
        memory: 512Mi
      requests:
        cpu: 100m
        memory: 256Mi
# empeche le lancement du test pas de starter
    readinessProbe:
      timeoutSeconds: 10
    livenessProbe:
      timeoutSeconds: 10
  starter:
    metadata:
      labels:
        k6: k6-${oc_user}
    securityContext:
      runAsNonRoot: true
EOF
```

Le test se lance par `kubectl create -f k6-testrun.yml`. Il dure ici 15 minutes pour laisser la plateforme réagirt à la charge. Le réduire pour les premiers tests.
Suivre le lancement des pod initializer, starter et le générateur de trafic lui-même. Suivre le résultat de ce dernier pod dans son log `kubectl logs k6-oc_user-...`.
Sans l'option cleanup, il faut détruire le test à la main `kubectl delete -f k6-testrun.yml`.

L'extension xk6-prometheus permet d'intégrer des métric dans le prometheus de la plateforme. Pour cela définissez un Service qui sera lié via le label au démarrage du TestRun.
```bash
oc_user=$(kubectl auth whoami -o jsonpath='{.status.userInfo.username}')
cat <<EOF > k6-service.yml
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    k6: k6-${oc_user}
  name: k6-${oc_user}
spec:
  clusterIP: None
  ports:
  - name: k6-${oc_user}
    port: 5656
    protocol: TCP
    targetPort: 5656
  selector:
    k6: k6-${oc_user}
  type: ClusterIP
status:
  loadBalancer: {}
EOF
kubectl apply -f k6-service.yml
```
Un ServiceMonitor va faire la collecte du service exporter en ajoutant un préfixe "k6_".
```bash
oc_user=$(kubectl auth whoami -o jsonpath='{.status.userInfo.username}')
cat <<EOF > k6-servicemonitor.yml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  creationTimestamp: null
  labels:
    k6: k6-${oc_user}
  name: k6-${oc_user}
spec:
  endpoints:
  - interval:  10s
    path:      /metrics
    port:      k6-${oc_user}
    scheme:    http
    metricRelabelings:
    - action: replace
      regex: (.*)
      replacement: 'k6_${1}'
      sourceLabels:
      - __name__
      targetLabel: __name__
  selector:
    matchLabels:
      k6: k6-${oc_user}
EOF
kubectl apply -f k6-servicemonitor.yml
```
A partir du TestRun suivant, vous aurez les metrics dans Prometheus intégré. Observe/Metrics : par exemple k6_http_reqs

Plus de possibilités de requêtes avec votre grafana.

## Vertical Pod Autoscaler

Activation du VPA en mode Off sur votre deployment grr.

```bash
oc_user=$(kubectl auth whoami -o jsonpath='{.status.userInfo.username}')
cat <<EOF > grr-vpa.yml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: grr-${oc_user}
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind:       Deployment 
    name:       grr 
  updatePolicy:
    updateMode: "Off" 
EOF
kubectl apply -f grr-vpa.yml
```
Lancer une campagne de test assez longue pour voir la recommandation upperBound changer. `oc get vpa grr-oc_user -o yaml`

Vous pouvez surveiller le temps de réponse en fonction du nombre de requêtes et fixer un requests pour le cpu. Prenons cpu=100m et memory=256Mi pour la suite.

## Horizontal Pod Autoscaler

On va laisser le HPA faire varier le ReplicaSet entre 1 et 4 pod. Attention à avoir suffisamment de ressources dans le namespace pour l'ensemble des pod, en particulier sur la mémoire. Sinon, certains pod comme le TestRun vont être arrêtés prématurément.
Prenons comme objectif de charger à 90% les pod du deployment grr.

```bash
cat <<EOF > grr-hpa.yml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: grr-hpa-90cpu
spec:
  minReplicas: 1
  maxReplicas: 4
  metrics:
  - resource:
      name: cpu
      target:
        averageUtilization: 90
        type: Utilization
    type: Resource
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: grr
EOF
kubectl apply -f grr-hpa.yml
kubectl create -f k6-testrun.yml
kubectl get hpa -w
```
Surveillez en parallèle la création de nouveaux replicas `kubectl get pod -w`. La plateforme est configurée par défaut pour ne pas réagir à des pointes ponctuelles de charge mais d'attendre des tendances sur 5min.
