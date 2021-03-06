# Highly available Persistent Volumes

- How can we achieve true durability?

- How can we store data that would survive the loss of a node?

--

- We need to use Persistent Volumes backed by highly available storage systems

- There are many ways to achieve that:

  - leveraging our cloud's storage APIs

  - using NAS/SAN systems or file servers

  - distributed storage systems

--

- We are going to see one distributed storage system in action

---

## Our test scenario

- We will set up a distributed storage system on our cluster

- We will use it to deploy a SQL database (PostgreSQL)

- We will insert some test data in the database

- We will disrupt the node running the database

- We will see how it recovers

---

## Portworx

- Portworx is a *commercial* persistent storage solution for containers

- It works with Kubernetes, but also Mesos, Swarm ...

- It provides [hyper-converged](https://en.wikipedia.org/wiki/Hyper-converged_infrastructure) storage

  (=storage is provided by regular compute nodes)

- We're going to use it here because it can be deployed on any Kubernetes cluster

  (it doesn't require any particular infrastructure)

- We don't endorse or support Portworx in any particular way

  (but we appreciate that it's super easy to install!)

---

## A useful reminder

- We're installing Portworx because we need a storage system

- If you are using AKS, EKS, GKE ... you already have a storage system

  (but you might want another one, e.g. to leverage local storage)

- If you have setup Kubernetes yourself, there are other solutions available too

  - on premises, you can use a good old SAN/NAS

  - on a private cloud like OpenStack, you can use e.g. Cinder

  - everywhere, you can use other systems, e.g. Gluster, StorageOS

---

## Installing Portworx

- Portworx installation is relatively simple

- ... But we made it *even simpler!*

- We are going to use a YAML manifest that will take care of everything

- Warning: this manifest is customized for a very specific setup

  (like the VMs that we provide during workshops and training sessions)

- It will probably *not work* If you are using a different setup

  (like Docker Desktop, k3s, MicroK8S, Minikube ...)

---

## The simplified Portworx installer

- The Portworx installation will take a few minutes

- Let's start it, then we'll explain what happens behind the scenes

.exercise[

- Install Portworx:
  ```bash
  kubectl apply -f ~/container.training/k8s/portworx.yaml
  ```

]

<!-- ##VERSION ## -->

*Note: this was tested with Kubernetes 1.18. Newer versions may or may not work.*

---

class: extra-details

## What's in this YAML manifest?

- Portworx installation itself, pre-configured for our setup

- A default *Storage Class* using Portworx

- A *Daemon Set* to create loop devices on each node of the cluster

---

class: extra-details

## Portworx installation

- The official way to install Portworx is to use [PX-Central](https://central.portworx.com/)

  (this requires a free account)

- PX-Central will ask us a few questions about our cluster

  (Kubernetes version, on-prem/cloud deployment, etc.)

- Using our answers, it will generate a YAML manifest that we can use

---

class: extra-details

## Portworx storage configuration

- Portworx needs at least one *block device*

- Block device = disk or partition on a disk

- We can see block devices with `lsblk`

  (or `cat /proc/partitions` if we're old school like that!)

- If we don't have a spare disk or partition, we can use a *loop device*

- A loop device is a block device actually backed by a file

- These are frequently used to mount ISO (CD/DVD) images or VM disk images

---

class: extra-details

## Setting up a loop device

- Our `portworx.yaml` manifest includes a *Daemon Set* that will:

  - create a 10 GB (empty) file on each node

  - load the `loop` module (if it's not already loaded)

  - associate a loop device with the 10 GB file

- After these steps, we have a block device that Portworx can use

---

class: extra-details

## Implementation details

- The file is `/portworx.blk`

  (it is a [sparse file](https://en.wikipedia.org/wiki/Sparse_file) created with `truncate`)

- The loop device is `/dev/loop4`

- This can be verified by running `sudo losetup`

- The *Daemon Set* uses a privileged *Init Container*

- We can check the logs of that container with:
  ```bash
    kubectl logs --selector=app=setup-loop4-for-portworx \
            -c setup-loop4-for-portworx
  ```

---

## Waiting for Portworx to be ready

- The installation process will take a few minutes

.exercise[

- Check out the logs:
  ```bash
  stern -n kube-system portworx
  ```

- Wait until it gets quiet

  (you should see `portworx service is healthy`, too)

<!--
```longwait PX node status reports portworx service is healthy```
```key ^C```
-->

]

---

## Dynamic provisioning of persistent volumes

- We are going to run PostgreSQL in a Stateful set

- The Stateful set will specify a `volumeClaimTemplate`

- That `volumeClaimTemplate` will create Persistent Volume Claims

- Kubernetes' [dynamic provisioning](https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/) will satisfy these Persistent Volume Claims

  (by creating Persistent Volumes and binding them to the claims)

- The Persistent Volumes are then available for the PostgreSQL pods

---

## Storage Classes

- It's possible that multiple storage systems are available

- Or, that a storage system offers multiple tiers of storage

  (SSD vs. magnetic; mirrored or not; etc.)

- We need to tell Kubernetes *which* system and tier to use

- This is achieved by creating a Storage Class

- A `volumeClaimTemplate` can indicate which Storage Class to use

- It is also possible to mark a Storage Class as "default"

  (it will be used if a `volumeClaimTemplate` doesn't specify one)

---

## Check our default Storage Class

- The YAML manifest applied earlier should define a default storage class

.exercise[

- Check that we have a default storage class:
  ```bash
  kubectl get storageclass
  ```

]

There should be a storage class showing as `portworx-replicated (default)`.

---

class: extra-details

## Our default Storage Class

This is our Storage Class (in `k8s/storage-class.yaml`):

```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1beta1
metadata:
  name: portworx-replicated
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: kubernetes.io/portworx-volume
parameters:
 repl: "2"
 priority_io: "high"
```

- It says "use Portworx to create volumes and keep 2 replicas of these volumes"

- The annotation makes this Storage Class the default one

---

## Our Postgres Stateful set

- The next slide shows `k8s/postgres.yaml`

- It defines a Stateful set

- With a `volumeClaimTemplate` requesting a 1 GB volume

- That volume will be mounted to `/var/lib/postgresql/data`

- There is another little detail: we enable the `stork` scheduler

- The `stork` scheduler is optional (it's specific to Portworx)

- It helps the Kubernetes scheduler to colocate the pod with its volume

  (see [this blog post](https://portworx.com/stork-storage-orchestration-kubernetes/) for more details about that)

---

.small[
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  selector:
    matchLabels:
      app: postgres
  serviceName: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      schedulerName: stork
      containers:
      - name: postgres
        image: postgres:12
        env:
        - name: POSTGRES_HOST_AUTH_METHOD
          value: trust
        volumeMounts:
        - mountPath: /var/lib/postgresql/data
          name: postgres
  volumeClaimTemplates:
  - metadata:
      name: postgres
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
```
]

---

## Creating the Stateful set

- Before applying the YAML, watch what's going on with `kubectl get events -w`

.exercise[

- Apply that YAML:
  ```bash
  kubectl apply -f ~/container.training/k8s/postgres.yaml
  ```

<!-- ```hide kubectl wait pod postgres-0 --for condition=ready``` -->

]

---

## Testing our PostgreSQL pod

- We will use `kubectl exec` to get a shell in the pod

- Good to know: we need to use the `postgres` user in the pod

.exercise[

- Get a shell in the pod, as the `postgres` user:
  ```bash
  kubectl exec -ti postgres-0 -- su postgres
  ```

<!--
autopilot prompt detection expects $ or # at the beginning of the line.
```wait postgres@postgres```
```keys PS1="\u@\h:\w\n\$ "```
```key ^J```
-->

- Check that default databases have been created correctly:
  ```bash
  psql -l
  ```

]

(This should show us 3 lines: postgres, template0, and template1.)

---

## Inserting data in PostgreSQL

- We will create a database and populate it with `pgbench`

.exercise[

- Create a database named `demo`:
  ```bash
  createdb demo
  ```

- Populate it with `pgbench`:
  ```bash
  pgbench -i demo
  ```

]

- The `-i` flag means "create tables"

- If you want more data in the test tables, add e.g. `-s 10` (to get 10x more rows)

---

## Checking how much data we have now

- The `pgbench` tool inserts rows in table `pgbench_accounts`

.exercise[

- Check that the `demo` base exists:
  ```bash
  psql -l
  ```

- Check how many rows we have in `pgbench_accounts`:
  ```bash
  psql demo -c "select count(*) from pgbench_accounts"
  ```

- Check that `pgbench_history` is currently empty:
  ```bash
  psql demo -c "select count(*) from pgbench_history"
  ```

]

---

## Testing the load generator

- Let's use `pgbench` to generate a few transactions

.exercise[

- Run `pgbench` for 10 seconds, reporting progress every second:
  ```bash
  pgbench -P 1 -T 10 demo
  ```

- Check the size of the history table now:
  ```bash
  psql demo -c "select count(*) from pgbench_history"
  ```

]

Note: on small cloud instances, a typical speed is about 100 transactions/second.

---

## Generating transactions

- Now let's use `pgbench` to generate more transactions

- While it's running, we will disrupt the database server

.exercise[

- Run `pgbench` for 10 minutes, reporting progress every second:
  ```bash
  pgbench -P 1 -T 600 demo
  ```

- You can use a longer time period if you need more time to run the next steps

<!-- ```tmux split-pane -h``` -->

]

---

## Find out which node is hosting the database

- We can find that information with `kubectl get pods -o wide`

.exercise[

- Check the node running the database:
  ```bash
  kubectl get pod postgres-0 -o wide
  ```

]

We are going to disrupt that node.

--

By "disrupt" we mean: "disconnect it from the network".

---

## Disconnect the node

- We will use `iptables` to block all traffic exiting the node

  (except SSH traffic, so we can repair the node later if needed)

.exercise[

- SSH to the node to disrupt:
  ```bash
  ssh `nodeX`
  ```

- Allow SSH traffic leaving the node, but block all other traffic:
  ```bash
  sudo iptables -I OUTPUT -p tcp --sport 22 -j ACCEPT
  sudo iptables -I OUTPUT 2 -j DROP
  ```

]

---

## Check that the node is disconnected

.exercise[

- Check that the node can't communicate with other nodes:
  ```
  ping node1
  ```

- Logout to go back on `node1`

<!-- ```key ^D``` -->

- Watch the events unfolding with `kubectl get events -w` and `kubectl get pods -w`

]

- It will take some time for Kubernetes to mark the node as unhealthy

- Then it will attempt to reschedule the pod to another node

- In about a minute, our pod should be up and running again

---

## Check that our data is still available

- We are going to reconnect to the (new) pod and check

.exercise[

- Get a shell on the pod:
  ```bash
  kubectl exec -ti postgres-0 -- su postgres
  ```

<!--
```wait postgres@postgres```
```keys PS1="\u@\h:\w\n\$ "```
```key ^J```
-->

- Check how many transactions are now in the `pgbench_history` table:
  ```bash
  psql demo -c "select count(*) from pgbench_history"
  ```

<!-- ```key ^D``` -->

]

If the 10-second test that we ran earlier gave e.g. 80 transactions per second,
and we failed the node after 30 seconds, we should have about 2400 row in that table.

---

## Double-check that the pod has really moved

- Just to make sure the system is not bluffing!

.exercise[

- Look at which node the pod is now running on
  ```bash
  kubectl get pod postgres-0 -o wide
  ```

]

---

## Re-enable the node

- Let's fix the node that we disconnected from the network

.exercise[

- SSH to the node:
  ```bash
  ssh `nodeX`
  ```

- Remove the iptables rule blocking traffic:
  ```bash
  sudo iptables -D OUTPUT 2
  ```

]

---

class: extra-details

## A few words about this PostgreSQL setup

- In a real deployment, you would want to set a password

- This can be done by creating a `secret`:
  ```
  kubectl create secret generic postgres \
          --from-literal=password=$(base64 /dev/urandom | head -c16)
  ```

- And then passing that secret to the container:
  ```yaml
  env:
  - name: POSTGRES_PASSWORD
    valueFrom:
      secretKeyRef:
        name: postgres
        key: password
  ```

---

class: extra-details

## Troubleshooting Portworx

- If we need to see what's going on with Portworx:
  ```
  PXPOD=$(kubectl -n kube-system get pod -l name=portworx -o json |
  	      jq -r .items[0].metadata.name)
  kubectl -n kube-system exec $PXPOD -- /opt/pwx/bin/pxctl status
  ```

- We can also connect to Lighthouse (a web UI)

  - check the port with `kubectl -n kube-system get svc px-lighthouse`

  - connect to that port

  - the default login/password is `admin/Password1`

  - then specify `portworx-service` as the endpoint

---

class: extra-details

## Removing Portworx

- Portworx provides a storage driver

- It needs to place itself "above" the Kubelet

  (it installs itself straight on the nodes)

- To remove it, we need to do more than just deleting its Kubernetes resources

- It is done by applying a special label:
  ```
  kubectl label nodes --all px/enabled=remove --overwrite
  ```

- Then removing a bunch of local files:
  ```
  sudo chattr -i /etc/pwx/.private.json
  sudo rm -rf /etc/pwx /opt/pwx
  ```

  (on each node where Portworx was running)

---

class: extra-details

## Dynamic provisioning without a provider

- What if we want to use Stateful sets without a storage provider?

- We will have to create volumes manually

  (by creating Persistent Volume objects)

- These volumes will be automatically bound with matching Persistent Volume Claims

- We can use local volumes (essentially bind mounts of host directories)

- Of course, these volumes won't be available in case of node failure

- Check [this blog post](https://kubernetes.io/blog/2018/04/13/local-persistent-volumes-beta/) for more information and gotchas

---

## Acknowledgements

The Portworx installation tutorial, and the PostgreSQL example,
were inspired by [Portworx examples on Katacoda](https://katacoda.com/portworx/scenarios/), in particular:

- [installing Portworx on Kubernetes](https://www.katacoda.com/portworx/scenarios/deploy-px-k8s)

  (with adapatations to use a loop device and an embedded key/value store)

- [persistent volumes on Kubernetes using Portworx](https://www.katacoda.com/portworx/scenarios/px-k8s-vol-basic)

  (with adapatations to specify a default Storage Class)

- [HA PostgreSQL on Kubernetes with Portworx](https://www.katacoda.com/portworx/scenarios/px-k8s-postgres-all-in-one)

  (with adaptations to use a Stateful Set and simplify PostgreSQL's setup)

???

:EN:- Using highly available persistent volumes
:EN:- Example: deploying a database that can withstand node outages

:FR:- Utilisation de volumes ?? haute disponibilit??
:FR:- Exemple : d??ployer une base de donn??es survivant ?? la d??faillance d'un n??ud
