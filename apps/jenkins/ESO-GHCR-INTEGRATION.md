# External Secrets (ESO) → Vault → GHCR Pull Secret

How Jenkins authenticates to `ghcr.io` to pull `ghcr.io/naidu72/jenkins:lts`,
using External Secrets Operator (ESO) to sync credentials from Vault into a
Kubernetes `dockerconfigjson` Secret.

## Diagram

```mermaid
flowchart TD
    subgraph Vault["Vault (vault.naidu72.info)"]
        KV["KV v2 mount: secret\nsecret/data/ghcr/credentials\n{ username, token }"]
        Policy["Policy: eso-policy\nread/list on secret/data/ghcr/*\nand secret/metadata/ghcr/*"]
        Role["Vault Role: eso-role\n(auth/kubernetes-pi5/role/eso-role)\nbound_service_account_names=external-secrets\nbound_service_account_namespaces=external-secrets\npolicies=eso-policy"]
        Role --> Policy
        Policy --> KV
    end

    subgraph K8s_ExternalSecrets["k8s: external-secrets namespace"]
        SA["ServiceAccount: external-secrets"]
        ESOPod["ESO operator pod"]
        SA --> ESOPod
    end

    subgraph K8s_Jenkins["k8s: jenkins namespace"]
        CSS["ClusterSecretStore: vault-backend-secret\n(09-clustersecretstore-vault-secret.yaml)\nprovider: vault, path: secret\nauth: kubernetes-pi5 / eso-role"]
        ES["ExternalSecret: ghcr-pull-secret\n(08-externalsecret-ghcr.yaml)\nsecretStoreRef -> vault-backend-secret\nremoteRef key: ghcr/credentials"]
        Secret["Secret: ghcr-pull-secret\ntype: kubernetes.io/dockerconfigjson"]
        Pod["Jenkins Pod\nimagePullSecrets:\n- ghcr-pull-secret"]

        CSS --> ES
        ES -- "renders" --> Secret
        Secret -- "imagePullSecrets" --> Pod
    end

    ESOPod -- "1. authenticate via\nk8s JWT (kubernetes-pi5 mount)" --> Role
    Role -- "2. issues Vault token\nwith eso-policy" --> ESOPod
    ESOPod -- "3. reads ghcr/credentials\nusing CSS connection info" --> KV
    ESOPod -- "4. syncs username/token\ninto k8s Secret" --> Secret

    GHCR["ghcr.io"]
    Pod -- "5. kubelet pulls image\nusing ghcr-pull-secret" --> GHCR
```

## Step-by-step

1. **Vault holds the credentials**
   `secret/data/ghcr/credentials` (KV v2, mount `secret`) contains:
   - `username` — GHCR username (`naidu72`)
   - `token` — GitHub PAT with `read:packages` scope

2. **Vault policy `eso-policy`**
   Grants `read`/`list` on `secret/data/ghcr/*` and `secret/metadata/ghcr/*`
   (in addition to the original `kv/data/apps/*`, `kv/data/ssh/*`,
   `kv/data/homelab/*`, `kv/metadata/*` rules).

3. **Vault role `eso-role`** (under the `kubernetes-pi5` auth mount)
   Binds the k8s identity to the policy:
   - Only the `external-secrets` ServiceAccount in the `external-secrets`
     namespace can assume this role.
   - Successful auth issues a Vault token carrying `eso-policy`.

4. **ClusterSecretStore `vault-backend-secret`**
   ([09-clustersecretstore-vault-secret.yaml](base/09-clustersecretstore-vault-secret.yaml))
   Tells ESO *how* to reach Vault: server URL, KV mount (`secret`), and the
   `kubernetes-pi5` / `eso-role` auth config. Cluster-scoped, so any
   namespace's `ExternalSecret` can reference it.

5. **ExternalSecret `ghcr-pull-secret`**
   ([08-externalsecret-ghcr.yaml](base/08-externalsecret-ghcr.yaml))
   References the ClusterSecretStore, pulls `username`/`token` from
   `ghcr/credentials`, and renders a `kubernetes.io/dockerconfigjson` Secret
   named `ghcr-pull-secret` in the `jenkins` namespace. Refreshed hourly.

6. **Jenkins Pod**
   ([03-deployment.yaml](base/03-deployment.yaml)) references
   `imagePullSecrets: [{name: ghcr-pull-secret}]`, so kubelet uses it to
   authenticate the `ghcr.io/naidu72/jenkins:lts` image pull.

## Key takeaway: two independent identities

- **`external-secrets` SA** (namespace `external-secrets`) — used only by the
  ESO operator to authenticate to Vault and produce the `ghcr-pull-secret`
  Secret.
- **`jenkins` SA** (namespace `jenkins`) — used by the Jenkins pod itself for
  in-cluster k8s API access (e.g. the JCasC Kubernetes cloud plugin spawning
  agent pods). It plays **no role** in fetching or using
  `ghcr-pull-secret` — that's purely a kubelet + pod-spec mechanism.
