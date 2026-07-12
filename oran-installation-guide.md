# O-RAN Implementation Guide (Ubuntu 20.04)

本文件整理自 `oran_implementation.pptx` 簡報中「O-RAN implementation guide on ubuntu 20.04」章節,涵蓋 K8s/Helm/Docker、Near-RT RIC、Non-RT RIC 的安裝流程,以及 Anomaly Detection (AD) Use Case 的 xApp 部署驗證。

## 目錄

1. [安裝 K8s, Helm, Docker](#1-安裝-k8s-helm-docker)
2. [安裝 Near-RT RIC](#2-安裝-near-rt-ric)
3. [安裝 Non-RT RIC](#3-安裝-non-rt-ric)
4. [Anomaly Detection Use Case](#4-anomaly-detection-use-case)
5. [版本資訊](#5-版本資訊)
6. [參考資料](#6-參考資料)

---

## 1. 安裝 K8s, Helm, Docker

### 1.1 Clone ric-dep

需要 clone 兩個不同版本的 `ric-dep` 資料夾至不同路徑:

```bash
# 路徑 1(用於實際安裝)
git clone --branch f-release https://gerrit.o-ran-sc.org/r/ric-plt/ric-dep /Home/ric-dep

# 路徑 2(僅用於取代腳本)
git clone https://gerrit.o-ran-sc.org/r/ric-plt/ric-dep /root/ric-dep
```

再將**路徑 2** 的 `install_k8s_and_helm.sh` 檔案取代**路徑 1** 內同名的檔案。

之後利用路徑 1 的資料夾安裝 k8s (1.19.11)、helm (3.5.4)、docker 及 Near-RT RIC (f-release)。

### 1.2 修改安裝腳本

編輯 `/root/ric-dep/bin/install_k8s_and_helm.sh`:

- 修改 k8s 版本至 `1.19.11`
- 將 flannel 網址改為(較新版本改用 `kube-flannel` namespace,故沿用舊版網址):

  ```bash
  # we refer to version 0.18.1 because later versions use namespace kube-flannel instead of kube-system
  kubectl apply -f "https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml"
  ```

- 將等待 pods 數量改為 7 個:

  ```bash
  wait_for_pods_running 7 kube-system
  ```

### 1.3 執行安裝

```bash
cd /root/ric-dep/bin
./install_k8s_and_helm.sh
./install_common_templates_to_helm.sh
./setup-ric-common-template
```

### 1.4 檢查版本

```bash
docker version   # Client/Server Version: 20.10.21
helm version      # v3.5.4
kubectl version   # v1.19.11
```

---

## 2. 安裝 Near-RT RIC

### 2.1 修改 tiller 版本

修改以下兩個檔案內 `tiller` 的 tag version 為 `v2.12.3`:

- `ric-dep/helm/appmgr/values.yaml`
- `ric-dep/helm/infrastructure/values.yaml`

```yaml
tiller:
  registry: ghcr.io
  name: helm/tiller
  tag: v2.12.3
```

### 2.2 建立 NFS storage

```bash
kubectl create ns ricinfra
helm repo add stable https://charts.helm.sh/stable
helm install nfs-release-1 stable/nfs-server-provisioner --namespace ricinfra
kubectl patch storageclass nfs -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
apt install nfs-common
```

### 2.3 加入 InfluxDB 元件

在 `dep/ric-dep/bin/install` 檔案的 `COMPONENTS` 這行加入 `influxdb`:

```bash
COMPONENTS="infrastructure dbaas appmgr rtmgr e2mgr e2term a1mediator submgr vespamgr o1mediator alarmmanager influxdb $LIST_OF_COMPONENTS"
```

### 2.4 修正 InfluxDB 授權問題

當 xApp 要使用 influxDB 資源、卻未設定授權選項時會出現錯誤。需進入:

```
ric-dep/helm/influxdb/values.yaml
```

將 `auth-enabled: true` 改為 `auth-enabled: false`。

### 2.5 安裝 RIC platform (Near-RT RIC)

```bash
./install -f ../RECIPE_EXAMPLE/example_recipe_latest_stable.yaml

# 查看 pods 狀態
kubectl get pods -n ricplt
```

---

## 3. 安裝 Non-RT RIC

```bash
git clone https://gerrit.o-ran-sc.org/r/it/dep

# 安裝 Non-RT RIC
dep/bin/deploy-nonrtric -f dep/nonrtric/RECIPE_EXAMPLE/example_recipe.yaml

# 查看 pods 狀態
kubectl get pods -n nonrtric
```

---

## 4. Anomaly Detection Use Case

xApp 部署順序:**AD → TS → QP**(Anomaly Detection → Traffic Steering → QoE Predictor)

xApp 佈署 3 步驟:取得 xApp → 上架 xApp → 安裝 xApp

### 4.1 安裝 dms_cli 工具

```bash
git clone https://gerrit.o-ran-sc.org/r/ric-plt/appmgr
cd appmgr/xapp_orchestrater/dev/xapp_onboarder
apt install -y python3-pip
pip3 uninstall xapp_onboarder
pip3 install ./

export CHART_REPO_URL=http://0.0.0.0:8090
dms_cli health   # check dms health -> status: OK

docker run --rm -u 0 -it -d -p 8090:8080 \
  -e DEBUG=1 -e STORAGE=local -e STORAGE_LOCAL_ROOTDIR=/charts \
  -v $(pwd)/charts:/charts chartmuseum/chartmuseum:latest
```

### 4.2 安裝 AD xApp

```bash
git clone --branch f-release https://gerrit.o-ran-sc.org/r/ric-app/ad
```

- 修改 `database.py` 與 `insert.py` 檔案內的 host IP(influxdb service IP)
- 在 `ad/Dockerfile` 加入:
  ```dockerfile
  RUN pip install --force-reinstall redis==3.0.1
  ```
- 從 `git clone https://github.com/ToriRobert/ad.git -b master` 取得 `schema.json`,並儲存至 `ad/xapp-descriptor` 路徑(**這一步不能遺漏**)

```bash
cd ad
docker build -t nexus3.o-ran-sc.org:10002/o-ran-sc/ric-app-ad:0.0.2 .

cd xapp-descriptor
export CHART_REPO_URL=http://0.0.0.0:8090
dms_cli onboard config.json schema.json   # 上架 xApp -> {"status": "Created"}
dms_cli get_charts_list                    # 檢查 xApp 是否成功上架
dms_cli install ad 0.0.2 ricxapp           # 安裝 xApp
```

### 4.3 安裝 TS (Traffic Steering) xApp

```bash
git clone --branch f-release https://gerrit.o-ran-sc.org/r/ric-app/ts
```

- 進入 `assets/bootstrap.rt`,將 `qpdrive` 修改為 `qp`

```bash
cd ts
docker build -t nexus3.o-ran-sc.org:10002/o-ran-sc/ric-app-ts:1.2.4 .

cd xapp-descriptor
export CHART_REPO_URL=http://0.0.0.0:8090
dms_cli onboard config-file.json schema.json
dms_cli get_charts_list
dms_cli install trafficxapp 1.2.4 ricxapp
```

### 4.4 安裝 QP (QoE Predictor) xApp

```bash
git clone --branch f-release https://gerrit.o-ran-sc.org/r/ric-app/qp
```

- 與 AD xApp 相同,修改 `database.py`、`insert.py` 的 host IP(influxdb service IP)
- 同樣在 Dockerfile 加入 `RUN pip install --force-reinstall redis==3.0.1`
- 在 `qp/setup.py` 指定套件版本:
  ```python
  install_requires=["ricxappframe>=1.1.1,<2.0.0", "joblib>=0.3.2", "statsmodels==0.13.5",
                     "mdclogpy<1.1.1", "influxdb", "pandas==1.3.5"]
  ```
- 從 `git clone https://github.com/ToriRobert/qp.git -b e-release` 取得 `schema.json`,並儲存至 `qp/xapp-descriptor` 路徑

```bash
cd qp
docker build -t nexus3.o-ran-sc.org:10002/o-ran-sc/ric-app-qp:0.0.4 .

cd xapp-descriptor
export CHART_REPO_URL=http://0.0.0.0:8090
dms_cli onboard config.json schema.json
dms_cli get_charts_list
dms_cli install qp 0.0.4 ricxapp
```

### 4.5 驗證與檢查 Log

```bash
kubectl get pods -n ricxapp                        # 檢查 xApp 狀態與 pod name
kubectl logs -f <xApp pod name> -n ricxapp
```

驗證流程(AD → TS → QP 依序觸發):

1. **AD xApp**:成功將異常 UE 的資訊傳送給 TS xApp。
2. **TS xApp**:收到 AD xApp 傳回的異常 UE 資訊,向 QP xApp 要求對該 UE 進行 QoE 預測。
3. **QP xApp**:接收到來自 TS xApp 的異常 UE 資訊,成功將 QoE 預測結果回傳給 TS xApp。
4. **TS xApp**:收到 QP xApp 傳回的 UE QoE 預測資訊,進行 UE handoff 決策,並將 handoff 要求訊息傳送至指定位置。

---

## 5. 版本資訊

| 元件 | 版本 |
|---|---|
| OS | Ubuntu 20.04 |
| Kubernetes | v1.19.11 |
| Helm | v3.5.4 |
| Docker | 20.10.21 |
| Tiller | v2.12.3 |
| Near-RT RIC | f-release |
| AD xApp | 0.0.2 |
| TS xApp | 1.2.4 |
| QP xApp | 0.0.4 |
| flannel | 0.18.1(對應舊版 `kube-system` namespace 的 manifest)|
| redis (xApp 相依) | 3.0.1 |
| statsmodels (qp) | 0.13.5 |
| pandas (qp) | 1.3.5 |

---

## 6. 參考資料

1. https://docs.o-ran-sc.org/projects/o-ran-sc-ric-plt-ric-dep/en/latest/installation-guides.html
2. https://wiki.o-ran-sc.org/display/SIM/Near-RT+RIC+Deployment
3. https://wiki.o-ran-sc.org/pages/viewpage.action?pageId=82706560
4. https://blog.51cto.com/gcherforever/4812069
5. https://docs.o-ran-sc.org/projects/o-ran-sc-ric-plt-ric-dep/en/latest/installation-guides.html#installing-near-realtime-ric-in-ric-cluster
6. https://zenn.dev/sandship/articles/a82add3764467a
7. https://www.cxyzjd.com/article/jeffyko/107975572
8. M. Polese, L. Bonati, S. D'Oro, S. Basagni and T. Melodia, "Understanding O-RAN: Architecture, Interfaces, Algorithms, Security, and Research Challenges," IEEE Communications Surveys & Tutorials, vol. 25, no. 2, pp. 1376-1411, 2023.
9. O-RAN Working Group 2, "O-RAN AI/ML workflow description and requirements 1.03," O-RAN.WG2.AIML-v01.03, Jul. 2021.
10. https://hackmd.io/@Min-xiang/SJD9Xh5c9
11. https://wiki.o-ran-sc.org/display/GS/Near+Realtime+RIC+Installation
12. https://hackmd.io/@mwnl-pre-6g-team/SJPEE-6a9
13. https://hackmd.io/@Min-xiang/HJZF3-xgt#75-Check-the-xApp-statuses-of-AD-Use-Case
14. https://wiki.o-ran-sc.org/display/GS/SMO+Installation
15. https://wiki.o-ran-sc.org/display/IAT/Traffic+Steering+Flows
16. https://wiki.o-ran-sc.org/display/RICP/Anomaly+Detection+Use+Case
17. https://wiki.o-ran-sc.org/display/IAT/AD+xApp+Flows
18. https://www.o-ran.org/
