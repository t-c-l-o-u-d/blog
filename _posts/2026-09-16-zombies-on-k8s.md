---
layout: post
title: "Zombies in a Kubernetes Cluster"
date: 2025-09-16
tags: kubernetes k8s openshift ocp
---

A recent finding arose when I was reviewing my Kubernetes cluster. I noticed several odd processes lingering on one of my nodes. As I began my investigation, I learned that they were zombie processes! 🧟🧟🧟 Read on to see how I learned about, detected, and emulated them.

---
## What is a zombie process?
Well, this is an obvious question I immediately had, and I'm sure some readers will too. Before this incident, I had never heard of a zombie process. Like a good engineer: to the man pages!

```bash
$ man -wK zombies
/usr/share/man/man1/perlfunc.1perl.gz
/usr/share/man/man1/perltoc.1perl.gz
/usr/share/man/man1/ps.1.gz
/usr/share/man/man1/perlipc.1perl.gz
/usr/share/man/man1/perlfaq8.1perl.gz
/usr/share/man/man1/perlfaq.1perl.gz
/usr/share/man/man8/fsck.minix.8.gz
/usr/share/man/man2/wait.2.gz
/usr/share/man/man2/sigaction.2.gz
/usr/share/man/man2/clone.2.gz
```

A lot of output, but it helps to have a little context. Since we're dealing with processes, the `ps` man page is a good start.

> "Z 	defunct (“zombie”) process, terminated but not reaped by its parent"

> "Processes marked <defunct> are dead processes (so-called "zombies") that remain because their parent has not destroyed them properly. These processes will be destroyed by init(8) if the parent process exits." - https://man.archlinux.org/man/ps.1

I also took to [Wikipedia](https://en.wikipedia.org/wiki/Zombie_process) to learn a little more.
> "Processes that stay defunct for a long time are usually an error and can cause a resource leak." - https://en.wikipedia.org/wiki/Zombie_process

Okay, so none of that sounds terrible, but let's get started with detecting these and remediating the underlying issue.


## How can we detect them?
[_] - Detection

Detecting them is quite straightforward. Logging into any `worker` node and running `ps` will show them as seen below:

```bash
$ ps -o pid,ppid,state,cmd -e | awk 'NR==1; $3 ~ /^[Zz]/'
    PID    PPID S CMD
 555382  555380 Z [zombie-1] <defunct>
 555390  555388 Z [zombie-2] <defunct>
 555398  555396 Z [zombie-3] <defunct>
 555406  555404 Z [zombie-4] <defunct>
 555414  555412 Z [zombie-5] <defunct>
 555422  555420 Z [zombie-6] <defunct>
 555430  555428 Z [zombie-7] <defunct>
 555438  555436 Z [zombie-8] <defunct>
 555446  555444 Z [zombie-9] <defunct>
 555454  555452 Z [zombie-10] <defunct>
```

Missing `ps`? This Bash loop prints PIDs of zombie processes by scanning /proc:

```bash
for i in $(ls /proc | awk '/^[0-9]/'); do
    cat /proc/$i/status 2>/dev/null | awk -v pid="${i}" '/State/ && $2 ~ "[Zz]" {print pid " "$2}'
done
```

As you can see, there are quite a few lingering on this node. As the `PIDs` are exclusive per node, each node will report something entirely different.


## How can we emulate one?
[X] - Detection  
[_] - Emulation

Great, detection works. Now how can we replicate this? Using a simple C program that invokes `fork()` makes this trivial. An example of a `Namespace`, `ConfigMap`, and `Pod` are all below that perform this in a single `kubectl apply -f zombie.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: example

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: zombie-config
  namespace: example
data:
  zombie.c: |
    #include <stdlib.h>
    #include <sys/types.h>
    #include <unistd.h>

    int main() {
        pid_t child_pid;
        child_pid = fork();
        if (child_pid > 0) {
            sleep(9001);
        } else {
            exit(0);
        }
        return 0;
    }

---
apiVersion: v1
kind: Pod
metadata:
  generateName: pod-with-a-zombie-
  namespace: example
spec:
  containers:
  - name: zombie-container
    image: docker.io/archlinux:base-devel
    command: ["/usr/bin/bash", "-c", "--"]
    args:
      - |
        for i in {1..10}; do
            cc /config/zombie.c -o /tmp/zombie-$i
            /tmp/zombie-$i &
            ps -o pid,ppid,state,cmd -C zombie-$i
            echo ""
        done &&\
        while true; do sleep 30; done
    securityContext:
        runAsUser: 1234
    volumeMounts:
      - name: config-volume
        mountPath: /config
  volumes:
    - name: config-volume
      configMap:
        name: zombie-config
```
This should create a `Pod` in the `example` `Namespace` that creates `10` zombie `PIDs`.

## Where are they?
[X] - Detection  
[X] - Emulation  
[_] - Namespace Connection

As you can see, we still don't have a way to fix the underlying issue creating these. In order to do that *safely*, we need to know the `Namespace`, `Pod`, and `Container(s)` these belong to so that we can investigate the underlying application(s) and why this is happening.

I have written a small script that grabs the information we need to continue our investigation.
```bash
#!/usr/bin/bash

is_zombie() {
    local pid=$1
    local state=$(cat /proc/$pid/status 2>/dev/null | grep State | awk '{print $2}')
    if [ "$state" = "Z" ]; then
        return 0
    else
        return 1
    fi
}

get_container_id() {
    local pid=$1
    cat /proc/$pid/cgroup 2>/dev/null | awk -F '-pod' '{print $2}' | awk -F '.slice' '{print $1}' | sed -e 's/_/-/g'
}

get_pod_and_namespace() {
    local container_id=$1
    kubectl --kubeconfig=/var/lib/kubelet/kubeconfig get pods --all-namespaces -o json | jq -r --arg container_id "$container_id" '
        .items[] | 
        select(.metadata.uid == $container_id) | 
        {namespace: .metadata.namespace, name: .metadata.name, containers: [.spec.containers[].name]}'
}

zombie_pids=$(ps -eo pid,state | awk '$2 == "Z" {print $1}')

if [ -z "$zombie_pids" ]; then
    echo "No zombie processes found."
    exit 0
fi

for pid in $zombie_pids; do
    if is_zombie $pid; then
        container_id=$(get_container_id $pid)

        pod_info=$(get_pod_and_namespace $container_id)
        if [ -n "$pod_info" ]; then
            echo "Zombie PID: $pid"
            echo "Container ID: $container_id"
            echo "Pod Information: $pod_info"
        fi
    fi
done
```

Running this from one of the `worker` nodes does indeed print out the information we suspect!
```bash
$ bash zombie-detect.sh
Zombie PID: 555454
Container ID: b5b4bd95-b268-46cc-9b4a-11f116d605b0
Pod Information: {
  "namespace": "example",
  "name": "pod-with-a-zombie-mtf8f",
  "container(s)": [
    "zombie-container"
  ]
}
```

And as an added bonus, I decided to expand the `10` to `100` and a `million`. This was a decent way to demonstrate that these zombie processes aren't free. Just the fact of running `10` of those consumes `~2Mi` more than a simple pod running `sleep`.
```bash
$ kubectl top pods
NAME                         CPU(cores)   MEMORY(bytes)
pod-with-10-zombies          0m           3Mi
pod-with-100-zombies         0m           22Mi
pod-with-a-million-zombies   25m          687Mi
sleepy-pod                   1m           1Mi
```

What you can't see is that my million pod eventually errored out. Oh well, it was just a fun exercise to see how many resources these zombie processes consumed.
```bash
$ kubectl logs pod/pod-with-a-million-zombies | head -n5
/usr/bin/bash: fork: retry: Resource temporarily unavailable
/usr/bin/bash: fork: retry: Resource temporarily unavailable
/usr/bin/bash: fork: retry: Resource temporarily unavailable
/usr/bin/bash: fork: retry: Resource temporarily unavailable
/usr/bin/bash: fork: retry: Resource temporarily unavailable
```


## Wrap-Up
[X] - Detection  
[X] - Emulation  
[X] - Namespace Connection

So here is what I learned, and hopefully you learned something too!
- Detecting - Using `ps` or looking at `/proc` will identify a `pid` as a zombie.
- Emulation - Fairly trivial to reproduce. This is great for testing.
- Namespace Connection - Finding the `Namespace` wasn't so straightforward and requires combining multiple pieces of information.
- Remediation - This is a very important part and one I could not cover. Each application is going to have a different solution to this problem and will need some investigation. Now that you know how to detect them though, you have a path forward.

Also, I want to say thanks for reading. This topic was completely foreign to me a few days ago and I wanted to do a small write-up on it so that I could talk about it with others. I did my best to not misrepresent anything or construct things out of thin air. If there are any suggestions or corrections for this, please open an issue on [Github](https://github.com/t-c-l-o-u-d/blog/issues/new).

---
### References
1. https://man.archlinux.org/man/ps.1
2. https://en.wikipedia.org/wiki/Zombie_process
