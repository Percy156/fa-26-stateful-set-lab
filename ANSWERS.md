# Deployment vs StatefulSet Answers

`kubectl get storageclass` showed `standard (default)` with the `rancher.io/local-path` provisioner, so no Ed post was needed.

**Q1. What do you observe about the pods?**

Two pods came up with random generated names (`note-65886f65df-nxf44` and `note-65886f65df-rxmgj`). Both were created at the same time, sat in `Pending` with the `data-note` claim also `Pending`, then both went `Running` together once the claim was `Bound`. A Deployment treats its replicas as interchangeable, so it starts them in parallel and gives them a ReplicaSet-hash-plus-random-suffix name rather than a stable identity.

**Q2. What do you observe about the note, and about the volumes?**

After writing `hello-from-a` on the first pod, `cat /data/note.txt` on the *other* pod printed `hello-from-a`. `kubectl get pvc` showed only one claim, `data-note`. Both replicas reference the same PVC in the pod template, so they share a single volume. (It works with `ReadWriteOnce` here because kind is a single node and RWO is enforced per node, not per pod.)

**Q3. In what order did the pods start, and what are they named?**

`note-0` was created first and went through `Pending -> ContainerCreating -> Running`; only after it was `Running` did `note-1` get created and start. The names are stable ordinals: `note-0` and `note-1`. A StatefulSet with the default `OrderedReady` policy creates pods one at a time in ordinal order and waits for each to be ready before the next.

**Q4. How does this differ from what you saw in Q2?**

`note-0` read back `hello-from-0` and `note-1` read back `hello-from-1`. The notes did not overwrite each other. `kubectl get pvc` showed two claims, `data-note-0` and `data-note-1`, each bound to a different volume. The `volumeClaimTemplates` give every replica its own PVC, instead of one shared claim like the Deployment.

**Q5. After `note-0` was deleted and came back, what stayed the same? How do these volumes differ from the Deployment?**

The replacement pod had a new UID but the same name, `note-0`, and still read `hello-from-0`; `note-1` was untouched. `data-note-0` and `data-note-1` were unchanged (same volume names) and stayed `Bound` throughout. The StatefulSet reattaches a pod to the PVC for its ordinal, so identity and data survive pod replacement. With the Deployment, there was one claim shared by all replicas, and deleting the Deployment's manifest removed that claim. StatefulSet PVCs are per-replica and are not deleted when pods or the StatefulSet are deleted.
