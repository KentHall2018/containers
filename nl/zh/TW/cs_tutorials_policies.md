---

copyright:
  years: 2014, 2018
lastupdated: "2018-08-06"

---

{:new_window: target="_blank"}
{:shortdesc: .shortdesc}
{:screen: .screen}
{:pre: .pre}
{:table: .aria-labeledby="caption"}
{:codeblock: .codeblock}
{:tip: .tip}
{:download: .download}



# 指導教學：使用 Calico 網路原則來封鎖資料流量
{: #policy_tutorial}

依預設，Kubernetes NodePort、LoadBalancer 及 Ingress 服務可讓您的應用程式能夠在所有公用及專用叢集網路介面上使用。`allow-node-port-dnat` 預設 Calico 原則允許將送入的資料流量從 NodePort、LoadBalancer 和 Ingress 服務送至那些服務公開的應用程式 Pod。Kubernetes 會使用目的地網址轉譯 (DNAT) 將服務要求轉遞至正確的 Pod。

不過，基於安全理由，您可能需要只容許從特定來源 IP 位址到網路服務的資料流量。您可以使用 [Calico Pre-DNAT 原則 ![外部鏈結圖示](../icons/launch-glyph.svg "外部鏈結圖示")](https://docs.projectcalico.org/v3.1/getting-started/bare-metal/policy/pre-dnat)，將來自或送至特定 IP 位址的資料流量列入白名單或黑名單。Pre-DNAT 原則可防止指定的資料流量到達您的應用程式，因為在 Kubernetes 使用一般 DNAT 將資料流量轉遞至 Pod 之前，已套用這些原則。建立 Calico Pre-DNAT 原則時，您可以選擇是否要將來源 IP 位址列入白名單或黑名單。對於大部分情境，白名單提供最安全的配置，因為除了來自已知允許之來源 IP 位址的資料流量外，所有資料流量都會遭到封鎖。黑名單通常只用於如下的情境：防止來自小型 IP 位址集的攻擊。

在此情境中，您扮演公關公司網路管理者的角色，並注意到一些不尋常的資料流量正在攻擊您的應用程式。本指導教學中的課程將逐步引導您建立範例 Web 伺服器應用程式、使用 LoadBalancer 服務公開應用程式，以及同時使用白名單及黑名單 Calico 原則，來保護應用程式避開不需要且不尋常的資料流量。

## 目標

- 學習藉由建立高階 Pre-DNAT 原則，以封鎖送入所有 NodePort 的所有資料流量。
- 學習藉由建立低階 Pre-DNAT 原則，以容許列入白名單的來源 IP 位址可存取 LoadBalancer 公用 IP 及埠。低階原則會置換高階原則。
- 學習藉由建立低階 Pre-DNAT 原則，以封鎖列入黑名單的來源 IP 位址，讓它們無法存取 LoadBalancer 公用 IP 及埠。

## 所需時間
1 小時

## 適用對象
本指導教學的適用對象是想要管理送至應用程式之網路資料流量的軟體開發人員及網路管理者。

## 必要條件

- [建立 1.10 版叢集](cs_clusters.html#clusters_ui)或[將現有叢集更新為 1.10 版](cs_versions.html#cs_v110)。若要在本指導教學中使用 3.1.1 Calico CLI 及 Calico 第 3 版原則語法，則需要有 Kubernetes 1.10 版或更新版本的叢集。
- [將 CLI 的目標設為叢集](cs_cli_install.html#cs_cli_configure)。
- [安裝並配置 Calico CLI](cs_network_policy.html#1.10_install)。

<br />


## 第 1 課：使用 LoadBalancer 來部署應用程式並將它公開
{: #lesson1}

第一課告訴您如何從多個 IP 位址及埠公開您的應用程式，以及公用資料流量是從哪裡進入您的叢集。
{: shortdesc}

開始部署要在整個指導教學中使用的範例 Web 伺服器應用程式。`echoserver` Web 伺服器會顯示有關建立用戶端與叢集之連線的資料，並讓您測試對公關公司叢集的存取。然後，藉由建立 LoadBalancer 服務來公開應用程式。LoadBalancer 服務可讓您的應用程式透過 LoadBalancer 服務 IP 位址及工作者節點的 NodePort 提供使用。

下圖顯示在第 1 課結束時如何藉由公用 NodePort 及公用 LoadBalancer，在網際網路公開 Web 伺服器應用程式：

<img src="images/cs_tutorial_policies_Lesson1.png" width="450" alt="在第 1 課結束時，會藉由公用 NodePort 及公用 LoadBalancer，在網際網路公開 Web 伺服器應用程式。" style="width:450px; border-style: none"/>

1. 建立要在整個指導教學中使用的測試名稱空間，稱為 `pr-firm`。
    ```
    kubectl create ns pr-firm
    ```
    {: pre}

2. 部署範例 Web 伺服器應用程式。與 Web 伺服器應用程式建立連線時，應用程式會以其在連線中收到的 HTTP 標頭回應。
    ```
    kubectl run webserver -n pr-firm --image=k8s.gcr.io/echoserver:1.10 --replicas=3
    ```
    {: pre}

3. 驗證 Web 伺服器應用程式 Pod 的 **STATUS** 為 `Running`。
    ```
    kubectl get pods -n pr-firm -o wide
    ```
    {: pre}

    輸出範例：
    ```
    NAME                         READY     STATUS    RESTARTS   AGE       IP               NODE
    webserver-855556f688-6dbsn   1/1       Running   0          1m        172.30.xxx.xxx   10.176.48.78
    webserver-855556f688-76rkp   1/1       Running   0          1m        172.30.xxx.xxx   10.176.48.78
    webserver-855556f688-xd849   1/1       Running   0          1m        172.30.xxx.xxx   10.176.48.78
    ```
    {: screen}

4. 若要在公用網際網路公開應用程式，請在文字編輯器中建立一個稱為 `webserver.yaml` 的 LoadBalancer 服務配置檔。
    ```
    apiVersion: v1
    kind: Service
    metadata:
      labels:
        run: webserver
      name: webserver-lb
      namespace: pr-firm
    spec:
      externalTrafficPolicy: Cluster
      ports:
      - name: webserver-port
        port: 80
        protocol: TCP
        targetPort: 8080
      selector:
        run: webserver
      type: LoadBalancer
    ```
    {: codeblock}

5. 部署 LoadBalancer。
    ```
    kubectl apply -f filepath/webserver.yaml
    ```
    {: pre}

6. 驗證您可以從電腦公開存取 LoadBalancer 所公開的應用程式。

    1. 取得 LoadBalancer 的公用 **EXTERNAL-IP** 位址。
        ```
        kubectl get svc -n pr-firm -o wide
        ```
        {: pre}

        輸出範例：
        ```
        NAME           CLUSTER-IP       EXTERNAL-IP        PORT(S)        AGE       SELECTOR
        webserver-lb   172.21.xxx.xxx   169.xx.xxx.xxx     80:31024/TCP   2m        run=webserver
        ```
        {: screen}

    2. 建立提要文字檔，並將 LoadBalancer IP 複製到文字檔。提要將協助您在稍後的課程中更快速地使用這些值。

    3. 驗證您可以公開存取 LoadBalancer 的外部 IP。
        ```
        curl --connect-timeout 10 <loadbalancer_IP>:80
        ```
        {: pre}

        下列輸出範例確認 LoadBalancer 會在 `169.1.1.1` 公用 LoadBalancer IP 位址上公開您的應用程式。`webserver-855556f688-76rkp` 應用程式 Pod 收到 Curl 要求：
        ```
        Hostname: webserver-855556f688-76rkp
        Pod Information:
            -no pod information available-
        Server values:
            server_version=nginx: 1.13.3 - lua: 10008
        Request Information:
            client_address=1.1.1.1
            method=GET
            real path=/
            query=
            request_version=1.1
            request_scheme=http
            request_uri=http://169.1.1.1:8080/
        Request Headers:
            accept=*/*
            host=169.1.1.1
            user-agent=curl/7.54.0
        Request Body:
            -no body in request-
        ```
        {: screen}

6. 驗證您可以從電腦公開存取 NodePort 所公開的應用程式。LoadBalancer 服務可讓您的應用程式透過 LoadBalancer 服務 IP 位址及工作者節點的 NodePort 提供使用。

    1. 取得 LoadBalancer 已指派給工作者節點的 NodePort。NodePort 位於 30000 到 32767 範圍內。
        ```
        kubectl get svc -n pr-firm -o wide
        ```
        {: pre}

        在下列輸出範例中，NodePort 為 `31024`：
        ```
        NAME           CLUSTER-IP       EXTERNAL-IP        PORT(S)        AGE       SELECTOR
        webserver-lb   172.21.xxx.xxx   169.xx.xxx.xxx     80:31024/TCP   2m        run=webserver
        ```
        {: screen}  

    2. 取得工作者節點的**公用 IP** 位址。
        ```
        ibmcloud ks workers <cluster_name>
        ```
        {: pre}

        輸出範例：
        ```
        ID                                                 Public IP        Private IP     Machine Type        State    Status   Zone    Version   
        kube-dal10-cr18e61e63c6e94b658596ca93d087eed9-w1   169.xx.xxx.xxx   10.176.48.67   u2c.2x4.encrypted   normal   Ready    dal10   1.10.5_1513*   
        kube-dal10-cr18e61e63c6e94b658596ca93d087eed9-w2   169.xx.xxx.xxx   10.176.48.79   u2c.2x4.encrypted   normal   Ready    dal10   1.10.5_1513*   
        kube-dal10-cr18e61e63c6e94b658596ca93d087eed9-w3   169.xx.xxx.xxx   10.176.48.78   u2c.2x4.encrypted   normal   Ready    dal10   1.10.5_1513*   
        ```
        {: screen}

    3. 將工作者節點及 NodePort 的公用 IP 複製到您的文字提要，以在稍後課程中使用。

    4. 驗證您可以透過 NodePort 存取工作者節點的公用 IP 位址。
        ```
        curl  --connect-timeout 10 <worker_IP>:<NodePort>
        ```
        {: pre}

        下列輸出範例確認，是透過工作者節點的專用 IP 位址 `10.1.1.1` 及 `31024` NodePort，對您的應用程式提出要求。`webserver-855556f688-xd849` 應用程式 Pod 收到 Curl 要求：
        ```
        Hostname: webserver-855556f688-xd849
        Pod Information:
            -no pod information available-
        Server values:
            server_version=nginx: 1.13.3 - lua: 10008
        Request Information:
            client_address=1.1.1.1
            method=GET
            real path=/
            query=
            request_version=1.1
            request_scheme=http
            request_uri=http://10.1.1.1:8080/
        Request Headers:
            accept=*/*
            host=10.1.1.1:31024
            user-agent=curl/7.60.0
        Request Body:
            -no body in request-
        ```
        {: screen}

此時，會從多個 IP 位址及埠公開您的應用程式。其中大部分 IP 都在叢集的內部，而且只能透過專用網路存取。只有公用 NodePort 及公用 LoadBalancer 埠才會在公用網際網路公開。

接下來，您可以開始建立並套用 Calico 原則來封鎖公用資料流量。

## 第 2 課：封鎖送入所有 NodePort 的所有資料流量
{: #lesson2}

若要保護公關公司的叢集，您必須封鎖對公開您應用程式之 LoadBalancer 服務及 NodePort 兩者的公開存取。首先，會封鎖對 NodePort 的存取。下圖顯示在第 2 課結束時如何允許資料流量送至 LoadBalancer，但不送至 NodePort：

<img src="images/cs_tutorial_policies_Lesson2.png" width="450" alt="在第 2 課結束時，僅透過公用 LoadBalancer，在網際網路公開 Web 伺服器應用程式。" style="width:450px; border-style: none"/>

1. 在文字編輯器中，建立一個稱為 `deny-nodeports.yaml` 的高階 Pre-DNAT 原則，來拒絕將 TCP 及 UDP 資料流量從任何來源 IP 送入所有 NodePort。
    ```
    apiVersion: projectcalico.org/v3
    kind: GlobalNetworkPolicy
    metadata:
      name: deny-nodeports
    spec:
      applyOnForward: true
      ingress:
      - action: Deny
            destination:
              ports:
          - 30000:32767
            protocol: TCP
            source: {}
      - action: Deny
            destination:
              ports:
          - 30000:32767
        protocol: UDP
        source: {}
      preDNAT: true
      selector: ibm.role=='worker_public'
      order: 1100
      types:
      - Ingress
        ```
    {: codeblock}

2. 套用原則。
    - Linux：

      ```
      calicoctl apply -f filepath/deny-nodeports.yaml
      ```
      {: pre}

    - Windows 及 OS X：

      ```
      calicoctl apply -f filepath/deny-nodeports.yaml --config=filepath/calicoctl.cfg
      ```
      {: pre}
輸出範例：
        ```
  Successfully applied 1 'GlobalNetworkPolicy' resource(s)
  ```
  {: screen}

3. 使用來自提要的值，驗證您無法公開存取工作者節點公用 IP 位址及 NodePort。
    ```
    curl  --connect-timeout 10 <worker_IP>:<NodePort>
    ```
    {: pre}

    連線逾時，因為您建立的 Calico 原則封鎖資料流量送入 NodePort。
    ```
    curl: (28) Connection timed out after 10016 milliseconds
    ```
    {: screen}

4. 將您在前一課中建立之 LoadBalancer 的 externalTrafficPolicy 從 `Cluster` 變更為 `Local`。`Local` 確保在下一步您對 LoadBalancer 的外部 IP 進行 Crul 處理時，會保留系統的來源 IP。
    ```
    kubectl patch svc -n pr-firm webserver -p '{"spec":{"externalTrafficPolicy":"Local"}}'
    ```
    {: pre}

5. 使用來自提要的值，驗證您仍然可以公開存取 LoadBalancer 外部 IP 位址。
    ```
    curl --connect-timeout 10 <loadbalancer_IP>:80
    ```
    {: pre}

    輸出範例：
    ```
    Hostname: webserver-855556f688-76rkp
    Pod Information:
        -no pod information available-
    Server values:
        server_version=nginx: 1.13.3 - lua: 10008
    Request Information:
        client_address=1.1.1.1
        method=GET
        real path=/
        query=
        request_version=1.1
        request_scheme=http
        request_uri=http://<loadbalancer_IP>:8080/
    Request Headers:
        accept=*/*
        host=<loadbalancer_IP>
        user-agent=curl/7.54.0
    Request Body:
        -no body in request-
    ```
    {: screen}
    在輸出的 `Request Information` 區段中，注意來源 IP 位址為：例如，`client_address=1.1.1.1`。來源 IP 位址是您要用來執行 Curl 之系統的公用 IP。否則，如果您是透過 Proxy 或 VPN 連接至網際網路，則 Proxy 或 VPN 可能會遮蔽系統的實際 IP 位址。在任一情況中，LoadBalancer 會將您系統的來源 IP 位址看成用戶端 IP 位址。

6. 將您系統的來源 IP 位址（前一個步驟輸出中的 `client_address=1.1.1.1`）複製到您的提要，以在稍後課桯中使用。

太棒了！此時，只會從公用 LoadBalancer 埠，在公用網際網路公開您的應用程式。送入公用 NodePort 的資料流量會遭到封鎖。您已局部鎖定叢集，不准不需要的資料流量送入其中。

接下檢，您可以建立並套用 Calico 原則，將來自特定來源 IP 的資料流量列入白名單。

## 第 3 課：容許將資料流量從列入白名單的 IP 送入 LoadBalancer
{: #lesson3}

您現在決定完全鎖定，不准資料流量送入公關公司的叢集，並將您自己電腦的 IP 位址列入白名單來測試存取。
{: shortdesc}

首先，除了 NodePorts 之外，您必須封鎖所有資料流量送入公開應用程式的 LoadBalancer。然後，您可以建立一個原則，將系統的 IP 位址列入白名單。在第 3 課結束時，將封鎖所有送至公用 NodePort 及 LoadBalancer 的資料流量，而且僅容許來自列入白名單之系統 IP 的資料流量：
<img src="images/cs_tutorial_policies_L3.png" width="600" alt="Web 伺服器應用程式僅由公用 LoadBalancer 在您的系統 IP 公開。" style="width:600px; border-style: none"/>
1. 在文字編輯器中，建立一個稱為 `deny-lb-port-80.yaml` 的高階 Pre-DNAT 原則，來拒絕將 TCP 及 UDP 資料流量從任何來源 IP 送入 LoadBalancer IP 位址及埠。將 `<loadbalancer_IP>` 取代為來自提要的 LoadBalancer 公用 IP 位址。
    ```
    apiVersion: projectcalico.org/v3
    kind: GlobalNetworkPolicy
    metadata:
      name: deny-lb-port-80
    spec:
      applyOnForward: true
      ingress:
      - action: Deny
        destination:
          nets:
          - <loadbalancer_IP>/32
          ports:
          - 80
        protocol: TCP
        source: {}
      - action: Deny
        destination:
          nets:
          - <loadbalancer_IP>/32
          ports:
          - 80
        protocol: UDP
        source: {}
      preDNAT: true
      selector: ibm.role=='worker_public'
      order: 1100
      types:
      - Ingress
        ```
    {: codeblock}

2. 套用原則。
    - Linux：

      ```
      calicoctl apply -f filepath/deny-lb-port-80.yaml
      ```
      {: pre}

    - Windows 及 OS X：

      ```
      calicoctl apply -f filepath/deny-lb-port-80.yaml --config=filepath/calicoctl.cfg
      ```
      {: pre}

3. 使用來自提要的值，驗證您現在無法存取公用 LoadBalancer IP 位址。連線逾時，因為您建立的 Calico 原則封鎖資料流量送入 LoadBalancer。
    ```
    curl --connect-timeout 10 <loadbalancer_IP>:80
    ```
    {: pre}

4. 在文字編輯器中，建立一個稱為 `whitelist.yaml` 的低階 Pre-DNAT 原則，來容許資料流量從系統的 IP 送至 LoadBalancer IP 位址及埠。使用來自提要的值，將 `<loadbalancer_IP>` 取代為 LoadBalancer 的公用 IP 位址，並將 `<client_address>` 取代為系統來源 IP 的公用 IP 位址。
    ```
    apiVersion: projectcalico.org/v3
    kind: GlobalNetworkPolicy
    metadata:
      name: whitelist
    spec:
      applyOnForward: true
      ingress:
      - action: Allow
        destination:
          nets:
          - <loadbalancer_IP>/32
          ports:
          - 80
        protocol: TCP
        source:
          nets:
          - <client_address>/32
      preDNAT: true
      selector: ibm.role=='worker_public'
      order: 500
      types:
      - Ingress
        ```
    {: codeblock}

5. 套用原則。
    - Linux：

      ```
      calicoctl apply -f filepath/whitelist.yaml
      ```
      {: pre}

    - Windows 及 OS X：

      ```
      calicoctl apply -f filepath/whitelist.yaml --config=filepath/calicoctl.cfg
      ```
      {: pre}
  您系統的 IP 位址現在已列入白名單。

6. 使用來自提要的值，驗證您現在可以存取公用 LoadBalancer IP 位址。
    ```
    curl --connect-timeout 10 <loadbalancer_IP>:80
    ```
    {: pre}

7. 如果您可以存取另一個具有不同 IP 位址的系統，請嘗試從該系統存取 LoadBalancer。
    ```
    curl --connect-timeout 10 <loadbalancer_IP>:80
    ```
    {: pre}
    連線逾時，因為該系統的 IP 位址未列入白名單。

此時，所有送至公用 NodePort 及 LoadBalancer 的資料流量都會遭到封鎖。只容許來自已列入白名單之系統 IP 的資料流量。

## 第 4 課：拒絕資料流量從列入黑名單的 IP 送入 LoadBalancer
{: #lesson4}

在前一課中，您已封鎖所有資料流量，並只將少數 IP 列入白名單。當您想要限制只存取少數受管制的來源 IP 位址時，該情境適用於測試目的。不過，公關公司具有需要大眾可以廣泛使用的應用程式。您需要確定允許所有資料流量，但您從少數 IP 位址看到的不尋常資料流量除外。黑名單有助於這類情境，因為它可以協助您防止來自小型 IP 位址集的攻擊。

在本課程中，您將藉由封鎖來自您自己系統之來源 IP 位址的資料流量，來測試黑名單。在第 4 課結束時，將封鎖所有送至公用 NodePort 的資料流量，而且將容許所有送至公用 LoadBalancer 的資料流量。只會封鎖從列入黑名單之系統 IP 送至 LoadBalancer 的資料流量：
<img src="images/cs_tutorial_policies_L4.png" width="600" alt="Web 伺服器應用程式是由公用 LoadBalancer 在網際網路公開。只封鎖來自您系統 IP 的資料流量。" style="width:600px; border-style: none"/>
1. 清除您在前一課中建立的白名單原則。
    ```
    calicoctl delete GlobalNetworkPolicy deny-lb-port-80
    ```
    {: pre}
    ```
    calicoctl delete GlobalNetworkPolicy whitelist
    ```
    {: pre}
    現有，再次允許將所有 TCP 及 UDP 資料流量從任何來源 IP 送入 LoadBalancer IP 位址及埠。

2. 若要拒絕將所有 TCP 及 UDP 資料流量從您系統的來源 IP 位址送入 LoadBalancer IP 位址及埠，請在文字編輯器中建立一個稱為 `deny-lb-port-80.yaml` 的低階 Pre-DNAT 原則。使用來自提要的值，將 `<loadbalancer_IP>` 取代為 LoadBalancer 的公用 IP 位址，並將 `<client_address>` 取代為系統來源 IP 的公用 IP 位址。
    ```
    apiVersion: projectcalico.org/v3
    kind: GlobalNetworkPolicy
    metadata:
      name: blacklist
    spec:
      applyOnForward: true
      ingress:
      - action: Deny
        destination:
          nets:
          - <loadbalancer_IP>/32
          ports:
          - 80
        protocol: TCP
        source:
          nets:
          - <client_address>/32
      - action: Deny
        destination:
          nets:
          - <loadbalancer_IP>/32
          ports:
          - 80
        protocol: UDP
        source:
          nets:
          - <client_address>/32
      preDNAT: true
      selector: ibm.role=='worker_public'
      order: 500
      types:
      - Ingress
        ```
    {: codeblock}

3. 套用原則。
    - Linux：

      ```
      calicoctl apply -f filepath/blacklist.yaml
      ```
      {: pre}

    - Windows 及 OS X：

      ```
      calicoctl apply -f filepath/blacklist.yaml --config=filepath/calicoctl.cfg
      ```
      {: pre}
  您系統的 IP 位址現在已列入黑名單。

4. 使用來自提要的值，從您的系統驗證您無法存取 LoadBalancer IP，因為您系統的 IP 已列入黑名單。
    ```
    curl --connect-timeout 10 <loadbalancer_IP>:80
    ```
    {: pre}
    此時，會封鎖所有送至公用 NodePort 的資料流量，而且容許所有送至公用 LoadBalancer 的資料流量。只會封鎖從列入黑名單之系統 IP 送至 LoadBalancer 的資料流量。

5. 若要清除此黑名單原則，請執行下列指令：
    ```
    calicoctl delete GlobalNetworkPolicy blacklist
    ```
    {: pre}

做得好！您已順利控制進入應用程式的資料流量，方法為使用 Calico Pre-DNAT 原則，將來源 IP 列入白名單及黑名單。

## 下一步為何？
{: #whats_next}

* 深入閱讀[使用網路原則控制資料流量](cs_network_policy.html)。
* 如需其他控制資料流量進出叢集的範例 Calico 網路原則，請參閱[主要原則展示 ![外部鏈結圖示](../icons/launch-glyph.svg "外部鏈結圖示")](https://docs.projectcalico.org/v3.1/getting-started/kubernetes/tutorials/stars-policy/) 及[進階網路原則 ![外部鏈結圖示](../icons/launch-glyph.svg "外部鏈結圖示")](https://docs.projectcalico.org/v3.1/getting-started/kubernetes/tutorials/advanced-policy)。
