# k8scamp

Kubernetes Camp Advanced München 14.-16.9.2021

https://zwerk.org/hedgedoc/k8s202109#

# Pod priority

Wichtigere Pods haben eine höhere Nummer. Die absolute Höhe ist egal, es geht nur ums Verhältnis.

Eviction: Workloads loswerden. Gründe: zu wenig Disk, zu wenig RAM, zu viele Prozesse.

```
# kubectl get priorityclasses
# kubectl get pc
```