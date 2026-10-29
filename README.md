# Mandiri DCS Provider Workload Cluster Upgrade

把已经在跑的 **`dcs-workloadcluster`** 从 ACP **v4.3.2** 升到 **v4.4.1**。

这不是创建集群。创建仓是旁边的 `DCS-Provider-workloadcluster-deploy`。本仓库只放 Phase 2 要 **新建** 的模板，以及 KCP / MD **对照改哪些字段**。

把本仓库拷到 Global Master 01。一次只 apply 一个允许 apply 的文件。不要脚本、不要 envsubst。带 `<...>` 的 YAML 禁止 apply。文件名带 `.edit.yaml` 的 **永远不要 apply**。

官方文档：

- 业务集群 CVO（Phase 1）：<https://docs.alauda.io/container_platform/4.4/upgrade/upgrade_workload_cluster.html>
- 升级前准备 / preflight：<https://docs.alauda.io/container_platform/4.4/upgrade/pre-upgrade.html>
- DCS Phase 2（Kube-OVN → 控面 → Worker）：<https://docs.alauda.io/immutable-infra/1.0/upgrade-cluster/huawei-dcs.html>
- OS 支持矩阵：<https://docs.alauda.io/immutable-infra/1.0/overview/os-support-matrix.html>
- 1.35 kubelet patch：<https://docs.alauda.io/immutable-infra/1.0/upgrade-cluster/index.html>

## 总顺序（闸门）

后面一步没到闸门，不要做。不要跳。

```text
0. Global 已经 4.4.1 Ready
   ↓ 没到：不要动 dcs-workloadcluster
1. Workload preflight 全过
   ↓ 没过：不要提交 desiredUpdate
2. Phase 1：CVO 把 dcs-workloadcluster 升到 4.4.1 Ready
   ↓ 没 Ready：不要 annotate kube-ovn，不要改 KCP
3. Phase 2 前：kube-ovn chart v4.4.12，AppRelease Success
   ↓ 没 Success：不要改 KCP
4. apply 新 CP DCSMachineTemplate（还不会换机）
5. 一次 edit KCP（这里才滚动控面 OS / Kubernetes）
   ↓ 三台 CP 不是 v1.35.6-acp.9 Running：不要动 Worker
6. apply 新 Worker 模板 + 新 KCT（还不会换机）
7. 一次 edit MachineDeployment（这里才滚动 Worker）
   ↓ Worker 没全部 Ready：不要删旧模板，不要升 Agnostic
8. Marketplace 升本集群 Agnostic 插件
```

**不要做：**

- 不要 apply IP Pool / Secret / DCSCluster / Cluster（已经在集群里）
- 不要原地改旧模板的 `vmTemplateName`
- 不要只改 `version`、过一会儿再换模板 / coredns / etcd / kubelet patch
- 不要 `kubectl apply -f cp/` 或 `kubectl apply -f worker/` 一把梭
- 不要 apply 任何 `.edit.yaml`
- 不要把 worker 的 pool 写成 `dcs-workloadcluster-cp-ippool`
- 不要从 KCP 抄 coredns / etcd 到 MD
- 控面已经有一台目标小版本 Machine join 之后，不要把 Kubernetes 改回去

`PROVIDER_ID` 和 `NODE_IP` 不要改。`maxSurge` 保持 `0`（pool-managed `/var/cpaas`）。

## 版本和现场值

```text
源 ACP / K8s:     v4.3.2 / v1.34.5-3
目标 ACP / K8s:   v4.4.1 / v1.35.6-acp.9
coredns:          1.14.7-v4.4.7          # 只写 KCP
etcd:             v3.6.13-acp.2          # 只写 KCP
kube-ovn chart:   v4.4.12
DCS Provider:     v1.0.22 或更高（kube-ovn v4.4+ 必需）
旧 VM 模板:       slem-alaudaos-vda
新 VM 模板:       441slem-alaudaos-vda   # 先在 DCS 上存在
```

所有 `kubectl` 在 Global Master 01 执行，对象都在 `cpaas-system`。对业务集群节点用它的 kubeconfig，不要切走 Global context。

| 项 | 本环境 | 升级时 |
|---|---|---|
| 集群名 | `dcs-workloadcluster` | 不要改名 |
| KCP | `dcs-workloadcluster-kcp` | **edit**，不要重建 |
| 旧 CP 模板 | `dcs-workloadcluster-cp-template` | 先留着 |
| 新 CP 模板 | `dcs-workloadcluster-cp441-template` | apply `cp/01` |
| MD | `dcs-workloadcluster-worker-deployment` | **edit**，不要重建 |
| 旧 Worker 模板 | `dcs-workloadcluster-worker-template` | 先留着 |
| 新 Worker 模板 | `dcs-workloadcluster-worker441-template` | apply `worker/01` |
| 旧 Worker KCT | `dcs-workloadcluster-worker-kct` | 先留着 |
| 新 Worker KCT | `dcs-workloadcluster-worker441-kct` | apply `worker/02` |
| CP pool | `dcs-workloadcluster-cp-ippool` | **不要动** |
| Worker pool | `dcs-workloadcluster-worker-ippool` | **不要动** |
| Folder / DVS / PG | `ACP_Cluster` / `ManagementDVS` / `VLAN-329` | 保持 |
| Datastore | `jkt01-POC-DEV-DCS-01` | 用 `datastoreName` |
| CP 规格 | 16C32G + etcd 10G | 保持 |
| Worker 规格 | 8C16G，无 etcd | 保持 |
| Registry | `10.243.166.5:11443` | 不要改 |

## 目录

```text
cp/       控制面：新模板（apply）+ KCP 对照（禁止 apply）
worker/   Worker：新模板 + 新 KCT（apply）+ MD 对照（禁止 apply）
```

### `cp/`

| 文件 | 角色 | apply？ |
|---|---|---|
| `01-dcs-cp441-machine-template.yaml` | 新 CP `DCSMachineTemplate` | 要（闸门 3 过了才能） |
| `02-kubeadm-control-plane.edit.yaml` | KCP 要改的字段对照 | **禁止 apply** |

### `worker/`

| 文件 | 角色 | apply？ |
|---|---|---|
| `01-dcs-worker441-machine-template.yaml` | 新 Worker `DCSMachineTemplate` | 要（闸门 5 过了才能） |
| `02-worker441-kubeadm-config-template.yaml` | 新 Worker `KubeadmConfigTemplate` | 要（闸门 5 过了才能） |
| `03-worker-machine-deployment.edit.yaml` | MD 要改的字段对照 | **禁止 apply** |

---

## 0. Global 必须先 4.4.1

业务集群只能升到 Global **已经到达** 的 Distribution Version。

```bash
kubectl -n cpaas-system get cvsh global
kubectl -n cpaas-system get cvsh dcs-workloadcluster
```

`global` 必须 `CURRENT=4.4.1` 且 Ready。不是就停，先完成 Global。本仓库不管 Global。

DCS Provider 必须已经是 **v1.0.22+**（管理集群 AppRelease，chart 名带 `chart-cluster-api-provider-dcs`）。kube-ovn chart `v4.4+` 不能靠手工改 AppRelease 源来凑。

[DCS 控制台] 模板 **`441slem-alaudaos-vda`** 必须已经在，和创建时用的 Folder / DVS / Port Group / Datastore 同一套。

---

## 1. Workload preflight（先做这个）

**When：** 提交 CVO 升级之前。没过不要提交。

```bash
bash upgrade.sh --cluster=dcs-workloadcluster --preflight
```

Preflight 只检查，不改集群。不过就按 [Upgrade Troubleshooting](https://docs.alauda.io/container_platform/4.4/upgrade/troubleshooting.html) 处理。本环境常见：`ResourcePatch` 要标

```text
config.cpaas.io/exempt-for-ver=4.4.1
```

值是 **`4.4.1`，不要加 `v`**。匹配 major.minor。

还要确认模块稳、集群健康：

```bash
kubectl get moduleinfo -l cpaas.io/cluster-name=dcs-workloadcluster \
  -o custom-columns='MODULE:.metadata.labels.cpaas\.io/module-name,CURRENT:.status.version,TARGET:.spec.version,PHASE:.status.phase'

kubectl get clustermodule dcs-workloadcluster \
  -o jsonpath='{.status.base.deployStatus}{"\n"}'
```

每个 Core / Aligned 模块 `PHASE=Running`。`deployStatus` 不能是 `Upgrading` / `DeployFailed` / `UpgradeFailed`。

---

## 2. Phase 1：CVO 升 Distribution Version

**闸门：** 步骤 1 全过。这里还 **不** 换 OS、**不** 改 KCP。

入口三选一（等价）：Console 集群页 Upgrade、Fleet Essentials、或：

```bash
ac adm upgrade --cluster=dcs-workloadcluster --to=4.4.1
```

确认框里的 **Maintenance mode 是平台级**，不是这个业务集群自己的。不打算锁全平台就关掉再提交。

提交后只观察现有请求，不要反复写 `desiredUpdate`，不要删 AppRelease。

```bash
kubectl -n cpaas-system get cvsh dcs-workloadcluster
kubectl -n cpaas-system get cvsh dcs-workloadcluster -o yaml
```

过闸门：`CURRENT=4.4.1`，Ready，不再 Reconciling / Stalled。没到不要做步骤 3。

etcd 按 ACP 支持的备份流程做一份并验证，再进 Phase 2。

---

## 3. Phase 2 前：先升 Kube-OVN

**闸门：** 步骤 2 Ready。**在改 KCP 之前。**

本环境 DCS Provider `v1.0.22+`，只 annotate，不要手工 patch `cni-kube-ovn` 的 chart 源：

```bash
kubectl -n cpaas-system get dcscluster dcs-workloadcluster \
  -o jsonpath='{.spec.networkType}{"\n"}'
# 期望：kube-ovn

kubectl annotate cluster dcs-workloadcluster -n cpaas-system \
  cpaas.io/kube-ovn-version=v4.4.12 --overwrite
```

在 **业务集群** kubeconfig 上查 AppRelease（不要只看 installedRevision）：

```bash
kubectl --kubeconfig /tmp/dcs-workloadcluster-kubeconfig \
  -n cpaas-system get apprelease cni-kube-ovn \
  -o jsonpath='Chart: {.spec.source.charts[0].name}{"\n"}Target: {.spec.source.charts[0].targetRevision}{"\n"}Installed: {.status.charts.*.installedRevision}{"\n"}Phase: {.status.charts.*.phase}{"\n"}'
```

过闸门：

- chart 名 `acp/chart-kube-ovn`（v4.4+，不是旧的 `chart-cpaas-kube-ovn`）
- target 和 installed 都是 `v4.4.12`
- `phase=Success`
- Sync / Health 为 True

`HealthChecking` 不算过。`DownloadFailed` / `DeployFailed` / `NotReady` 就停。

---

## 4. 新 CP 模板（apply，还不会换机）

**闸门：** 步骤 3 Success。只 apply 模板，**先不要 edit KCP**。

文件：`cp/01-dcs-cp441-machine-template.yaml`

本次 **改过** 的字段：

| 字段 | 值 |
|---|---|
| `metadata.name` | `dcs-workloadcluster-cp441-template` |
| `vmTemplateName` | `441slem-alaudaos-vda` |

**保持不动：** `ipHostPoolRef=dcs-workloadcluster-cp-ippool`，16C / 32768，etcd 10G，Folder / DVS / PG / datastore。

```bash
grep -nE 'name:|vmTemplateName:|ipHostPoolRef|etcd|resourceVersion|uid:|providerID|worker-ippool' \
  cp/01-dcs-cp441-machine-template.yaml

kubectl apply --dry-run=server -f cp/01-dcs-cp441-machine-template.yaml
kubectl apply -f cp/01-dcs-cp441-machine-template.yaml
kubectl -n cpaas-system get dcsmachinetemplate
```

应同时有旧 `dcs-workloadcluster-cp-template` 和新 `dcs-workloadcluster-cp441-template`。旧的先留着。

---

## 5. 一次改 KCP（这里才滚动控面）

**闸门：** `cp/01` 已在集群里，步骤 3 kube-ovn Success。文件 `cp/02-kubeadm-control-plane.edit.yaml` **禁止 apply**。

```bash
kubectl -n cpaas-system get kubeadmcontrolplane dcs-workloadcluster-kcp \
  -o yaml | grep -nE 'version:|infrastructureRef:|imageTag:|maxSurge|NeverVerify'
```

```bash
kubectl -n cpaas-system edit kubeadmcontrolplane dcs-workloadcluster-kcp
```

**一次保存改齐：**

| 字段 | 从 | 到 |
|---|---|---|
| `spec.version` | `v1.34.5-3` | `v1.35.6-acp.9` |
| `machineTemplate.infrastructureRef.name` | `dcs-workloadcluster-cp-template` | `dcs-workloadcluster-cp441-template` |
| `clusterConfiguration.dns.imageTag` | `1.14.2-v4.3.11` | `1.14.7-v4.4.7` |
| `clusterConfiguration.etcd.local.imageTag` | `v3.5.28-260625` | `v3.6.13-acp.2` |
| kubelet JSON | 无此项 | `"imagePullCredentialsVerificationPolicy": "NeverVerify"` |
| `rolloutStrategy.rollingUpdate.maxSurge` | `0` | **保持 0** |

kubelet 文件路径：`/etc/kubernetes/patches/kubeletconfiguration0+strategic.json`。在原有 JSON 里加一行，不要把 `protectKernelDefaults` / tls / `clientCAFile` 丢掉。若该 file 是 `contentFrom.secret`，复制成新 Secret 再改，不要改正在用的旧 Secret。

**不要动：** SSH 公钥、PSA、audit、`preKubeadmCommands` 三句、`PROVIDER_ID` / `NODE_IP`、`replicas: 3`。

保存后：

```bash
kubectl -n cpaas-system get kubeadmcontrolplane dcs-workloadcluster-kcp -w
kubectl -n cpaas-system get machines \
  -l cluster.x-k8s.io/cluster-name=dcs-workloadcluster,cluster.x-k8s.io/control-plane
```

期望：一台删、一台用 `441slem-alaudaos-vda` 新建。`maxSurge=0` 所以一次一台。

过闸门（三台都要）：

```bash
kubectl -n cpaas-system get machines \
  -l cluster.x-k8s.io/cluster-name=dcs-workloadcluster,cluster.x-k8s.io/control-plane \
  -o custom-columns='NAME:.metadata.name,VER:.spec.version,PHASE:.status.phase'
```

`VER=v1.35.6-acp.9` 且 `PHASE=Running`。业务集群节点 Ready。没齐不要做步骤 6 / 7。

已经有一台目标小版本 CP Machine **join** 之后：不要把 version / coredns / etcd 改回 1.34。只能向前修，或按 ACP 备份恢复。

---

## 6. 新 Worker 模板 + 新 KCT（apply，还不会换机）

**闸门：** 步骤 5 三台 CP 都是 `v1.35.6-acp.9` Running。只 apply，**先不要 edit MD**。

### 6.1 模板

文件：`worker/01-dcs-worker441-machine-template.yaml`

| 字段 | 值 |
|---|---|
| `metadata.name` | `dcs-workloadcluster-worker441-template` |
| `vmTemplateName` | `441slem-alaudaos-vda` |
| `ipHostPoolRef.name` | `dcs-workloadcluster-worker-ippool` |

**保持：** 8C / 16384，kubelet + containerd 各 100G，**没有** `/var/lib/etcd`。

```bash
grep -nE 'name:|vmTemplateName:|ipHostPoolRef|etcd|cp-ippool|providerID' \
  worker/01-dcs-worker441-machine-template.yaml

kubectl apply --dry-run=server -f worker/01-dcs-worker441-machine-template.yaml
kubectl apply -f worker/01-dcs-worker441-machine-template.yaml
```

### 6.2 KCT

文件：`worker/02-worker441-kubeadm-config-template.yaml`

Kubernetes 1.35：**必须换名字**，不能改旧 KCT 原地。

**先**从现网把公钥抄进这个文件，去掉 `<ssh-authorized-keys>`：

```bash
kubectl -n cpaas-system get kubeadmconfigtemplate dcs-workloadcluster-worker-kct \
  -o jsonpath='{.spec.template.spec.users[0].sshAuthorizedKeys[0]}{"\n"}'
```

写公钥文本本身，不要写路径。必须和现网 worker / CP 同一把。`PROVIDER_ID` / `NODE_IP` 不要改。不要写 `>> /etc/hosts`。

```bash
grep -nE '<[^>]+>|填写实际' worker/02-worker441-kubeadm-config-template.yaml
kubectl apply --dry-run=server -f worker/02-worker441-kubeadm-config-template.yaml
kubectl apply -f worker/02-worker441-kubeadm-config-template.yaml
kubectl -n cpaas-system get kubeadmconfigtemplate
```

---

## 7. 一次改 MachineDeployment（这里才滚动 Worker）

**闸门：** 步骤 5 控面齐；步骤 6 两个新对象都在。文件 `worker/03-worker-machine-deployment.edit.yaml` **禁止 apply**。

```bash
kubectl -n cpaas-system get machinedeployment dcs-workloadcluster-worker-deployment \
  -o yaml | grep -nE 'version:|infrastructureRef:|configRef:|maxSurge|replicas'
```

`maxSurge` 必须是 **0**。

```bash
kubectl -n cpaas-system edit machinedeployment dcs-workloadcluster-worker-deployment
```

**一次保存改齐：**

| 字段 | 到 |
|---|---|
| `spec.template.spec.version` | `v1.35.6-acp.9` |
| `infrastructureRef.name` | `dcs-workloadcluster-worker441-template` |
| `bootstrap.configRef.name` | `dcs-workloadcluster-worker441-kct` |
| `strategy.rollingUpdate.maxSurge` | **保持 0** |
| `maxUnavailable` | 现网有就保持 |

Worker **没有** coredns / etcd `imageTag`。

```bash
kubectl -n cpaas-system get machinedeployment dcs-workloadcluster-worker-deployment -w
kubectl -n cpaas-system get machines \
  -l cluster.x-k8s.io/deployment-name=dcs-workloadcluster-worker-deployment -w
```

期望：一台删、一台用 `441slem-alaudaos-vda` 新建。

过闸门：MD Ready；Worker Machine Running；业务集群节点 Ready 且 `v1.35.6-acp.9`，label `kube-ovn/role=worker`。

旧 CP / Worker 模板和旧 KCT **先不要删**，直到新节点全部健康。

---

## 8. Agnostic 插件（最后）

**闸门：** 步骤 7 Worker 全部目标版本 Ready。CVO 不管 Agnostic。

Console → Marketplace → Cluster Plugins（切到 `dcs-workloadcluster`），按各插件发行说明和 Kubernetes 1.35 兼容性逐个升。

---

## 对照：创建仓有、本仓没有

这些对象升级时 **不要再 apply**：

- DCS Secret、CP / Worker IP Pool
- `DCSCluster`、CAPI `Cluster`
- 旧 `dcs-workloadcluster-cp-template` / `dcs-workloadcluster-worker-template` / `dcs-workloadcluster-worker-kct`

IP、VIP、CIDR、site、规格、`maxSurge=0` 都保持创建时的值。变的是：新模板名、新 KCT 名、K8s / coredns / etcd / kube-ovn / NeverVerify。

---

## 失败时（Phase 2）

1. **还没有**目标小版本的 CP Machine：可以恢复 kube-ovn 注解和 KCP / MD 的旧字段，取消这次滚动。
2. 只换了模板 / OS、Kubernetes **小版本没变**：把 ref 指回旧模板，会再滚动一轮。
3. 已经有目标小版本 CP Machine **join**：不要降级 Kubernetes / coredns / etcd。停滚动，向前修，或用升级前备份恢复。

旧 VM 已经被删掉。模板恢复是新建替换机，不是把原 VM 救回来。pool-managed 盘会按同一 IP 槽挂回，升级窗口里写进这些盘的数据会留下。
