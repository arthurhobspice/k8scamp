# k8scamp

Kubernetes Camp Advanced München 14.-16.9.2021

Trainer Erkan Yanar

https://zwerk.org/hedgedoc/k8s202109#

https://linsenraum.de/KubernetesCamp/

Beispieldateien aus github.com/erkules/k8sworkshop, teilweise hier reinkopiert.

Slides gibt's auch offline als Docker Image: docker pull erkules/kubernetescamp

# Die Basics

Vier zentrale Komponenten auf dem Master Node (christoph0): etcd, Controller, Scheduler, API Server.
Sind alle static pods, Definition in /etc/kubernetes/manifests.
kubelet sorgt dafür, dass die Pods laufen. Test: verschiebe Manifest nach /tmp => Pod weg, hole
Manifest zurück => Pod wieder da.
etcd persistiert seine Daten auf Disk, hält den Zustand des Clusters.

Alle Kommunikation läuft über den API Server.

Host Port: Eigenschaft eines Pods, analog zu Docker-Portmapping. Unter welchem Port ist der Pod erreichbar,
(nur) dort wo er läuft. Architektonisch will man i.d.R. keine Host Ports haben.

Node Port: Eigenschaft eines Services, wird auf jedem Node angelegt und vom
kube-proxy verwaltet, egal ob dahinter ein Pod läuft.

# Pod Priority

Wichtigere Pods haben eine höhere Nummer. Die absolute Höhe ist egal, es geht nur ums Verhältnis.

Eviction: Workloads loswerden. Gründe: zu wenig Disk, zu wenig RAM, zu viele Prozesse.

```
# kubectl get priorityclasses
# kubectl get pc
```

Sollte man in Produktion auf jeden Fall machen.
PriorityClass wird vom Scheduler verwendet. Wenn man hinreichend viele Pods der "Wichtiger-Klasse"
startet, läuft irgendwann mal keiner der "Unwichtiger-Klasse" mehr. Man kann keine Mindestanzahl
festlegen (allein mit PriorityClass).

Übung: Verzeichnis PodPriority, beide PC und beide Deployments ausrollen, mit den Replicas spielen.

# Network Policies

Erkan mag Calico und Cilium, Weave nicht.

CNI = Container Network Interface. Nicht alle CNI Plugins unterstützen Network Policies.

Immer mit einer Richtung anfangen, die meisten Topologien lassen sich damit aufbauen.

Mit der ersten Regel wechselt die Kommunikation von "alles erlaubt" auf "alles verboten, außer
es gibt eine Regel" (für die Pods, die mit PodSelector ausgewählt wurden!).

```
root@christoph0 ~/Git/k8scamp/NetworkPolicies (master)$ k apply -f basics.yaml
namespace/netz created
deployment.apps/blau created
service/blau created
deployment.apps/lila created
service/lila created
deployment.apps/rot created
service/rot created
root@christoph0 ~/Git/k8scamp/NetworkPolicies (master)$ k -n netz get svc
NAME   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
blau   ClusterIP   10.101.204.168   <none>        80/TCP    87s
lila   ClusterIP   10.105.111.226   <none>        80/TCP    87s
rot    ClusterIP   10.110.211.173   <none>        80/TCP    87s
root@christoph0 ~/Git/k8scamp/NetworkPolicies (master)$ k -n netz get pods --show-labels
NAME                    READY   STATUS    RESTARTS   AGE   LABELS
blau-86bf77df48-n2jq9   1/1     Running   0          47m   pod-template-hash=86bf77df48,pods=blau
lila-6b68f957d4-g4c9r   1/1     Running   0          47m   pod-template-hash=6b68f957d4,pods=lila
rot-5c9fb675b7-vwvh7    1/1     Running   0          47m   pod-template-hash=5c9fb675b7,pods=rot
```

Zum Testen: curl auf einem Pod ausführen

```
root@christoph0 ~/Git/k8sworkshop (master)$ k -n netz exec -ti lila-6b68f957d4-g4c9r -- sh
/ #
```

Jetzt Network Policy ausrollen:
```
root@christoph0 ~/Git/k8scamp/NetworkPolicies (master)$ k apply -f pod2pod.yaml
networkpolicy.networking.k8s.io/tored created
```

Von blauen Pods zu roten Pods wird Port 80 erlaubt. Durch PodSelector greift die Regel nur
für die roten Pods. Ergo: rote Pods sind nur noch von blauen Pods erreichbar. Blaue und lila Pods
sind aber nach wie vor von überall erreichbar.

Bei Hochskalieren greifen die Regeln automatisch für neue Pods. Performance problematisch bei
mehreren 1.000 Pods. Neue Technologien: eBPF (noch in den Kinderschuhen).

Network Policies sind "namespaced resources".

Wichtig: für Kommunikation von außen muss Ingress erlaubt werden. Für Kommunikation raus muss
DNS Resolution erlaubt werden.

```
root@christoph0 ~/Git/k8scamp/NetworkPolicies (master)$ cat allepodsimNamespaceErreichensichII.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-own-name-space
  namespace: netz
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector: {}
```

ist eine "Alle-Pods-im-Namespace-erreichen-sich"-Regel. Lässt man die letzten drei Zeilen weg, hat man
eine "Deny all"-Regel.

Mit namespaceSelector kann man Pods aus dem anderen Namespace als Source auswählen. Beispiel: traefik-Ingress
aus Namespace kube-system Zugriff erlauben. Siehe IngressDarfAufNamespacezugreifen.yaml.

Typischerweise wird Prometheus der Zugriff für den Abruf von Metriken erlaubt.

Network Policies sind Kubernetes-Standard. Man braucht ein CNI-Plugin das die Regeln implementiert: Calico,
Cilium, etc. Ohne das kann man Policies definieren wie man will, es gibt nichts, was sie umsetzt.

```
root@christoph0 ~/Git/k8scamp/NetworkPolicies (master)$ k explain networkpolicies --recursive | less
```

allepodsimNamespaceOutgoing.yaml: DNS-Traffic ausgehend überall hin erlaubt, ansonsten nur Traffic nach draußen.
allepodsimNamespaceOutgoing2.yaml: DNS-Traffic ist nur nach draußen erlaubt.
(Jeder Arrayeintrag ist ein Satz von Regeln.)

# Pod Security Policy

Capabilities - Vergleich mit Docker

```
root@christoph0 ~/Git/k8scamp (master)$ grep CapEff /proc/self/status
CapEff: 0000003fffffffff
root@christoph0 ~/Git/k8scamp (master)$ capsh --decode=0000003fffffffff  | tr , "\n"
0x0000003fffffffff=cap_chown
cap_dac_override
cap_dac_read_search
cap_fowner
cap_fsetid
(...)
root@christoph0 ~/Git/k8scamp (master)$ docker container run -ti alpine
/ # grep CapEff /proc/self/status
CapEff: 00000000a80425fb
/ # ls -l /proc/self/ns
total 0
lrwxrwxrwx    1 root     root             0 Sep 14 12:36 cgroup -> cgroup:[4026531835]
lrwxrwxrwx    1 root     root             0 Sep 14 12:36 ipc -> ipc:[4026532538]
lrwxrwxrwx    1 root     root             0 Sep 14 12:36 mnt -> mnt:[4026532536]
lrwxrwxrwx    1 root     root             0 Sep 14 12:36 net -> net:[4026532541]
lrwxrwxrwx    1 root     root             0 Sep 14 12:36 pid -> pid:[4026532539]
lrwxrwxrwx    1 root     root             0 Sep 14 12:36 pid_for_children -> pid:[4026532539]
lrwxrwxrwx    1 root     root             0 Sep 14 12:36 user -> user:[4026531837]
lrwxrwxrwx    1 root     root             0 Sep 14 12:36 uts -> uts:[4026532537]
/ # exit
root@christoph0 ~/Git/k8scamp (master)$ ls -l /proc/self/ns
total 0
lrwxrwxrwx 1 root root 0 Sep 14 14:36 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 root root 0 Sep 14 14:36 ipc -> 'ipc:[4026531839]'
lrwxrwxrwx 1 root root 0 Sep 14 14:36 mnt -> 'mnt:[4026531840]'
lrwxrwxrwx 1 root root 0 Sep 14 14:36 net -> 'net:[4026531992]'
lrwxrwxrwx 1 root root 0 Sep 14 14:36 pid -> 'pid:[4026531836]'
lrwxrwxrwx 1 root root 0 Sep 14 14:36 pid_for_children -> 'pid:[4026531836]'
lrwxrwxrwx 1 root root 0 Sep 14 14:36 user -> 'user:[4026531837]'
lrwxrwxrwx 1 root root 0 Sep 14 14:36 uts -> 'uts:[4026531838]'
```

Docker-Container haben weniger Privilegien als der Host, außer sie laufen "privileged". Andere Namespaces.

Wir möchten verhindern, dass ein kompromittierter Container Unfug auf dem Host-System anstellt.

Pod Security Policies sind "deprecated".
Verwende stattdessen OPA Gatekeeper, Kyverno (einfacher als OPA), ...

# Kyverno

Wir möchten eine Policy bauen, die das Anlegen von NodePort-Services verhindert.

```
root@christoph0 ~/Git/k8scamp/Kyverno (master)$ kubectl apply -f check_nodeport.yaml
policy.kyverno.io/check-node-port created
root@christoph0 ~/Git/k8scamp/Kyverno (master)$ kubectl get policy
NAME              BACKGROUND   ACTION
check-node-port   true         enforce
```

Deployment und Service nginx erzeugen:

```
root@christoph0 ~/Git/k8scamp/nginx (master)$ kubectl apply -f nginx.yaml
deployment.apps/nginx-deployment created
root@christoph0 ~/Git/k8scamp/nginx (master)$ kubectl get pods -n nginx
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-66b6c48dd5-fskpx   1/1     Running   0          14s
nginx-deployment-66b6c48dd5-krz6s   1/1     Running   0          14s
```

Policies sind "namespaced resources" => Namespace mit angeben:

```
root@christoph0 ~/Git/k8scamp/Kyverno (master)$ kubectl apply -f check_nodeport.yaml -n nginx
policy.kyverno.io/check-node-port created
root@christoph0 ~/Git/k8scamp/Kyverno (master)$ cd ..
root@christoph0 ~/Git/k8scamp (master)$ cd nginx/
root@christoph0 ~/Git/k8scamp/nginx (master)$ kubectl apply -f service2.yaml
Error from server: error when creating "service2.yaml": admission webhook "validate.kyverno.svc" denied the request:

resource Service/nginx/nginx-nodeport-service was blocked due to the following policies

check-node-port:
  check-node-port: 'validation error: NodePort type is not allowed. Rule check-node-port
    failed at path /spec/type/'
```

Man kann die Policy auch als ClusterPolicy anlegen, dann hat man aber Seiteneffekte (einige Sachen gehen nicht mehr).
Wenn ClusterPolicy, dann besser Namespaces einschränken (Verwendung von Wildcards). Vorsicht bei ClusterPolicies
auf kube-system!

Validierung: Pod muss ein Label haben.

```
root@christoph0 ~/Git/k8scamp/Kyverno (master)$ kubectl create namespace test
namespace/test created
root@christoph0 ~/Git/k8scamp/Kyverno (master)$ kubectl apply -f policy_validate_pod_label.yaml -n test
policy.kyverno.io/validate-pod-label-app created
root@christoph0 ~/Git/k8scamp/Kyverno (master)$ kubectl get policy -n test
NAME                     BACKGROUND   ACTION
validate-pod-label-app   true         enforce
root@christoph0 ~/Git/k8scamp/Kyverno (master)$ cp ~/Git/k8sworkshop/Pods/pod.yaml .
root@christoph0 ~/Git/k8scamp/Kyverno (master)$ kubectl apply -f pod.yaml
Error from server: error when creating "pod.yaml": admission webhook "validate.kyverno.svc" denied the request:

resource Pod/test/www was blocked due to the following policies

validate-pod-label-app:
  validate-pod-label-app: 'validation error: Pod muss ein app Label haben!. Rule validate-pod-label-app
    failed at path /metadata/labels/'
root@christoph0 ~/Git/k8scamp/Kyverno (master)$ vi pod.yaml
(pod.yaml fixen: Label hinzufügen)
root@christoph0 ~/Git/k8scamp/Kyverno (master)$ k apply -f pod.yaml
pod/www created
root@christoph0 ~/Git/k8scamp/Kyverno (master)$ k get pods -n test
NAME   READY   STATUS    RESTARTS   AGE
www    1/1     Running   0          7s
root@christoph0 ~/Git/k8scamp/Kyverno (master)$ k get pods -n test --show-labels
NAME   READY   STATUS    RESTARTS   AGE   LABELS
www    1/1     Running   0          14s   app=nginxhostname
```

podpriorityComplex.yaml: am Anfang wird eine ConfigMap definiert (key-value), auf die nachfolgend zugegriffen wird - Pod Priorities
für namespaces foo und bar. Deployments, DaemonSets haben Templates, wie Pods definiert werden. Schon beim Erstellen von
Deployments etc. werden Fehler ausgegeben.

Slide 19/23: einfaches Beispiel, wie man bei Angabe von "latest" immer das latest-Image pullen lassen kann (wird z.B. von docker[-compose]
nicht gemacht).

Weitere Beispiele zeigen: man kann mit Kyverno eine Menge machen, was man auch mit ArgoCD oder Flux machen könnte.
Man sollte Scopes festlegen, wer macht was, wenn man alles im Einsatz hat, sonst schwer zu debuggen!

# Pod Disruption Budgets

PDB legen fest, wie viele Pods laufen müssen (max. nicht verfügbar sein dürfen). Relevant z.B. bei Node Draining, Quorum-basierten
Clusterdiensten.

```
root@christoph0 ~/Git/k8scamp/PDB (master)$ k apply -f simple-deployment-pdb.yaml -n pdb
poddisruptionbudget.policy/simplests-pdb created
deployment.apps/wwwanti created
root@christoph0 ~/Git/k8scamp/PDB (master)$ k drain christoph3 --ignore-daemonsets --delete-emptydir-data --force
node/christoph3 cordoned
WARNING: ignoring DaemonSet-managed Pods: calico-system/calico-node-mkczk, kube-system/kube-proxy-hq5kl
evicting pod pdb/wwwanti-5f45bbd9c8-g57z4
evicting pod calico-apiserver/calico-apiserver-554fbf9554-255nm
evicting pod kyverno/kyverno-7b7f89c6f7-cqcfs
error when evicting pods/"wwwanti-5f45bbd9c8-g57z4" -n "pdb" (will retry after 5s): Cannot evict pod as it would violate the pod's disruption budget.
evicting pod pdb/wwwanti-5f45bbd9c8-g57z4
error when evicting pods/"wwwanti-5f45bbd9c8-g57z4" -n "pdb" (will retry after 5s): Cannot evict pod as it would violate the pod's disruption budget.
evicting pod pdb/wwwanti-5f45bbd9c8-g57z4
error when evicting pods/"wwwanti-5f45bbd9c8-g57z4" -n "pdb" (will retry after 5s): Cannot evict pod as it would violate the pod's disruption budget.
evicting pod pdb/wwwanti-5f45bbd9c8-g57z4
error when evicting pods/"wwwanti-5f45bbd9c8-g57z4" -n "pdb" (will retry after 5s): Cannot evict pod as it would violate the pod's disruption budget.
pod/calico-apiserver-554fbf9554-255nm evicted
^C
root@christoph0 ~/Git/k8scamp/PDB (master)$
```

PDB verhindert den Drain: drei Pods müssen immer laufen. Fix: minAvailable auf 2 setzen, nochmal kubectl drain...

# Helm

Komplexe Deployments mit Templates und Values

helm upgrade --install ist idempotent, helm install nicht (Fehlermeldung, wenn's schon da ist).

Verschiedene Stages, zum Beispiel: values.yaml kommt mit dem Chart, mit Defaultwerten. Mit helm show values
exportieren, anpassen, je Stage mit -f ... wieder reinschießen.

k8scamp/Helm/simple -> einfachstes Beispiel für eine Helm-Chart-Struktur

Basisstruktur wird mit create schon vorgegeben:

```
root@christoph0 ~/Git/k8scamp/Helm (master)$ helm create created
Creating created
root@christoph0 ~/Git/k8scamp/Helm (master)$ helm package ./created
Successfully packaged chart and saved it to: /root/Git/k8scamp/Helm/created-0.1.0.tgz
root@christoph0 ~/Git/k8scamp/Helm (master)$ vi created/Chart.yaml
(Version hochzählen...)
root@christoph0 ~/Git/k8scamp/Helm (master)$ helm package ./created
Successfully packaged chart and saved it to: /root/Git/k8scamp/Helm/created-0.1.1.tgz
```

Prüfen vor dem Ausrollen:
```
root@christoph0 ~/Git/k8scamp/Helm (master)$ helm lint simple/
==> Linting simple/
[ERROR] Chart.yaml: appVersion should be of type string but it's of type float64
[INFO] Chart.yaml: icon is recommended

Error: 1 chart(s) linted, 1 chart(s) failed
root@christoph0 ~/Git/k8scamp/Helm (master)$ helm template RELEASENAME ./templateexample/
```

(helm template: lokales Rendern von Templates)

Experimental: Harbor Charts als Docker Images

# Eviction

Linux hat einen OOM-Killer, der Prozesse wegschießt, wenn Speicher knapp wird. Der folgt einer eigenen Logik.
Wir würden das aber gerne über K8s kontrollieren (kubelet), müssen damit aber dem OOM-Killer zuvorkommen
-> Resource Quotas. Sind RQ im Namespace gesetzt, müssen die Pods Anforderungen mitbringen.

Siehe ResourceQuota, LimitRange

Eviction findet aufgrund von RAM oder Disk statt, nicht aufgrund von CPU (wird nur sehr langsam).

# Harbor

Docker Registry mit Security Scanner (Trivy, braucht i.Ggs. zu z.B. Qualys keine DB), Notary (Images zertifizieren).

Ohne Zertifikate ist Harbor nichts wert.

```
root@christoph0 ~/Git/k8scamp/Harbor (master)$ k apply -f traefik-ds-http.yaml
clusterrole.rbac.authorization.k8s.io/traefik-ingress-controller-clusterrole created
clusterrolebinding.rbac.authorization.k8s.io/traefik-ingress-controller-binding created
root@christoph0 ~/Git/k8scamp/Harbor (master)$ cp -r ../../k8sworkshop/CertManager/ .
root@christoph0 ~/Git/k8scamp/Harbor (master)$ cd CertManager/
root@christoph0 ~/Git/k8scamp/Harbor/CertManager (master)$ helm repo add jetstack https://charts.jetstack.io
"jetstack" has been added to your repositories
root@christoph0 ~/Git/k8scamp/Harbor/CertManager (master)$ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "jetstack" chart repository
...Successfully got an update from the "stable" chart repository
Update Complete. ⎈Happy Helming!⎈
root@christoph0 ~/Git/k8scamp/Harbor/CertManager (master)$ helm install \
>    cert-manager jetstack/cert-manager \
>      --namespace cert-manager \
>        --create-namespace \
>          --version v1.5.3 \
>             --set installCRDs=true

NAME: cert-manager
LAST DEPLOYED: Wed Sep 15 14:21:58 2021
NAMESPACE: cert-manager
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
cert-manager v1.5.3 has been deployed successfully!

In order to begin issuing certificates, you will need to set up a ClusterIssuer
or Issuer resource (for example, by creating a 'letsencrypt-staging' issuer).

More information on the different types of issuers and how to configure them
can be found in our documentation:

https://cert-manager.io/docs/configuration/

For information on how to configure cert-manager to automatically provision
Certificates for Ingress resources, take a look at the `ingress-shim`
documentation:

https://cert-manager.io/docs/usage/ingress/
root@christoph0 ~/Git/k8scamp/Harbor/CertManager (master)$
root@christoph0 ~/Git/k8scamp/Harbor/CertManager (master)$ helm list -n cert-manager
NAME            NAMESPACE       REVISION        UPDATED                                         STATUS          CHART                   APP VERSION
cert-manager    cert-manager    1               2021-09-15 14:21:58.359732797 +0200 CEST        deployed        cert-manager-v1.5.3     v1.5.3
root@christoph0 ~/Git/k8scamp/Harbor/CertManager (master)$ k apply -f clusterissuerLEStaging.yaml
clusterissuer.cert-manager.io/le-clusterissuer-staging created
root@christoph0 ~/Git/k8scamp/Harbor/CertManager (master)$ k apply -f clusterissuerLEprod.yaml
clusterissuer.cert-manager.io/le-clusterissuer-prod created
root@christoph0 ~/Git/k8scamp/Harbor/CertManager (master)$ mv harbor.erkan.zwerk.org-staging.yaml harbor.christoph.zwerk.org-staging.yaml
root@christoph0 ~/Git/k8scamp/Harbor/CertManager (master)$ vi harbor.christoph.zwerk.org-staging.yaml
(DNS anpassen...)
root@christoph0 ~/Git/k8scamp/Harbor/CertManager (master)$ mv notary.erkan.zwerk.org-staging.yaml notary.christoph.zwerk.org-staging.yaml
root@christoph0 ~/Git/k8scamp/Harbor/CertManager (master)$ vi notary.christoph.zwerk.org-staging.yaml
(DNS anpassen...)
root@christoph0 ~/Git/k8scamp/Harbor/CertManager (master)$ k apply -f harbor.christoph.zwerk.org-staging.yaml
certificate.cert-manager.io/harbor-tls-prod created
root@christoph0 ~/Git/k8scamp/Harbor/CertManager (master)$ k apply -f notary.christoph.zwerk.org-staging.yaml
certificate.cert-manager.io/notary-tls-staging created
(Das gleiche nochmal für prod...)
root@christoph0 ~/Git/k8scamp/Harbor/CertManager (master)$ k get cert
NAME                 READY   SECRET                           AGE
harbor-tls-prod      True    harborchristophzwerkorgprod      2m32s
harbor-tls-staging   True    harborchristophzwerkorgstaging   2m51s
notary-tls-prod      True    notarychristophzwerkorgprod      2m24s
notary-tls-staging   True    notarychristophzwerkorgstaging   5s
root@christoph0 ~/Git/k8scamp/Harbor/CertManager (master)$ helm repo add harbor https://helm.goharbor.io
"harbor" has been added to your repositories
root@christoph0 ~/Git/k8scamp/Harbor/CertManager (master)$ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "harbor" chart repository
...Successfully got an update from the "jetstack" chart repository
...Successfully got an update from the "stable" chart repository
Update Complete. ⎈Happy Helming!⎈
root@christoph0 ~/Git/k8scamp/Harbor/CertManager (master)$ cd ..
root@christoph0 ~/Git/k8scamp/Harbor (master)$ cp ~/Git/k8sworkshop/Harbor/values.yaml .
root@christoph0 ~/Git/k8scamp/Harbor (master)$ vi values.yaml
(erkan durch christoph ersetzen)
root@christoph0 ~/Git/k8scamp/Harbor (master)$ helm upgrade --install -f values.yaml harbor harbor/harbor
Release "harbor" does not exist. Installing it now.
NAME: harbor
LAST DEPLOYED: Wed Sep 15 14:45:09 2021
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Please wait for several minutes for Harbor deployment to complete.
Then you should be able to visit the Harbor portal at https://harbor.christoph.zwerk.org
For more details, please visit https://github.com/goharbor/harbor
```

Zertifikate und Harbor sind jetzt im default namespace, könnte man verbessern...

Traefik hatten wir installiert, weil wir ohne Zertifikat ein Zertifikat nur über HTTP holen können.
Jetzt tauschen wir aus:
```
root@christoph0 ~/Git/k8scamp/Harbor (master)$ k delete -f traefik-ds-http.yaml
clusterrole.rbac.authorization.k8s.io "traefik-ingress-controller-clusterrole" deleted
clusterrolebinding.rbac.authorization.k8s.io "traefik-ingress-controller-binding" deleted
serviceaccount "traefik-ingress-controller" deleted
daemonset.apps "traefik-ingress-controller" deleted
root@christoph0 ~/Git/k8scamp/Harbor (master)$ k apply -f traefik-ds.yaml
clusterrole.rbac.authorization.k8s.io/traefik-ingress-controller-clusterrole created
clusterrolebinding.rbac.authorization.k8s.io/traefik-ingress-controller-binding created
serviceaccount/traefik-ingress-controller created
daemonset.apps/traefik-ingress-controller created
```

Jetzt ist Harbor erreichbar unter https://harbor.christoph.zwerk.org/.

```
root@christoph0 ~/Git/k8scamp/Harbor (master)$ helm repo add --username admin --password dontshow mycharts https://harbor.christoph.zwerk.org/chartrepo/arthurhobspice
"mycharts" has been added to your repositories
root@christoph0 ~/Git/k8scamp/Harbor (master)$ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "mycharts" chart repository
...Successfully got an update from the "jetstack" chart repository
...Successfully got an update from the "harbor" chart repository
...Successfully got an update from the "stable" chart repository
Update Complete. ⎈Happy Helming!⎈
```

# ArgoCD

"GitOps im Cluster"

Installieren gemäß Slides. Auf PC Port Forwarding aktivieren (wir bauen keinen Ingress):

```
Christoph.Bauer@E0423066 MINGW64 ~/.kube
$ kubectl cluster-info
Kubernetes master is running at https://95.216.142.146:6443
CoreDNS is running at https://95.216.142.146:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.

Christoph.Bauer@E0423066 MINGW64 ~/.kube
$ kubectl -n argocd port-forward  svc/argocd-server 8080:80
Forwarding from 127.0.0.1:8080 -> 8080
Forwarding from [::1]:8080 -> 8080
```

Erstes Beispiel: gitlab.yaml, hole von GitLab und rolle auf eigenem Cluster (https://kubernetes.default.svc) aus.

```
root@christoph0 ~/Git/k8scamp/ArgoCD (master)$ k apply -f gitlab.yaml
application.argoproj.io/firstgit created
```

Update der deployment.yaml auf GitLab => nach spätestens drei Minuten rollt ArgoCD die Änderungen automatisch aus.
ArgoCD macht kubectl apply auf das GitLab-Verzeichnis (alle yaml, die darin gefunden werden).

Deployment löschen => ArgoCD stellt es sofort wieder her.

Helm-Beispiel: tomcat-helm.yml. Dieses Beispiel "pinnt" die Revision. Man könnte auch auf Target Rev. "latest" gehen.

Vorteil von ArgoCD: Management von Deployments in vielen Clustern ist wesentlich einfacher.
Nachteil: obwohl wir ein Helm Chart verwenden, sind einige Features von Helm nicht anwendbar (Debuggen, Hooks, ...), weil das Chart
nicht "in unserem Helm Scope ist". Daher geht Erkan jetzt weg von ArgoCD zu Flux.

Empfehlung: keine ArgoCD-Hooks bauen, weil man sich zu viele Abhängigkeiten von ArgoCD schafft.
