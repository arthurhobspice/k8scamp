# k8scamp

Kubernetes Camp Advanced München 14.-16.9.2021
Trainer Erkan Yanar

https://zwerk.org/hedgedoc/k8s202109#
https://linsenraum.de/KubernetesCamp/

Beispieldateien aus github.com/erkules/k8sworkshop, teilweise hier reinkopiert.

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
