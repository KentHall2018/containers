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


# Contrôle du trafic à l'aide de règles réseau
{: #network_policies}

Chaque cluster Kubernetes est configuré avec un plug-in réseau nommé Calico. Des règles réseau par défaut sont mises en place pour sécuriser l'interface réseau publique de chaque noeud worker dans {{site.data.keyword.containerlong}}.
{: shortdesc}

Si vous avez des exigences particulières en matière de sécurité ou si vous disposez d'un cluster à zones multiples avec la fonction Spanning VLAN activée, vous pouvez utiliser Calico et Kubernetes afin de créer des règles réseau pour un cluster. Avec les règles réseau Kubernetes, vous pouvez spécifier le trafic réseau que vous désirez autoriser ou bloquer vers et depuis un pod au sein d'un cluster. Pour définir des règles réseau plus avancées, par exemple le blocage de trafic entrant (ingress) vers les services d'équilibreur de charge (LoadBalancer), utilisez les règles réseau Calico.

<ul>
  <li>
  [Règles réseau Kubernetes ![Icône de lien externe](../icons/launch-glyph.svg "Icône de lien externe")](https://kubernetes.io/docs/concepts/services-networking/network-policies/) : ces règles indiquent comment les pods communiquent avec d'autres pods et avec des noeuds finaux externes. A partir de Kubernetes version 1.8, le trafic réseau entrant et sortant peut être autorisé ou bloqué selon le protocole, le port et les adresses IP source et de destination. Le trafic peut également être filtré en fonction des libellés de pod et d'espace de nom. Les règles réseau Kubernetes sont appliquées à l'aide de commandes `kubectl` ou d'API Kubernetes. Lorsque ces règles sont appliquées, elles sont automatiquement converties en règles réseau Calico et Calico les met en oeuvre.
  </li>
  <li>
  Règles réseau Calico pour les clusters Kubernetes version [1.10 et ultérieure ![Icône de lien externe](../icons/launch-glyph.svg "Icône de lien externe")](https://docs.projectcalico.org/v3.1/getting-started/kubernetes/tutorials/advanced-policy) ou [1.9 et antérieure ![Icône de lien externe](../icons/launch-glyph.svg "Icône de lien externe")](https://docs.projectcalico.org/v2.6/getting-started/kubernetes/tutorials/advanced-policy) : ces règles sont un sur-ensemble de règles réseau Kubernetes et sont appliquées à l'aide de commandes `calicoctl`. Les règles Calico ajoutent les fonctions suivantes.
    <ul>
    <li>Autorisation ou blocage du trafic réseau sur des interfaces réseau spécifiques, sans tenir compte de la source de pod Kubernetes, de l'adresse IP de destination ou du routage CIDR.</li>
    <li>Autorisation ou blocage du trafic réseau pour les pods entre les espaces de nom.</li>
    <li>[Blocage de trafic entrant (ingress) vers les services Kubernetes LoadBalancer ou NodePort](#block_ingress).</li>
    </ul>
  </li>
  </ul>

Calico met en vigueur ces règles, y compris les éventuelles règles réseau Kubernetes converties automatiquement en règles Calico, en configurant des règles Linux iptables sur les noeuds worker Kubernetes. Les règles iptables font office de pare-feu pour le noeud worker en définissant les caractéristiques que le trafic réseau doit respecter pour être acheminé vers la ressource ciblée.

Pour avoir recours aux services Ingress et LoadBalancer, utilisez des règles Calico et Kubernetes pour gérer le trafic réseau en provenance et à destination de votre cluster. N'utilisez pas les [groupes de sécurité](/docs/infrastructure/security-groups/sg_overview.html#about-security-groups) de l'infrastructure IBM Cloud (SoftLayer). Ces groupes de sécurité sont appliqués à l'interface réseau d'un serveur virtuel unique pour filtrer le trafic au niveau de l'hyperviseur. Toutefois, les groupes de sécurité ne prennent pas en charge le protocole VRRP qui est utilisé par {{site.data.keyword.containershort_notm}} pour gérer l'adresse IP du service LoadBalancer. Si le protocole VRRP n'est pas présent pour gérer l'adresse IP du service LoadBalancer, les services Ingress et LoadBalancer ne fonctionneront pas correctement.
{: tip}

<br />


## Règles réseau Calico et Kubernetes par défaut
{: #default_policy}

Lorsqu'un cluster avec un réseau local virtuel (VLAN) public est créé, une ressource HostEndpoint avec le libellé `ibm.role: worker_public` est générée automatiquement pour chaque noeud worker et l'interface réseau publique associée. Pour protéger l'interface réseau publique d'un noeud worker, des règles Calico par défaut sont appliquées à chaque noeud final d'hôte avec le libellé `ibm.role: worker_public`.
{:shortdesc}

Ces règles Calico par défaut autorisent tout le trafic réseau sortant et autorisent le trafic réseau entrant sur des composants de cluster spécifiques, par exemple les services Kubernetes NodePort, LoadBalancer et Ingress. Tout autre trafic réseau entrant provenant d'Internet vers vos noeuds worker qui n'est pas spécifié dans les règles par défaut est bloqué. Les règles par défaut n'affectent pas le trafic entre les pods.

Consultez les règles réseau Calico par défaut suivantes qui sont automatiquement appliquées à votre cluster.

**Important :** prenez soin de ne pas supprimer de règle appliquée à un noeud final d'hôte à moins de comprendre parfaitement la règle. Assurez-vous de ne pas avoir besoin du trafic autorisé par la règle en question.

 <table summary="La première ligne du tableau est répartie sur deux colonnes. La lecture des autres lignes s'effectue de gauche à droite, avec la zone du serveur dans la première colonne et les adresses IP correspondantes dans la deuxième colonne.">
 <caption>Règles Calico par défaut pour chaque cluster</caption>
  <thead>
  <th colspan=2><img src="images/idea.png" alt="Icône Idée"/> Règles Calico par défaut pour chaque cluster</th>
  </thead>
  <tbody>
    <tr>
      <td><code>allow-all-outbound</code></td>
      <td>Autorise tout le trafic sortant.</td>
    </tr>
    <tr>
      <td><code>allow-bigfix-port</code></td>
      <td>Autorise le trafic entrant sur le port 52311 vers l'application BigFix à accepter les mises à jour de noeud worker nécessaires.</td>
    </tr>
    <tr>
      <td><code>allow-icmp</code></td>
      <td>Autorise tous les paquets ICMP entrants (pings).</td>
     </tr>
    <tr>
      <td><code>allow-node-port-dnat</code></td>
      <td>Autorise le trafic entrant des services NodePort, LoadBalancer et Ingress vers les pods exposés par ces services. <strong>Remarque</strong> : vous n'avez pas besoin d'indiquer les ports exposés car Kubernetes utilise la conversion d'adresse réseau de destination (DNAT) pour transférer les demandes de service aux pods appropriés. Ce réacheminement intervient avant l'application des règles de noeud final d'hôte dans des iptables.</td>
   </tr>
   <tr>
      <td><code>allow-sys-mgmt</code></td>
      <td>Autorise les connexions entrantes pour des systèmes d'infrastructure IBM Cloud (SoftLayer) spécifiques utilisés pour gérer les noeuds worker.</td>
   </tr>
   <tr>
    <td><code>allow-vrrp</code></td>
    <td>Autorise les paquets VRRP utilisés pour le suivi et le déplacement d'adresses IP virtuelles entre les noeuds worker.</td>
   </tr>
  </tbody>
</table>

Dans les clusters Kubernetes version 1.10 et les clusters plus récents, une règle Kubernetes par défaut limitant l'accès au tableau de bord Kubernetes est également créée. Les règles Kubernetes ne s'appliquent pas au noeud final d'hôte, mais au pod `kube-dashboard` à la place. Cette règle s'applique aux clusters connectés uniquement à un VLAN privé et aux clusters connectés à un VLAN public et privé.

<table>
<caption>Règles Kubernetes par défaut pour chaque cluster</caption>
<thead>
<th colspan=2><img src="images/idea.png" alt="Icône Idée"/> Règles Kubernetes par défaut pour chaque cluster</th>
</thead>
<tbody>
 <tr>
  <td><code>kubernetes-dashboard</code></td>
  <td><b>Dans Kubernetes v1.10 uniquement</b>, règle fournie dans l'espace de nom <code>kube-system</code> : blocage de l'accès de tous les pods au tableau de bord Kubernetes. Cette règle n'affecte pas l'accès au tableau de bord à partir de l'interface utilisateur {{site.data.keyword.Bluemix_notm}} ou via le proxy <code>kubectl proxy</code>. Si un pod nécessite l'accès au tableau de bord, déployez-le dans un espace de nom ayant le libellé <code>kubernetes-dashboard-policy: allow</code>.</td>
 </tr>
</tbody>
</table>

<br />


## Installation et configuration de l'interface de ligne de commande (CLI) de Calico
{: #cli_install}

Pour afficher, gérer et ajouter des règles Calico, installez et configurez l'interface CLI de Calico.
{:shortdesc}

La compatibilité entre les versions de Calico pour la configuration de l'interface CLI et les règles varient en fonction de la version Kubernetes de votre cluster. Pour installer et configurer l'interface CLI de Calico, cliquez sur l'un des liens suivants en fonction de la version de votre cluster :

* [Clusters Kubernetes version 1.10 ou ultérieure](#1.10_install)
* [Clusters Kubernetes version 1.9 ou antérieure](#1.9_install)

Avant de mettre à jour votre cluster de la version Kubernetes 1.9 ou antérieure à la version 1.10 ou ultérieure, consultez la rubrique [Préparation à la mise à jour vers Calico v3](cs_versions.html#110_calicov3).
{: tip}

### Installation et configuration de l'interface CLI de Calico version 3.1.1 pour les clusters exécutant Kubernetes version 1.10 ou ultérieure
{: #1.10_install}

Avant de commencer, [ciblez l'interface CLI de Kubernetes sur le cluster](cs_cli_install.html#cs_cli_configure). Incluez l'option `--admin` avec la commande `ibmcloud ks cluster-config`, laquelle est utilisée pour télécharger les fichiers de certificat et d'autorisations. Ce téléchargement inclut également les clés pour le rôle Superutilisateur, dont vous aurez besoin pour exécuter des commandes Calico.

  ```
  ibmcloud ks cluster-config <cluster_name> --admin
  ```
  {: pre}

Pour installer et configurer l'interface CLI de Calico 3.1.1 :

1. [Téléchargez l'interface CLI de Calico ![Icône de lien externe](../icons/launch-glyph.svg "Icône de lien externe")](https://github.com/projectcalico/calicoctl/releases/tag/v3.1.1).

    Si vous utilisez OS X, téléchargez la version `-darwin-amd64`. Si vous utilisez Windows, installez l'interface CLI de Calico dans le même répertoire que celle d'{{site.data.keyword.Bluemix_notm}}. Cette configuration vous évite diverses modifications de chemin de fichier lorsque vous exécuterez des commandes plus tard. Veillez à sauvegarder le fichier sous `calicoctl.exe`.
    {: tip}

2. Utilisateurs OS X et Linux : procédez comme suit.
    1. Déplacez le fichier exécutable vers le répertoire _/usr/local/bin_.
        - Linux :

          ```
          mv filepath/calicoctl /usr/local/bin/calicoctl
          ```
          {: pre}

        - OS X :

          ```
          mv filepath/calicoctl-darwin-amd64 /usr/local/bin/calicoctl
          ```
          {: pre}

    2. Rendez le fichier exécutable.

        ```
        chmod +x /usr/local/bin/calicoctl
        ```
        {: pre}

3. Assurez-vous que les commandes `calico` se sont correctement exécutées en vérifiant la version du client CLI de Calico.

    ```
    calicoctl version
    ```
    {: pre}

4. Si les règles réseau d'entreprise utilisent des proxys ou des pare-feux pour empêcher l'accès depuis votre système local à des noeuds finaux publics, [autorisez l'accès TCP pour les commandes Calico](cs_firewall.html#firewall).

5. Pour Linux et OS X, créez le répertoire `/etc/calico`. Pour Windows, vous pouvez utiliser n'importe quel répertoire.

  ```
  sudo mkdir -p /etc/calico/
  ```
  {: pre}

6. Créez un fichier `calicoctl.cfg`.
    - Linux et OS X :

      ```
      sudo vi /etc/calico/calicoctl.cfg
      ```
      {: pre}

    - Windows : créez le fichier à l'aide d'un éditeur de texte.

7. Entrez les informations suivantes dans le fichier <code>calicoctl.cfg</code>.

    ```
    apiVersion: projectcalico.org/v3
    kind: CalicoAPIConfig
    metadata:
    spec:
        datastoreType: etcdv3
        etcdEndpoints: <ETCD_URL>
        etcdKeyFile: <CERTS_DIR>/admin-key.pem
        etcdCertFile: <CERTS_DIR>/admin.pem
        etcdCACertFile: <CERTS_DIR>/<ca-*pem_file>
    ```
    {: codeblock}

    1. Extrayez la valeur `<ETCD_URL>`.

      - Linux et OS X :

          ```
          kubectl get cm -n kube-system calico-config -o yaml | grep "etcd_endpoints:" | awk '{ print $2 }'
          ```
          {: pre}

          Exemple de sortie :

          ```
          https://169.xx.xxx.xxx:30000
          ```
          {: screen}

      - Windows :
        <ol>
        <li>Récupérez les valeurs de configuration calico dans le fichier configmap. </br><pre class="codeblock"><code>kubectl get cm -n kube-system calico-config -o yaml</code></pre></br>
        <li>Dans la section `data`, localisez la valeur etcd_endpoints. Exemple : <code>https://169.xx.xxx.xxx:30000</code>
        </ol>

    2. Extrayez le répertoire `<CERTS_DIR>` dans lequel les certificats Kubernetes sont téléchargés.

        - Linux et OS X :

          ```
          dirname $KUBECONFIG
          ```
          {: pre}

          Exemple de sortie :

          ```
          /home/sysadmin/.bluemix/plugins/container-service/clusters/<cluster_name>-admin/
          ```
          {: screen}

        - Windows :

          ```
          ECHO %KUBECONFIG%
          ```
          {: pre}

            Exemple de sortie :

          ```
          C:/Users/<user>/.bluemix/plugins/container-service/mycluster-admin/kube-config-prod-dal10-mycluster.yml
          ```
          {: screen}

        **Remarque** : pour obtenir le chemin de répertoire, retirez le nom de fichier `kube-config-prod-<zone>-<cluster_name>.yml` à la fin de la sortie.

    3. Extrayez la valeur `ca-*pem_file`.

        - Linux et OS X :

          ```
          ls `dirname $KUBECONFIG` | grep "ca-"
          ```
          {: pre}

        - Windows :
          <ol><li>Ouvrez le répertoire que vous avez récupéré à l'étape précédente.</br><pre class="codeblock"><code>C:\Users\<user>\.bluemix\plugins\container-service\&lt;cluster_name&gt;-admin\</code></pre>
          <li> Localisez le fichier <code>ca-*pem_file</code>.</ol>

8. Sauvegardez le fichier et assurez-vous que vous êtes bien dans le répertoire dans lequel se trouve le fichier.

9. Vérifiez que la configuration Calico fonctionne correctement.

    - Linux et OS X :

      ```
      calicoctl get nodes
      ```
      {: pre}

    - Windows :

      ```
      calicoctl get nodes --config=filepath/calicoctl.cfg
      ```
      {: pre}

      Sortie :

      ```
      NAME
              kube-dal10-crc21191ee3997497ca90c8173bbdaf560-w1.cloud.ibm
              kube-dal10-crc21191ee3997497ca90c8173bbdaf560-w2.cloud.ibm
              kube-dal10-crc21191ee3997497ca90c8173bbdaf560-w3.cloud.ibm
      ```
      {: screen}


### Installation et configuration de l'interface CLI de Calico version 1.6.3 pour les clusters exécutant Kubernetes version 1.9 ou antérieure
{: #1.9_install}

Avant de commencer, [ciblez l'interface CLI de Kubernetes sur le cluster](cs_cli_install.html#cs_cli_configure). Incluez l'option `--admin` avec la commande `ibmcloud ks cluster-config`, laquelle est utilisée pour télécharger les fichiers de certificat et d'autorisations. Ce téléchargement inclut également les clés pour le rôle Superutilisateur, dont vous aurez besoin pour exécuter des commandes Calico.

  ```
  ibmcloud ks cluster-config <cluster_name> --admin
  ```
  {: pre}

Pour installer et configurer l'interface CLI de Calico 1.6.3 :

1. [Téléchargez l'interface CLI de Calico ![Icône de lien externe](../icons/launch-glyph.svg "Icône de lien externe")](https://github.com/projectcalico/calicoctl/releases/tag/v1.6.3).

    Si vous utilisez OS X, téléchargez la version `-darwin-amd64`. Si vous utilisez Windows, installez l'interface CLI de Calico dans le même répertoire que celle d'{{site.data.keyword.Bluemix_notm}}. Cette configuration vous évite diverses modifications de chemin de fichier lorsque vous exécuterez des commandes plus tard.
    {: tip}

2. Utilisateurs OS X et Linux : procédez comme suit.
    1. Déplacez le fichier exécutable vers le répertoire _/usr/local/bin_.
        - Linux :

          ```
          mv filepath/calicoctl /usr/local/bin/calicoctl
          ```
          {: pre}

        - OS X :

          ```
          mv filepath/calicoctl-darwin-amd64 /usr/local/bin/calicoctl
          ```
          {: pre}

    2. Rendez le fichier exécutable.

        ```
        chmod +x /usr/local/bin/calicoctl
        ```
        {: pre}

3. Assurez-vous que les commandes `calico` se sont correctement exécutées en vérifiant la version du client CLI de Calico.

    ```
    calicoctl version
    ```
    {: pre}

4. Si les règles réseau d'entreprise utilisent des proxys ou des pare-feux pour empêcher l'accès de votre système local à des noeuds finaux publics : voir [Exécution des commandes `calicoctl` derrière un pare-feu](cs_firewall.html#firewall) afin d'obtenir les instructions permettant d'autoriser l'accès TCP pour les commandes Calico.

5. Pour Linux et OS X, créez le répertoire `/etc/calico`. Pour Windows, vous pouvez utiliser n'importe quel répertoire.
    ```
    sudo mkdir -p /etc/calico/
    ```
    {: pre}

6. Créez un fichier `calicoctl.cfg`.
    - Linux et OS X :

      ```
      sudo vi /etc/calico/calicoctl.cfg
      ```
      {: pre}

    - Windows : créez le fichier à l'aide d'un éditeur de texte.

7. Entrez les informations suivantes dans le fichier <code>calicoctl.cfg</code>.

    ```
    apiVersion: v1
      kind: calicoApiConfig
      metadata:
      spec:
          etcdEndpoints: <ETCD_URL>
          etcdKeyFile: <CERTS_DIR>/admin-key.pem
          etcdCertFile: <CERTS_DIR>/admin.pem
          etcdCACertFile: <CERTS_DIR>/<ca-*pem_file>
    ```
    {: codeblock}

    1. Extrayez la valeur `<ETCD_URL>`.

      - Linux et OS X :

          ```
          kubectl get cm -n kube-system calico-config -o yaml | grep "etcd_endpoints:" | awk '{ print $2 }'
          ```
          {: pre}

      - Exemple de sortie :

          ```
          https://169.xx.xxx.xxx:30001
          ```
          {: screen}

      - Windows :
        <ol>
        <li>Récupérez les valeurs de configuration calico dans le fichier configmap. </br><pre class="codeblock"><code>kubectl get cm -n kube-system calico-config -o yaml</code></pre></br>
        <li>Dans la section `data`, localisez la valeur etcd_endpoints. Exemple : <code>https://169.xx.xxx.xxx:30001</code>
        </ol>

    2. Extrayez le répertoire `<CERTS_DIR>`, dans lequel les certificats Kubernetes sont téléchargés.

        - Linux et OS X :

          ```
          dirname $KUBECONFIG
          ```
          {: pre}

          Exemple de sortie :

          ```
          /home/sysadmin/.bluemix/plugins/container-service/clusters/<cluster_name>-admin/
          ```
          {: screen}

        - Windows :

          ```
          ECHO %KUBECONFIG%
          ```
          {: pre}

          Exemple de sortie :

          ```
          C:/Users/<user>/.bluemix/plugins/container-service/mycluster-admin/kube-config-prod-dal10-mycluster.yml
          ```
          {: screen}

        **Remarque** : pour obtenir le chemin de répertoire, retirez le nom de fichier `kube-config-prod-<zone>-<cluster_name>.yml` à la fin de la sortie.

    3. Extrayez la valeur `ca-*pem_file`.

        - Linux et OS X :

          ```
          ls `dirname $KUBECONFIG` | grep "ca-"
          ```
          {: pre}

        - Windows :
          <ol><li>Ouvrez le répertoire que vous avez récupéré à l'étape précédente.</br><pre class="codeblock"><code>C:\Users\<user>\.bluemix\plugins\container-service\&lt;cluster_name&gt;-admin\</code></pre>
          <li> Localisez le fichier <code>ca-*pem_file</code>.</ol>

    4. Vérifiez que la configuration Calico fonctionne correctement.

        - Linux et OS X :

          ```
          calicoctl get nodes
          ```
          {: pre}

        - Windows :

          ```
          calicoctl get nodes --config=filepath/calicoctl.cfg
          ```
          {: pre}

          Sortie :

          ```
          NAME
              kube-dal10-crc21191ee3997497ca90c8173bbdaf560-w1.cloud.ibm
              kube-dal10-crc21191ee3997497ca90c8173bbdaf560-w2.cloud.ibm
              kube-dal10-crc21191ee3997497ca90c8173bbdaf560-w3.cloud.ibm
          ```
          {: screen}

          **Important** : les utilisateurs de Windows et OS X doivent inclure l'indicateur `--config=filepath/calicoctl.cfg` à chaque exécution de la commande `calicoctl`.

<br />


## Affichage des règles réseau
{: #view_policies}

Affichez les détails des règles réseau par défaut ou ayant été ajoutées qui sont appliquées à votre cluster.
{:shortdesc}

Avant de commencer :
1. [Installez et configurez l'interface CLI de Calico.](#cli_install)
2. [Ciblez l'interface CLI de Kubernetes sur le cluster](cs_cli_install.html#cs_cli_configure). Incluez l'option `--admin` avec la commande `ibmcloud ks cluster-config`, laquelle est utilisée pour télécharger les fichiers de certificat et d'autorisations. Ce téléchargement inclut également les clés pour le rôle Superutilisateur, dont vous aurez besoin pour exécuter des commandes Calico.
    ```
    ibmcloud ks cluster-config <cluster_name> --admin
    ```
    {: pre}

La compatibilité entre les versions de Calico pour la configuration de l'interface CLI et les règles varient en fonction de la version Kubernetes de votre cluster. Pour installer et configurer l'interface CLI de Calico, cliquez sur l'un des liens suivants en fonction de la version de votre cluster :

* [Clusters Kubernetes version 1.10 ou ultérieure](#1.10_examine_policies)
* [Clusters Kubernetes version 1.9 ou antérieure](#1.9_examine_policies)

Avant de mettre à jour votre cluster de la version Kubernetes 1.9 ou antérieure à la version 1.10 ou ultérieure, consultez la rubrique [Préparation à la mise à jour vers Calico v3](cs_versions.html#110_calicov3).
{: tip}

### Afficher les règles réseau dans les clusters exécutant Kubernetes version 1.10 ou ultérieure
{: #1.10_examine_policies}

Les utilisateurs de Linux n'ont pas besoin d'inclure l'indicateur `--config=filepath/calicoctl.cfg` dans les commandes `calicoctl`.
{: tip}

1. Examinez le noeud final d'hôte Calico.

    ```
    calicoctl get hostendpoint -o yaml --config=filepath/calicoctl.cfg
    ```
    {: pre}

2. Examinez toutes les règles réseau Calico et Kubernetes créées pour le cluster. Cette liste inclut des règles qui n'ont peut-être pas encore été appliquées à des pods ou à des hôtes. Pour qu'une règle réseau soit appliquée, une ressource Kubernetes correspondant au sélecteur défini dans la règle réseau Calico doit être localisée.

    [Les règles réseau ![Icône de lien externe](../icons/launch-glyph.svg "Icône de lien externe")](https://docs.projectcalico.org/v3.1/reference/calicoctl/resources/networkpolicy) sont limitées à des espaces de nom spécifiques :
    ```
    calicoctl get NetworkPolicy --all-namespaces -o wide --config=filepath/calicoctl.cfg
    ```
    {:pre}

    [Les règles réseau globales ![Icône de lien externe](../icons/launch-glyph.svg "Icône de lien externe")](https://docs.projectcalico.org/v3.1/reference/calicoctl/resources/globalnetworkpolicy) ne sont pas limitées à des espaces de nom spécifiques :
    ```
    calicoctl get GlobalNetworkPolicy -o wide --config=filepath/calicoctl.cfg
    ```
    {: pre}

3. Affichez les informations détaillées d'une règle réseau.

    ```
    calicoctl get NetworkPolicy -o yaml <policy_name> --namespace <policy_namespace> --config=filepath/calicoctl.cfg
    ```
    {: pre}

4. Affichez les informations détaillées de toutes les règles réseau globales pour le cluster.

    ```
    calicoctl get GlobalNetworkPolicy -o yaml --config=filepath/calicoctl.cfg
    ```
    {: pre}

### Afficher les règles réseau dans les clusters exécutant Kubernetes version 1.9 ou antérieure
{: #1.9_examine_policies}

Les utilisateurs de Linux n'ont pas besoin d'inclure l'indicateur `--config=filepath/calicoctl.cfg` dans les commandes `calicoctl`.
{: tip}

1. Examinez le noeud final d'hôte Calico.

    ```
    calicoctl get hostendpoint -o yaml --config=filepath/calicoctl.cfg
    ```
    {: pre}

2. Examinez toutes les règles réseau Calico et Kubernetes créées pour le cluster. Cette liste inclut des règles qui n'ont peut-être pas encore été appliquées à des pods ou à des hôtes. Pour qu'une règle réseau soit appliquée, une ressource Kubernetes correspondant au sélecteur défini dans la règle réseau Calico doit être localisée.

    ```
    calicoctl get policy -o wide --config=filepath/calicoctl.cfg
    ```
    {: pre}

3. Affichez les informations détaillées d'une règle réseau.

    ```
    calicoctl get policy -o yaml <policy_name> --config=filepath/calicoctl.cfg
    ```
    {: pre}

4. Affichez les informations détaillées de toutes les règles réseau pour le cluster.

    ```
    calicoctl get policy -o yaml --config=filepath/calicoctl.cfg
    ```
    {: pre}

<br />


## Ajout de règles réseau
{: #adding_network_policies}

Dans la plupart des cas, les règles par défaut n'ont pas besoin d'être modifiées. Seuls les scénarios avancés peuvent demander des modifications. Si vous constatez que vous devez apporter des modifications, vous pouvez créer vos propres règles réseau.
{:shortdesc}

Pour créer des règles réseau Kubernetes, voir la [documentation Kubernetes Network Policies ![Icône de lien externe](../icons/launch-glyph.svg "Icône de lien externe")](https://kubernetes.io/docs/concepts/services-networking/network-policies/).

Pour créer des règles Calico, procédez comme suit :

Avant de commencer :
1. [Installez et configurez l'interface CLI de Calico.](#cli_install)
2. [Ciblez l'interface CLI de Kubernetes sur le cluster](cs_cli_install.html#cs_cli_configure). Incluez l'option `--admin` avec la commande `ibmcloud ks cluster-config`, laquelle est utilisée pour télécharger les fichiers de certificat et d'autorisations. Ce téléchargement inclut également les clés pour le rôle Superutilisateur, dont vous aurez besoin pour exécuter des commandes Calico.
    ```
    ibmcloud ks cluster-config <cluster_name> --admin
    ```
    {: pre}

La compatibilité entre les versions de Calico pour la configuration de l'interface CLI et les règles varient en fonction de la version Kubernetes de votre cluster. Cliquez sur l'un des liens suivants en fonction de la version de votre cluster :

* [Clusters Kubernetes version 1.10 ou ultérieure](#1.10_create_new)
* [Clusters Kubernetes version 1.9 ou antérieure](#1.9_create_new)

Avant de mettre à jour votre cluster de la version Kubernetes 1.9 ou antérieure à la version 1.10 ou ultérieure, consultez la rubrique [Préparation à la mise à jour vers Calico v3](cs_versions.html#110_calicov3).
{: tip}

### Ajout de règles Calico dans les clusters exécutant Kubernetes version 1.10 ou ultérieure
{: #1.10_create_new}

1. Définissez votre [règle réseau ![Icône de lien externe](../icons/launch-glyph.svg "Icône de lien externe")](https://docs.projectcalico.org/v3.1/reference/calicoctl/resources/networkpolicy) ou [règle réseau globale ![Icône de lien externe](../icons/launch-glyph.svg "Icône de lien externe")](https://docs.projectcalico.org/v3.1/reference/calicoctl/resources/globalnetworkpolicy) Calico en créant un script de configuration (`.yaml`). Ces fichiers de configuration incluent les sélecteurs qui décrivent les pods, les espaces de nom ou les hôtes, auxquels s'appliquent ces règles. Reportez-vous à ces [exemples de règles Calico ![Icône de lien externe](../icons/launch-glyph.svg "Icône de lien externe")](http://docs.projectcalico.org/v3.1/getting-started/kubernetes/tutorials/advanced-policy) pour vous aider à créer vos propres règles.
    **Remarque** : Les clusters Kubernetes version 1.10 ou ultérieure doivent utiliser la syntaxe des règles de Calico v3.

2. Appliquez les règles au cluster.
    - Linux et OS X :

      ```
      calicoctl apply -f policy.yaml
      ```
      {: pre}

    - Windows :

      ```
      calicoctl apply -f filepath/policy.yaml --config=filepath/calicoctl.cfg
      ```
      {: pre}

### Ajout de règles Calico dans les clusters exécutant Kubernetes version 1.9 ou antérieure
{: #1.9_create_new}

1. Définissez votre [règle réseau Calico ![Icône de lien externe](../icons/launch-glyph.svg "Icône de lien externe")](http://docs.projectcalico.org/v2.6/reference/calicoctl/resources/policy) en créant un script de configuration (`.yaml`). Ces fichiers de configuration incluent les sélecteurs qui décrivent les pods, les espaces de nom ou les hôtes, auxquels s'appliquent ces règles. Reportez-vous à ces [exemples de règles Calico ![Icône de lien externe](../icons/launch-glyph.svg "Icône de lien externe")](http://docs.projectcalico.org/v2.6/getting-started/kubernetes/tutorials/advanced-policy) pour vous aider à créer vos propres règles.
    **Remarque** : Les clusters Kubernetes version 1.9 ou antérieure doivent utiliser la syntaxe des règles de Calico v2.


2. Appliquez les règles au cluster.
    - Linux et OS X :

      ```
      calicoctl apply -f policy.yaml
      ```
      {: pre}

    - Windows :

      ```
      calicoctl apply -f filepath/policy.yaml --config=filepath/calicoctl.cfg
      ```
      {: pre}

<br />


## Contrôle du trafic entrant vers les services LoadBalancer ou NodePort
{: #block_ingress}

[Par défaut](#default_policy), les services Kubernetes NodePort et LoadBalancer sont conçus pour rendre accessible votre application sur toutes les interfaces de cluster publiques et privées. Vous pouvez toutefois utiliser des règles Calico pour bloquer le trafic entrant vers vos services en fonction de l'origine ou de la destination du trafic.
{:shortdesc}

Un service Kubernetes LoadBalancer est également un service NodePort. Un service LoadBalancer rend accessible votre application via l'adresse IP et le port du service LoadBalancer et la rend accessible via les ports de noeud (NodePort) du service. Les ports de noeud sont accessibles sur toutes les adresses IP (publiques et privées) pour tous les noeuds figurant dans le cluster.

L'administrateur du cluster peut utiliser des règles réseau `preDNAT` Calico pour bloquer :

  - Le trafic vers les services NodePort. Le trafic vers les services LoadBalancer est autorisé.
  - Le trafic basé sur un adresse source ou CIDR.

Quelques utilisations classiques des règles réseau `preDNAT` Calico :

  - Blocage du trafic vers les services NodePort publics d'un service LoadBalancer privé.
  - Blocage du trafic vers les services NodePort publics sur des clusters qui exécutent des [noeuds worker de périphérie](cs_edge.html#edge). Le blocage des services NodePort garantit que les noeuds worker de périphérie sont les seuls noeuds worker à traiter le trafic entrant.

Les règles par défaut de Kubernetes et Calico sont difficiles à appliquer pour protéger les services Kubernetes NodePort et LoadBalancer en raison des règles iptables DNAT générées pour ces services.

Les règles réseau `preDNAT` de Calico peuvent vous aider car elles génèrent des règles iptables en fonction d'une
ressource de règle réseau globale. Les clusters Kubernetes version 1.10 ou ultérieure utilisent [des règles réseau avec la syntaxe `calicoctl.cfg` v3 ![Icône de lien externe](../icons/launch-glyph.svg "Icône de lien externe")](https://docs.projectcalico.org/v3.1/reference/calicoctl/resources/networkpolicy). Les clusters Kubernetes version 1.9 ou antérieure utilisent [des règles réseau avec la syntaxe `calicoctl.cfg` v2 ![Icône de lien externe](../icons/launch-glyph.svg "Icône de lien externe")](https://docs.projectcalico.org/v2.6/reference/calicoctl/resources/policy).

1. Définissez une règle réseau `preDNAT` de Calico pour l'accès entrant (trafic entrant) aux services Kubernetes.

    * Les clusters Kubernetes version 1.10 ou ultérieure doivent utiliser la syntaxe des règles Calico v3.

        Exemple de ressource qui bloque tous les services NodePort :

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

    * Les clusters Kubernetes version 1.9 ou antérieure doivent utiliser la syntaxe des règles Calico v2.

        Exemple de ressource qui bloque tous les services NodePort :

        ```
        apiVersion: v1
        kind: policy
        metadata:
          name: deny-nodeports
        spec:
          preDNAT: true
          selector: ibm.role in { 'worker_public', 'master_public' }
          ingress:
          - action: deny
            protocol: tcp
            destination:
              ports:
              - 30000:32767
          - action: deny
            protocol: udp
            destination:
              ports:
              - 30000:32767
        ```
        {: codeblock}

2. Appliquez la règle réseau preDNAT de Calico. Comptez environ 1 minute pour que les modifications de règle
soient appliquées dans tout le cluster.

  - Linux et OS X :

    ```
    calicoctl apply -f deny-nodeports.yaml
    ```
    {: pre}

  - Windows :

    ```
    calicoctl apply -f filepath/deny-nodeports.yaml --config=filepath/calicoctl.cfg
    ```
    {: pre}

3. Facultatif : dans les clusters à zones multiples, l'équilibreur de charge MZLB effectue un diagnostic d'intégrité des équilibreurs de charge d'application dans chaque zone de votre cluster et conserve les résultats de recherche DNS mis à jour en fonction de ces diagnostics. Si vous utilisez des règles pre-DNAT pour bloquer tout le trafic entrant vers les services Ingress, vous devez également inclure dans la liste blanche les [adresses IP IPv4 de Cloudflare ![Icône de lien externe](../icons/launch-glyph.svg "Icône de lien externe")](https://www.cloudflare.com/ips/) utilisées pour effectuer des diagnostics d'intégrité de ALB. Pour obtenir les étapes nécessaires permettant de créer une règle pre-DAT Calico pour mettre ces adresses IP sur liste blanche, voir la leçon 3 du [Tutoriel sur les règles réseau de Calico](cs_tutorials_policies.html#lesson3).

Pour voir comment mettre des adresses IP source sur liste blanche ou sur liste noire, essayez le [tutoriel sur l'utilisation des règles réseau de Calico pour bloquer le trafic](cs_tutorials_policies.html#policy_tutorial). Pour obtenir d'autres exemples de règles réseau Calico utilisées pour contrôler le trafic en provenance et à destination de votre cluster, vous pouvez consulter les pages [Stars Policy Demo ![Icône de lien externe](../icons/launch-glyph.svg "Icône de lien externe")](https://docs.projectcalico.org/v3.1/getting-started/kubernetes/tutorials/stars-policy/) et [Advanced network policy ![Icône de lien externe](../icons/launch-glyph.svg "Icône de lien externe")](https://docs.projectcalico.org/v3.1/getting-started/kubernetes/tutorials/advanced-policy).
{: tip}

## Contrôle du trafic entre les pods
{: #isolate_services}

Les règles Kubernetes protège les pods du trafic réseau interne. Vous pouvez créer plusieurs règles réseau Kubernetes simples permettant d'isoler les uns des autres des microservices d'application au sein d'un espace de nom ou entre différents espaces de nom.
{: shortdesc}

Pour en savoir plus sur comment les règles réseau Kubernetes contrôlent le trafic de pod à pod et obtenir d'autres exemples de règles, voir la [documentation de Kubernetes  ![Icône de lien externe](../icons/launch-glyph.svg "Icône de lien externe")](https://kubernetes.io/docs/concepts/services-networking/network-policies/).
{: tip}

### Isolement des services d'application au sein d'un espace de nom
{: #services_one_ns}

Le scénario suivant illustre comment gérer le trafic entre les microservices d'application au sein d'un espace de nom.

Une équipe nommée Accounts déploie plusieurs services d'application dans un espace de nom, mais ces services doivent être isolés pour n'autoriser que la communication nécessaire entre les microservices sur le réseau public. Pour l'application Srv1, l'équipe dispose des services de front end et de back end, et d'un service de base de données. Elle indique pour chaque service le libellé `app: Srv1` et le libellé `tier: frontend`, `tier: backend` ou `tier: db`.

<img src="images/cs_network_policy_single_ns.png" width="200" alt="Utilisation d'une règle réseau pour gérer le trafic entre les espaces de nom." style="width:200px; border-style: none"/>

L'équipe Accounts souhaite autoriser le trafic du service de front end vers le service de back end et du service de back end vers le service de base de données. Elle utilise des libellés dans les règles réseau pour désigner les trafics autorisés à circuler entre les microservices.

L'équipe commence par créer une règle réseau Kubernetes autorisant le trafic à partir du service front end vers le service de back end :

```
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: backend-allow
spec:
  podSelector:
    matchLabels:
      app: Srv1
      tier: backend
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: Srv1
          Tier: frontend
```
{: codeblock}

La section `spec.podSelector.matchLabels` répertorie les libellés pour le service de back end Srv1 pour que la règle ne s'applique que _vers_ ces pods. La section `spec.ingress.from.podSelector.matchLabels` répertorie les libellés du service de front end Srv1 pour que le trafic entrant ne soit autorisé que _depuis_ ces pods.

Ensuite, l'équipe crée une règle Kubernetes similaire pour autoriser le trafic du service de back end vers la base de données :

```
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: db-allow
spec:
  podSelector:
    matchLabels:
      app: Srv1
      tier: db
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: Srv1
          Tier: backend
  ```
  {: codeblock}

La section `spec.podSelector.matchLabels` répertorie les libellés pour le service de base de données Srv1 pour que la règle ne s'applique que _vers_ ces pods. La section `spec.ingress.from.podSelector.matchLabels` répertorie les libellés du service de back end Srv1 pour que le trafic entrant ne soit autorisé que _depuis_ ces pods.

Le trafic peut désormais circuler du service de front end vers le service de back end et du service de back end vers le service de base de données. La base de données peut répondre au service de back end et le service de back end peut répondre au service de front end, mais les connexions de trafic inverses ne peuvent pas être établies.

### Isolement des services d'application entre les espaces de nom
{: #services_across_ns}

Le scénario suivant illustre comment gérer le trafic entre les microservices d'application entre plusieurs espaces de nom.

Les services détenus par plusieurs équipes subalternes doivent communiquer mais ces services sont déployés dans différents espaces de nom au sein du même cluster. L'équipe Accounts déploie des services de front end, de back end et de base de données pour l'application app Srv1 dans l'espace de nom Accounts. L'équipe Finance déploie des services de front end, de back end et de base de données pour l'application app Srv2 dans l'espace de nom Finance. Ces deux équipes étiquettent chaque service avec le libellé `app: Srv1` ou `app: Srv2` et le libellé `tier: frontend`, `tier: backend` ou `tier: db`. Elles étiquettent également les espaces de nom avec le libellé `usage: finance` ou `usage: accounts`.

<img src="images/cs_network_policy_multi_ns.png" width="475" alt="Utilisation d'une règle réseau pour gérer le trafic entre différents espaces de nom." style="width:475px; border-style: none"/>

Le service Srv2 de l'équipe Finance doit demander des informations au service de back end Srv1 de l'équipe Accounts. Par conséquent, l'équipe Accounts crée une règle réseau Kubernetes utilisant des libellés pour autoriser tout le trafic en provenance de l'espace de nom Finance vers le service de back end Srv1 dans l'espace de nom Accounts. L'équipe indique également le port 3111 pour isoler l'accès uniquement via ce port.

```
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  Namespace: accounts
  name: accounts-allow
spec:
  podSelector:
    matchLabels:
      app: Srv1
      Tier: backend
  ingress:
  - from:
    - NamespaceSelector:
        matchLabels:
          usage: finance
      ports:
        port: 3111
```
{: codeblock}

La section `spec.podSelector.matchLabels` répertorie les libellés pour le service de back end Srv1 pour que la règle ne s'applique que _vers_ ces pods. La section `spec.ingress.from.NamespaceSelector.matchLabels` répertorie le libellé de l'espace de nom Finance pour que le trafic entrant soit autorisé uniquement _depuis_ les services figurant dans cet espace de nom.

Le trafic peut désormais circuler des microservices Finance vers le service de back end Srv1. Le service de back end Srv1 de Accounts peut répondre aux microservices Finance, mais une connexion de trafic inverse n'est pas possible.

**Remarque** : vous ne pouvez pas autoriser le trafic en provenance de pods d'application spécifiques d'un autre espace de nom car les éléments `podSelector` et `namespaceSelector` ne peuvent pas être combinés. Dans cet exemple, l'ensemble du trafic en provenance de tous les microservices dans l'espace de nom Finance est autorisé.
