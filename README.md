# CGroupsV2_in_AKS_1.25_JDKBelow11_OOM
Java apps running < JDK ver 11 moving to K8s 1.25 showing memory issue - increased resource consumption and at times OOM issue. 

One of our customer has moved to 1.25 with older SDK noticed their cluster node size has almost doubled to meet the app resource demand otherwise OOM issue(17 nodes to 
30+ nodes)

Background:
1-	In 1.25 cgroupsV2 became default also in 1.25 we introduced ubuntu 22.04 which has cgroupsv2 as default too. In the k8s announcement an important caveat is this “If you deploy Java applications with the JDK, prefer to use JDK 11.0.16 and later or JDK 15 and later, which fully support cgroup v2.”
2-	Customers who are using any JDK below 11.0.16 i.e., 11.0.10 and who upgraded to 1.25 started noticing that their pods are not respecting resource and limits leading to all sorts of performance issues and OOMs. 
https://kubernetes.io/blog/2022/08/31/cgroupv2-ga-1-25/

Recommendations:

1-	Best path is to update our JDK version to a one that has the fix i.e. 11.0.16/17 or above of course, the docs up stream points out that the fix got backported to 8 too however it doesn’t seem that 372 has made it to any mainstream distribution of the JDK 
https://bugs.openjdk.org/browse/JDK-8230305
2-	If we can’t update our JDK in-time (given 1.24 EOL), then the attached yaml would be the ideal solution, this will allow us to upgrade to 1.25 and keep cgroupsv1 as the effective cgroups mode.
3-	Worth mentioning that its preferred to keep this scoped to a nodepool  (modify the node Affinity rules in the DS file) where you have the workloads running the outdated JDK version, so other workloads can still benefit from cgroupsv2

```code
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: revert-cgroups
  namespace: kube-system
spec:
  selector:
    matchLabels:
      name: revert-cgroups
  template:
    metadata:
      labels:
        name: revert-cgroups
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: cgroup-version
                    operator: NotIn
                    values:
                      - v1
      tolerations:
        - operator: Exists
          effect: NoSchedule
      containers:
        - name: revert-cgroups
          image: mcr.microsoft.com/cbl-mariner/base/core:1.0
          command:
            - nsenter
            - --target
            - "1"
            - --mount
            - --uts
            - --ipc
            - --net
            - --pid
            - --
            - bash
            - -exc
            - |
              CGROUP_VERSION=`stat -fc %T /sys/fs/cgroup/`
              if [ "$CGROUP_VERSION" == "cgroup2fs" ]; then
                echo "Using v2, reverting..."
                sed -i 's/GRUB_CMDLINE_LINUX=""/GRUB_CMDLINE_LINUX="systemd.unified_cgroup_hierarchy=0"/' /etc/default/grub
                update-grub
                kubectl --kubeconfig=/var/lib/kubelet/kubeconfig label node ${HOSTNAME,,} cgroup-version=v1
                reboot
              else
                kubectl --kubeconfig=/var/lib/kubelet/kubeconfig label node ${HOSTNAME,,} cgroup-version=v1
              fi

              sleep infinity
          resources:
            limits:
              memory: 200Mi
            requests:
              cpu: 100m
              memory: 16Mi
          securityContext:
            privileged: true
      hostNetwork: true
      hostPID: true
      hostIPC: true
      terminationGracePeriodSeconds: 0
```
