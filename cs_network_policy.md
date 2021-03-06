---

copyright:
  years: 2014, 2018
lastupdated: "2018-09-25"

---

{:new_window: target="_blank"}
{:shortdesc: .shortdesc}
{:screen: .screen}
{:pre: .pre}
{:table: .aria-labeledby="caption"}
{:codeblock: .codeblock}
{:tip: .tip}
{:download: .download}


# Controlling traffic with network policies
{: #network_policies}

Every Kubernetes cluster is set up with a network plug-in called Calico. Default network policies are set up to secure the public network interface of every worker node in {{site.data.keyword.containerlong}}.
{: shortdesc}

If you have unique security requirements or you have a multizone cluster with VLAN spanning enabled, you can use Calico and Kubernetes to create network policies for a cluster. With Kubernetes network policies, you can specify the network traffic that you want to allow or block to and from a pod within a cluster. To set more advanced network policies such as blocking inbound (ingress) traffic to LoadBalancer services, use Calico network policies.

<ul>
  <li>
  [Kubernetes network policies ![External link icon](../icons/launch-glyph.svg "External link icon")](https://kubernetes.io/docs/concepts/services-networking/network-policies/): These policies specify how pods can communicate with other pods and with external endpoints. As of Kubernetes version 1.8, both incoming and outgoing network traffic can be allowed or blocked based on protocol, port, and source or destination IP addresses. Traffic can also be filtered based on pod and namespace labels. Kubernetes network policies are applied by using `kubectl` commands or the Kubernetes APIs. When these policies are applied, they are automatically converted into Calico network policies and Calico enforces these policies.
  </li>
  <li>
  Calico network policies for Kubernetes version [1.10 and later clusters ![External link icon](../icons/launch-glyph.svg "External link icon")](https://docs.projectcalico.org/v3.1/getting-started/kubernetes/tutorials/advanced-policy) or [1.9 and earlier clusters ![External link icon](../icons/launch-glyph.svg "External link icon")](https://docs.projectcalico.org/v2.6/getting-started/kubernetes/tutorials/advanced-policy): These policies are a superset of the Kubernetes network policies and are applied by using `calicoctl` commands. Calico policies add the following features.
    <ul>
    <li>Allow or block network traffic on specific network interfaces regardless of the Kubernetes pod source or destination IP address or CIDR.</li>
    <li>Allow or block network traffic for pods across namespaces.</li>
    <li>[Block inbound (ingress) traffic to LoadBalancer or NodePort Kubernetes services](#block_ingress).</li>
    </ul>
  </li>
  </ul>

Calico enforces these policies, including any Kubernetes network policies that are automatically converted to Calico policies, by setting up Linux iptables rules on the Kubernetes worker nodes. Iptables rules serve as a firewall for the worker node to define the characteristics that the network traffic must meet to be forwarded to the targeted resource.

To use Ingress and LoadBalancer services, use Calico and Kubernetes policies to manage network traffic into and out of your cluster. Do not use IBM Cloud infrastructure (SoftLayer) [security groups](/docs/infrastructure/security-groups/sg_overview.html#about-security-groups). IBM Cloud infrastructure (SoftLayer) security groups are applied to the network interface of a single virtual server to filter traffic at the hypervisor level. However, security groups do not support the VRRP protocol, which {{site.data.keyword.containerlong_notm}} uses to manage the LoadBalancer IP address. If the VRRP protocol isn't present to manage the LoadBalancer IP, Ingress and LoadBalancer services do not work properly.
{: tip}

<br />


## Default Calico and Kubernetes network policies
{: #default_policy}

When a cluster with a public VLAN is created, a HostEndpoint resource with the `ibm.role: worker_public` label is created automatically for each worker node and its public network interface. To protect the public network interface of a worker node, default Calico policies are applied to any host endpoint with the `ibm.role: worker_public` label.
{:shortdesc}

These default Calico policies allow all outbound network traffic and allow inbound traffic to specific cluster components, such as Kubernetes NodePort, LoadBalancer, and Ingress services. Any other inbound network traffic from the internet to your worker nodes that isn't specified in the default policies is blocked. The default policies don't affect pod to pod traffic.

Review the following default Calico network policies that are automatically applied to your cluster.

**Important:** Do not remove policies that are applied to a host endpoint unless you fully understand the policy. Be sure that you do not need the traffic that is being allowed by the policy.

 <table summary="The first row in the table spans both columns. Read the rest of the rows from left to right, with the server zone in column one and IP addresses to match in column two.">
 <caption>Default Calico policies for each cluster</caption>
  <thead>
  <th colspan=2><img src="images/idea.png" alt="Idea icon"/> Default Calico policies for each cluster</th>
  </thead>
  <tbody>
    <tr>
      <td><code>allow-all-outbound</code></td>
      <td>Allows all outbound traffic.</td>
    </tr>
    <tr>
      <td><code>allow-bigfix-port</code></td>
      <td>Allows incoming traffic on port 52311 to the BigFix app to allow necessary worker node updates.</td>
    </tr>
    <tr>
      <td><code>allow-icmp</code></td>
      <td>Allows incoming ICMP packets (pings).</td>
     </tr>
    <tr>
      <td><code>allow-node-port-dnat</code></td>
      <td>Allows incoming NodePort, LoadBalancer, and Ingress service traffic to the pods that those services are exposing. <strong>Note</strong>: You don't need to specify the exposed ports because Kubernetes uses destination network address translation (DNAT) to forward the service requests to the correct pods. That forwarding takes place before the host endpoint policies are applied in iptables.</td>
   </tr>
   <tr>
      <td><code>allow-sys-mgmt</code></td>
      <td>Allows incoming connections for specific IBM Cloud infrastructure (SoftLayer) systems that are used to manage the worker nodes.</td>
   </tr>
   <tr>
    <td><code>allow-vrrp</code></td>
    <td>Allow VRRP packets, which are used to monitor and move virtual IP addresses between worker nodes.</td>
   </tr>
  </tbody>
</table>

In Kubernetes version 1.10 and newer clusters, a default Kubernetes policy that limits access to the Kubernetes Dashboard is also created. Kubernetes policies don't apply to the host endpoint, but to the `kube-dashboard` pod instead. This policy applies to clusters connected only to a private VLAN and clusters connected to a public and private VLAN.

<table>
<caption>Default Kubernetes policies for each cluster</caption>
<thead>
<th colspan=2><img src="images/idea.png" alt="Idea icon"/> Default Kubernetes policies for each cluster</th>
</thead>
<tbody>
 <tr>
  <td><code>kubernetes-dashboard</code></td>
  <td><b>In Kubernetes v1.10 only</b>, provided in the <code>kube-system</code> namespace: Blocks all pods from accessing the Kubernetes Dashboard. This policy does not impact accessing the dashboard from the {{site.data.keyword.Bluemix_notm}} UI or by using <code>kubectl proxy</code>. If a pod requires access to the dashboard, deploy the pod in a namespace that has the <code>kubernetes-dashboard-policy: allow</code> label.</td>
 </tr>
</tbody>
</table>

<br />


## Installing and configuring the Calico CLI
{: #cli_install}

To view, manage, and add Calico policies, install and configure the Calico CLI.
{:shortdesc}

The compatibility of Calico versions for CLI configuration and policies varies based on the Kubernetes version of your cluster. To install and configure the Calico CLI, click one of the following links based on your cluster version:

* [Kubernetes version 1.10 or later clusters](#1.10_install)
* [Kubernetes version 1.9 or earlier clusters](#1.9_install)

Before you update your cluster from Kubernetes version 1.9 or earlier to version 1.10 or later, review [Preparing to update to Calico v3](cs_versions.html#110_calicov3).
{: tip}

### Install and configure the version 3.1.1 Calico CLI for clusters that are running Kubernetes version 1.10 or later
{: #1.10_install}

Before you begin, [target the Kubernetes CLI to the cluster](cs_cli_install.html#cs_cli_configure). Include the `--admin` option with the `ibmcloud ks cluster-config` command, which is used to download the certificates and permission files. This download also includes the keys to access your infrastructure portfolio and run Calico commands on your worker nodes.

  ```
  ibmcloud ks cluster-config <cluster_name> --admin
  ```
  {: pre}

To install and configure the 3.1.1 Calico CLI:

1. [Download the Calico CLI ![External link icon](../icons/launch-glyph.svg "External link icon")](https://github.com/projectcalico/calicoctl/releases/tag/v3.1.1).

    If you are using OSX, download the `-darwin-amd64` version. If you are using Windows, install the Calico CLI in the same directory as the {{site.data.keyword.Bluemix_notm}} CLI. This setup saves you some filepath changes when you run commands later. Make sure to save the file as `calicoctl.exe`.
    {: tip}

2. For OSX and Linux users, complete the following steps.
    1. Move the executable file to the _/usr/local/bin_ directory.
        - Linux:

          ```
          mv filepath/calicoctl /usr/local/bin/calicoctl
          ```
          {: pre}

        - OS X:

          ```
          mv filepath/calicoctl-darwin-amd64 /usr/local/bin/calicoctl
          ```
          {: pre}

    2. Make the file an executable file.

        ```
        chmod +x /usr/local/bin/calicoctl
        ```
        {: pre}

3. Verify that the `calico` commands ran properly by checking the Calico CLI client version.

    ```
    calicoctl version
    ```
    {: pre}

4. If corporate network policies use proxies or firewalls to prevent access from your local system to public endpoints, [allow TCP access for Calico commands](cs_firewall.html#firewall).

5. For Linux and OS X, create the `/etc/calico` directory. For Windows, any directory can be used.

  ```
  sudo mkdir -p /etc/calico/
  ```
  {: pre}

6. Create a `calicoctl.cfg` file.
    - Linux and OS X:

      ```
      sudo vi /etc/calico/calicoctl.cfg
      ```
      {: pre}

    - Windows: Create the file with a text editor.

7. Enter the following information in the <code>calicoctl.cfg</code> file.

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

    1. Retrieve the `<ETCD_URL>`.

      - Linux and OS X:

          ```
          kubectl get cm -n kube-system calico-config -o yaml | grep "etcd_endpoints:" | awk '{ print $2 }'
          ```
          {: pre}

          Example output:

          ```
          https://169.xx.xxx.xxx:30000
          ```
          {: screen}

      - Windows:
        <ol>
        <li>Get the calico configuration values from the configmap. </br><pre class="codeblock"><code>kubectl get cm -n kube-system calico-config -o yaml</code></pre></br>
        <li>In the `data` section, locate the etcd_endpoints value. Example: <code>https://169.xx.xxx.xxx:30000</code>
        </ol>

    2. Retrieve the `<CERTS_DIR>`, the directory that the Kubernetes certificates are downloaded in.

        - Linux and OS X:

          ```
          dirname $KUBECONFIG
          ```
          {: pre}

          Example output:

          ```
          /home/sysadmin/.bluemix/plugins/container-service/clusters/<cluster_name>-admin/
          ```
          {: screen}

        - Windows:

          ```
          ECHO %KUBECONFIG%
          ```
          {: pre}

            Output example:

          ```
          C:/Users/<user>/.bluemix/plugins/container-service/mycluster-admin/kube-config-prod-dal10-mycluster.yml
          ```
          {: screen}

        **Note**: To get the directory path, remove the file name `kube-config-prod-<zone>-<cluster_name>.yml` from the end of the output.

    3. Retrieve the `ca-*pem_file`.

        - Linux and OS X:

          ```
          ls `dirname $KUBECONFIG` | grep "ca-"
          ```
          {: pre}

        - Windows:
          <ol><li>Open the directory that you retrieved in the last step.</br><pre class="codeblock"><code>C:\Users\<user>\.bluemix\plugins\container-service\&lt;cluster_name&gt;-admin\</code></pre>
          <li> Locate the <code>ca-*pem_file</code> file.</ol>

8. Save the file, and make sure you are in the directory where the file is located.

9. Verify that the Calico configuration is working correctly.

    - Linux and OS X:

      ```
      calicoctl get nodes
      ```
      {: pre}

    - Windows:

      ```
      calicoctl get nodes --config=filepath/calicoctl.cfg
      ```
      {: pre}

      Output:

      ```
      NAME
      kube-dal10-crc21191ee3997497ca90c8173bbdaf560-w1.cloud.ibm
      kube-dal10-crc21191ee3997497ca90c8173bbdaf560-w2.cloud.ibm
      kube-dal10-crc21191ee3997497ca90c8173bbdaf560-w3.cloud.ibm
      ```
      {: screen}


### Installing and configuring the version 1.6.3 Calico CLI for clusters that are running Kubernetes version 1.9 or earlier
{: #1.9_install}

Before you begin, [target the Kubernetes CLI to the cluster](cs_cli_install.html#cs_cli_configure). Include the `--admin` option with the `ibmcloud ks cluster-config` command, which is used to download the certificates and permission files. This download also includes the keys to access your infrastructure portfolio and run Calico commands on your worker nodes.

  ```
  ibmcloud ks cluster-config <cluster_name> --admin
  ```
  {: pre}

To install and configure the 1.6.3 Calico CLI:

1. [Download the Calico CLI ![External link icon](../icons/launch-glyph.svg "External link icon")](https://github.com/projectcalico/calicoctl/releases/tag/v1.6.3).

    If you are using OSX, download the `-darwin-amd64` version. If you are using Windows, install the Calico CLI in the same directory as the {{site.data.keyword.Bluemix_notm}} CLI. This setup saves you some filepath changes when you run commands later.
    {: tip}

2. For OSX and Linux users, complete the following steps.
    1. Move the executable file to the _/usr/local/bin_ directory.
        - Linux:

          ```
          mv filepath/calicoctl /usr/local/bin/calicoctl
          ```
          {: pre}

        - OS X:

          ```
          mv filepath/calicoctl-darwin-amd64 /usr/local/bin/calicoctl
          ```
          {: pre}

    2. Make the file an executable file.

        ```
        chmod +x /usr/local/bin/calicoctl
        ```
        {: pre}

3. Verify that the `calico` commands ran properly by checking the Calico CLI client version.

    ```
    calicoctl version
    ```
    {: pre}

4. If corporate network policies use proxies or firewalls to prevent access from your local system to public endpoints: See [Running `calicoctl` commands from behind a firewall](cs_firewall.html#firewall) for instructions on how to allow TCP access for Calico commands.

5. For Linux and OS X, create the `/etc/calico` directory. For Windows, any directory can be used.
    ```
    sudo mkdir -p /etc/calico/
    ```
    {: pre}

6. Create a `calicoctl.cfg` file.
    - Linux and OS X:

      ```
      sudo vi /etc/calico/calicoctl.cfg
      ```
      {: pre}

    - Windows: Create the file with a text editor.

7. Enter the following information in the <code>calicoctl.cfg</code> file.

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

    1. Retrieve the `<ETCD_URL>`.

      - Linux and OS X:

          ```
          kubectl get cm -n kube-system calico-config -o yaml | grep "etcd_endpoints:" | awk '{ print $2 }'
          ```
          {: pre}

      - Output example:

          ```
          https://169.xx.xxx.xxx:30001
          ```
          {: screen}

      - Windows:
        <ol>
        <li>Get the calico configuration values from the configmap. </br><pre class="codeblock"><code>kubectl get cm -n kube-system calico-config -o yaml</code></pre></br>
        <li>In the `data` section, locate the etcd_endpoints value. Example: <code>https://169.xx.xxx.xxx:30001</code>
        </ol>

    2. Retrieve the `<CERTS_DIR>`, the directory that the Kubernetes certificates are downloaded in.

        - Linux and OS X:

          ```
          dirname $KUBECONFIG
          ```
          {: pre}

          Example output:

          ```
          /home/sysadmin/.bluemix/plugins/container-service/clusters/<cluster_name>-admin/
          ```
          {: screen}

        - Windows:

          ```
          ECHO %KUBECONFIG%
          ```
          {: pre}

          Example output:

          ```
          C:/Users/<user>/.bluemix/plugins/container-service/mycluster-admin/kube-config-prod-dal10-mycluster.yml
          ```
          {: screen}

        **Note**: To get the directory path, remove the file name `kube-config-prod-<zone>-<cluster_name>.yml` from the end of the output.

    3. Retrieve the `ca-*pem_file`.

        - Linux and OS X:

          ```
          ls `dirname $KUBECONFIG` | grep "ca-"
          ```
          {: pre}

        - Windows:
          <ol><li>Open the directory that you retrieved in the last step.</br><pre class="codeblock"><code>C:\Users\<user>\.bluemix\plugins\container-service\&lt;cluster_name&gt;-admin\</code></pre>
          <li> Locate the <code>ca-*pem_file</code> file.</ol>

    4. Verify that the Calico configuration is working correctly.

        - Linux and OS X:

          ```
          calicoctl get nodes
          ```
          {: pre}

        - Windows:

          ```
          calicoctl get nodes --config=filepath/calicoctl.cfg
          ```
          {: pre}

          Output:

          ```
          NAME
          kube-dal10-crc21191ee3997497ca90c8173bbdaf560-w1.cloud.ibm
          kube-dal10-crc21191ee3997497ca90c8173bbdaf560-w2.cloud.ibm
          kube-dal10-crc21191ee3997497ca90c8173bbdaf560-w3.cloud.ibm
          ```
          {: screen}

          **Important**: Windows and OS X users must include the `--config=filepath/calicoctl.cfg` flag every time you run a `calicoctl` command.

<br />


## Viewing network policies
{: #view_policies}

View the details for default and any added network policies that are applied to your cluster.
{:shortdesc}

Before you begin:
1. [Install and configure the Calico CLI.](#cli_install)
2. [Target the Kubernetes CLI to the cluster](cs_cli_install.html#cs_cli_configure). Include the `--admin` option with the `ibmcloud ks cluster-config` command, which is used to download the certificates and permission files. This download also includes the keys to access your infrastructure portfolio and run Calico commands on your worker nodes.
    ```
    ibmcloud ks cluster-config <cluster_name> --admin
    ```
    {: pre}

The compatibility of Calico versions for CLI configuration and policies varies based on the Kubernetes version of your cluster. To install and configure the Calico CLI, click one of the following links based on your cluster version:

* [Kubernetes version 1.10 or later clusters](#1.10_examine_policies)
* [Kubernetes version 1.9 or earlier clusters](#1.9_examine_policies)

Before you update your cluster from Kubernetes version 1.9 or earlier to version 1.10 or later, review [Preparing to update to Calico v3](cs_versions.html#110_calicov3).
{: tip}

### View network policies in clusters that are running Kubernetes version 1.10 or later
{: #1.10_examine_policies}

Linux and Mac users don't need to include the `--config=filepath/calicoctl.cfg` flag in `calicoctl` commands.
{: tip}

1. View the Calico host endpoint.

    ```
    calicoctl get hostendpoint -o yaml --config=filepath/calicoctl.cfg
    ```
    {: pre}

2. View all of the Calico and Kubernetes network policies that were created for the cluster. This list includes policies that might not be applied to any pods or hosts yet. For a network policy to be enforced, a Kubernetes resource must be found that matches the selector that was defined in the Calico network policy.

    [Network policies ![External link icon](../icons/launch-glyph.svg "External link icon")](https://docs.projectcalico.org/v3.1/reference/calicoctl/resources/networkpolicy) are scoped to specific namespaces:
    ```
    calicoctl get NetworkPolicy --all-namespaces -o wide --config=filepath/calicoctl.cfg
    ```
    {:pre}

    [Global network policies ![External link icon](../icons/launch-glyph.svg "External link icon")](https://docs.projectcalico.org/v3.1/reference/calicoctl/resources/globalnetworkpolicy) are not scoped to specific namespaces:
    ```
    calicoctl get GlobalNetworkPolicy -o wide --config=filepath/calicoctl.cfg
    ```
    {: pre}

3. View details for a network policy.

    ```
    calicoctl get NetworkPolicy -o yaml <policy_name> --namespace <policy_namespace> --config=filepath/calicoctl.cfg
    ```
    {: pre}

4. View the details of all global network policies for the cluster.

    ```
    calicoctl get GlobalNetworkPolicy -o yaml --config=filepath/calicoctl.cfg
    ```
    {: pre}

### View network policies in clusters that are running Kubernetes version 1.9 or earlier
{: #1.9_examine_policies}

Linux users don't need to include the `--config=filepath/calicoctl.cfg` flag in `calicoctl` commands.
{: tip}

1. View the Calico host endpoint.

    ```
    calicoctl get hostendpoint -o yaml --config=filepath/calicoctl.cfg
    ```
    {: pre}

2. View all of the Calico and Kubernetes network policies that were created for the cluster. This list includes policies that might not be applied to any pods or hosts yet. For a network policy to be enforced, a Kubernetes resource must be found that matches the selector that was defined in the Calico network policy.

    ```
    calicoctl get policy -o wide --config=filepath/calicoctl.cfg
    ```
    {: pre}

3. View details for a network policy.

    ```
    calicoctl get policy -o yaml <policy_name> --config=filepath/calicoctl.cfg
    ```
    {: pre}

4. View the details of all network policies for the cluster.

    ```
    calicoctl get policy -o yaml --config=filepath/calicoctl.cfg
    ```
    {: pre}

<br />


## Adding network policies
{: #adding_network_policies}

In most cases, the default policies do not need to be changed. Only advanced scenarios might require changes. If you find that you must make changes, you can create your own network policies.
{:shortdesc}

To create Kubernetes network policies, see the [Kubernetes network policy documentation ![External link icon](../icons/launch-glyph.svg "External link icon")](https://kubernetes.io/docs/concepts/services-networking/network-policies/).

To create Calico policies, use the following steps.

Before you begin:
1. [Install and configure the Calico CLI.](#cli_install)
2. [Target the Kubernetes CLI to the cluster](cs_cli_install.html#cs_cli_configure). Include the `--admin` option with the `ibmcloud ks cluster-config` command, which is used to download the certificates and permission files. This download also includes the keys to access your infrastructure portfolio and run Calico commands on your worker nodes.
    ```
    ibmcloud ks cluster-config <cluster_name> --admin
    ```
    {: pre}

The compatibility of Calico versions for CLI configuration and policies varies based on the Kubernetes version of your cluster. Click one of the following links based on your cluster version:

* [Kubernetes version 1.10 or later clusters](#1.10_create_new)
* [Kubernetes version 1.9 or earlier clusters](#1.9_create_new)

Before you update your cluster from Kubernetes version 1.9 or earlier to version 1.10 or later, review [Preparing to update to Calico v3](cs_versions.html#110_calicov3).
{: tip}

### Adding Calico policies in clusters that are running Kubernetes version 1.10 or later
{: #1.10_create_new}

1. Define your Calico [network policy ![External link icon](../icons/launch-glyph.svg "External link icon")](https://docs.projectcalico.org/v3.1/reference/calicoctl/resources/networkpolicy) or [global network policy ![External link icon](../icons/launch-glyph.svg "External link icon")](https://docs.projectcalico.org/v3.1/reference/calicoctl/resources/globalnetworkpolicy) by creating a configuration script (`.yaml`). These configuration files include the selectors that describe what pods, namespaces, or hosts that these policies apply to. Refer to these [sample Calico policies ![External link icon](../icons/launch-glyph.svg "External link icon")](http://docs.projectcalico.org/v3.1/getting-started/kubernetes/tutorials/advanced-policy) to help you create your own.
    **Note**: Kubernetes version 1.10 or later clusters must use Calico v3 policy syntax.

2. Apply the policies to the cluster.
    - Linux and OS X:

      ```
      calicoctl apply -f policy.yaml
      ```
      {: pre}

    - Windows:

      ```
      calicoctl apply -f filepath/policy.yaml --config=filepath/calicoctl.cfg
      ```
      {: pre}

### Adding Calico policies in clusters that are running Kubernetes version 1.9 or earlier
{: #1.9_create_new}

1. Define your [Calico network policy ![External link icon](../icons/launch-glyph.svg "External link icon")](http://docs.projectcalico.org/v2.6/reference/calicoctl/resources/policy) by creating a configuration script (`.yaml`). These configuration files include the selectors that describe what pods, namespaces, or hosts that these policies apply to. Refer to these [sample Calico policies ![External link icon](../icons/launch-glyph.svg "External link icon")](http://docs.projectcalico.org/v2.6/getting-started/kubernetes/tutorials/advanced-policy) to help you create your own.
    **Note**: Kubernetes version 1.9 or earlier clusters must use Calico v2 policy syntax.


2. Apply the policies to the cluster.
    - Linux and OS X:

      ```
      calicoctl apply -f policy.yaml
      ```
      {: pre}

    - Windows:

      ```
      calicoctl apply -f filepath/policy.yaml --config=filepath/calicoctl.cfg
      ```
      {: pre}

<br />


## Controlling inbound traffic to LoadBalancer or NodePort services
{: #block_ingress}

[By default](#default_policy), Kubernetes NodePort and LoadBalancer services are designed to make your app available on all public and private cluster interfaces. However, you can use Calico policies to block incoming traffic to your services based on traffic source or destination.
{:shortdesc}

A Kubernetes LoadBalancer service is also a NodePort service. A LoadBalancer service makes your app available over the LoadBalancer IP address and port and makes your app available over the service's NodePorts. NodePorts are accessible on every IP address (public and private) for every node within the cluster.

The cluster administrator can use Calico `preDNAT` network policies to block:

  - Traffic to NodePort services. Traffic to LoadBalancer services is allowed.
  - Traffic that is based on a source address or CIDR.

Some common uses for Calico `preDNAT` network policies:

  - Block traffic to public NodePorts of a private LoadBalancer service.
  - Block traffic to public NodePorts on clusters that are running [edge worker nodes](cs_edge.html#edge). Blocking NodePorts ensures that the edge worker nodes are the only worker nodes that handle incoming traffic.

Default Kubernetes and Calico policies are difficult to apply to protecting Kubernetes NodePort and LoadBalancer services due to the DNAT iptables rules generated for these services.

Calico `preDNAT` network policies can help you because they generate iptables rules based on a Calico
network policy resource. Kubernetes version 1.10 or later clusters use [network policies with `calicoctl.cfg` v3 syntax ![External link icon](../icons/launch-glyph.svg "External link icon")](https://docs.projectcalico.org/v3.1/reference/calicoctl/resources/networkpolicy). Kubernetes version 1.9 or earlier clusters use [policies with `calicoctl.cfg` v2 syntax ![External link icon](../icons/launch-glyph.svg "External link icon")](https://docs.projectcalico.org/v2.6/reference/calicoctl/resources/policy).

1. Define a Calico `preDNAT` network policy for ingress (inbound traffic) access to Kubernetes services.

    * Kubernetes version 1.10 or later clusters must use Calico v3 policy syntax.

        Example resource that blocks all NodePorts:

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

    * Kubernetes version 1.9 or earlier clusters must use Calico v2 policy syntax.

        Example resource that blocks all NodePorts:

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

2. Apply the Calico preDNAT network policy. It takes about 1 minute for the
policy changes to be applied throughout the cluster.

  - Linux and OS X:

    ```
    calicoctl apply -f deny-nodeports.yaml
    ```
    {: pre}

  - Windows:

    ```
    calicoctl apply -f filepath/deny-nodeports.yaml --config=filepath/calicoctl.cfg
    ```
    {: pre}

3. Optional: In multizone clusters, a MZLB health checks the ALBs in each zone of your cluster and keeps the DNS lookup results updated based on these health checks. If you use pre-DNAT policies to block all incoming traffic to Ingress services, you must also whitelist [Cloudflare's IPv4 IPs ![External link icon](../icons/launch-glyph.svg "External link icon")](https://www.cloudflare.com/ips/) that are used to check the health of your ALBs. For steps on how to create a Calico pre-DNAT policy to whitelist these IPs, see Lesson 3 of the [Calico network policy tutorial](cs_tutorials_policies.html#lesson3).

To see how to whitelist or blacklist source IP addresses, try the [Using Calico network policies to block traffic tutorial](cs_tutorials_policies.html#policy_tutorial). For more example Calico network policies that control traffic to and from your cluster, you can check out the [stars policy demo ![External link icon](../icons/launch-glyph.svg "External link icon")](https://docs.projectcalico.org/v3.1/getting-started/kubernetes/tutorials/stars-policy/) and the [advanced network policy ![External link icon](../icons/launch-glyph.svg "External link icon")](https://docs.projectcalico.org/v3.1/getting-started/kubernetes/tutorials/advanced-policy).
{: tip}

## Isolating clusters on the private network
{: #isolate_workers}

If you have a multizone cluster, multiple VLANs for a single zone cluster, or multiple subnets on the same VLAN, you must [enable VLAN spanning](/docs/infrastructure/vlans/vlan-spanning.html#vlan-spanning) so that your worker nodes can communicate with each other on the private network. However, when VLAN spanning is enabled, any system that is connected to any of the private VLANs in the same IBM Cloud account can communicate with workers.

You can isolate your cluster from other systems on the private network by applying [Calico private network policies ![External link icon](../icons/launch-glyph.svg "External link icon")](https://github.com/IBM-Cloud/kube-samples/tree/master/calico-policies/private-network-isolation). This set of Calico policies and host endpoints isolate the private network traffic of a cluster from other resources in the account's private network.

The policies target the worker node private interface (eth0) and the pod network of a cluster.

**Worker nodes**

* Private interface egress is permitted only to pod IPs, workers in this cluster, and the UPD/TCP port 53 for DNS access.
* Private interface ingress is permitted only from workers in the cluster and only to DNS, kubelet, ICMP, and VRRP.

**Pods**

* All ingress to pods is permitted from workers in the cluster.
* Egress from pods is restricted only to public IPs, DNS, kubelet, and other pods in the cluster.

Before you begin:
1. [Install and configure the Calico CLI.](#cli_install)
2. [Target the Kubernetes CLI to the cluster](cs_cli_install.html#cs_cli_configure). Include the `--admin` option with the `ibmcloud ks cluster-config` command, which is used to download the certificates and permission files. This download also includes the keys to access your infrastructure portfolio and run Calico commands on your worker nodes.
    ```
    ibmcloud ks cluster-config <cluster_name> --admin
    ```
    {: pre}

To isolate your cluster on the private network using Calico policies:

1. Clone the `IBM-Cloud/kube-samples` respository.
    ```
    git clone https://github.com/IBM-Cloud/kube-samples.git
    ```
    {: pre}

2. Navigate to the private policy directory for the Calico version that your cluster version is compatible with.
    * Kubernetes version 1.10 or later clusters:
      ```
      cd <filepath>/IBM-Cloud/kube-samples/calico-policies/private-network-isolation/calico-v3
      ```
      {: pre}

    * Kubernetes version 1.9 or earlier clusters:
      ```
      cd <filepath>/IBM-Cloud/kube-samples/calico-policies/private-network-isolation/calico-v2
      ```
      {: pre}

3. Set up a policy for the private host endpoint.
    1. Open the `generic-privatehostendpoint.yaml` policy.
    2. Replace `<worker_name>` with the name of a worker node and `<worker-node-private-ip>` with the private IP address for the worker node. To see your worker nodes' private IPs, run `ibmcloud ks workers --cluster <my_cluster>`.
    3. Repeat this step in a new section for each worker node in your cluster.
    **Note**: Each time you add a worker node to a cluster, you must update the host endpoints file with the new entries.

4. Apply all of the policies to your cluster.
    - Linux and OS X:

      ```
      calicoctl apply -f allow-all-workers-private.yaml
      calicoctl apply -f allow-dns-10250.yaml
      calicoctl apply -f allow-egress-pods.yaml
      calicoctl apply -f allow-icmp-private.yaml
      calicoctl apply -f allow-vrrp-private.yaml
      calicoctl apply -f generic-privatehostendpoint.yaml
      ```
      {: pre}

    - Windows:

      ```
      calicoctl apply -f allow-all-workers-private.yaml --config=filepath/calicoctl.cfg
      calicoctl apply -f allow-dns-10250.yaml --config=filepath/calicoctl.cfg
      calicoctl apply -f allow-egress-pods.yaml --config=filepath/calicoctl.cfg
      calicoctl apply -f allow-icmp-private.yaml --config=filepath/calicoctl.cfg
      calicoctl apply -f allow-vrrp-private.yaml --config=filepath/calicoctl.cfg
      calicoctl apply -f generic-privatehostendpoint.yaml --config=filepath/calicoctl.cfg
      ```
      {: pre}

## Controlling traffic between pods
{: #isolate_services}

Kubernetes policies protect pods from internal network traffic. You can create simple Kubernetes network policies to isolate app microservices from each other within a namespace or across namespaces.
{: shortdesc}

For more information about how Kubernetes network policies control pod-to-pod traffic and for more example policies, see the [Kubernetes documentation ![External link icon](../icons/launch-glyph.svg "External link icon")](https://kubernetes.io/docs/concepts/services-networking/network-policies/).
{: tip}

### Isolate app services within a namespace
{: #services_one_ns}

The following scenario demonstrates how to manage traffic between app microservices within one namespace.

An Accounts team deploys multiple app services in one namespace, but they need isolation to permit only necessary communication between the microservices over the public network. For the app Srv1, the team has frontend, backend, and database services. They label each service with the `app: Srv1` label and the `tier: frontend`, `tier: backend`, or `tier: db` label.

<img src="images/cs_network_policy_single_ns.png" width="200" alt="Use a network policy to manage cross-namespace traffic." style="width:200px; border-style: none"/>

The Accounts team wants to allow traffic from the frontend to the backend, and from the backend to the database. They use labels in their network policies to designate which traffic flows are permitted between microservices.

First, they create a Kubernetes network policy that allows traffic from the frontend to the backend:

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

The `spec.podSelector.matchLabels` section lists the labels for the Srv1 backend service so that the policy applies only _to_ those pods. The `spec.ingress.from.podSelector.matchLabels` section lists the labels for the Srv1 frontend service so that ingress is permitted only _from_ those pods.

Then, they create a similar Kubernetes network policy that allows traffic from the backend to the database:

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

The `spec.podSelector.matchLabels` section lists the labels for the Srv1 database service so that the policy applies only _to_ those pods. The `spec.ingress.from.podSelector.matchLabels` section lists the labels for the Srv1 backend service so that ingress is permitted only _from_ those pods.

Traffic can now flow from the frontend to the backend, and from the backend to the database. The database can respond to the backend, and the backend can respond to the frontend, but no reverse traffic connections can be established.

### Isolate app services between namespaces
{: #services_across_ns}

The following scenario demonstrates how to manage traffic between app microservices across multiple namespaces.

Services owned by different subteams need to communicate, but the services are deployed in different namespaces within the same cluster. The Accounts team deploys frontend, backend, and database services for the app Srv1 in the accounts namespace. The Finance team deploys frontend, backend, and database services for the app Srv2 in the finance namespace. Both teams label each service with the `app: Srv1` or `app: Srv2` label and the `tier: frontend`, `tier: backend`, or `tier: db` label. They also label the namespaces with the `usage: accounts` or `usage: finance` label.

<img src="images/cs_network_policy_multi_ns.png" width="475" alt="Use a network policy to manage cross-namepsace traffic." style="width:475px; border-style: none"/>

The Finance team's Srv2 needs to call information from the Accounts team's Srv1 backend. So the Accounts team creates a Kubernetes network policy that uses labels to allow all traffic from the finance namespace to the Srv1 backend in the accounts namespace. The team also specifies the port 3111 to isolate access through that port only.

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

The `spec.podSelector.matchLabels` section lists the labels for the Srv1 backend service so that the policy applies only _to_ those pods. The `spec.ingress.from.NamespaceSelector.matchLabels` section lists the label for the finance namespace so that ingress is permitted only _from_ services in that namespace.

Traffic can now flow from finance microservices to the accounts Srv1 backend. The accounts Srv1 backend can respond to finance microservices, but can't establish a reverse traffic connection.

**Note**: You can't allow traffic from specific app pods in another namespace because `podSelector` and `namespaceSelector` can't be combined. In this example, all traffic from all microservices in the finance namespace is permitted.

## Logging denied traffic
{: #log_denied}

To log denied traffic requests to certain pods in your cluster, you can create a Calico log network policy.
{: shortdesc}

When you set up network policies to limit traffic to app pods, traffic requests that are not permitted by these policies are denied and dropped. In some scenarios, you might want more information about denied traffic requests. For example, you might notice some unusual traffic that is continuously being denied by one of your network policies. To monitor the potential security threat, you can set up logging to record every time that the policy denies an attempted action on specified app pods.

Before you begin:
1. [Install and configure the Calico CLI.](#cli_install) **Note**: The policies in these steps use Calico v3 syntax that is compatible with clusters that run Kubernetes version 1.10 or later. For clusters that run Kubernetes version 1.9 or earlier, you must use [Calico v2 policy syntax ![External link icon](../icons/launch-glyph.svg "External link icon")](http://docs.projectcalico.org/v2.6/reference/calicoctl/resources/policy).
2. [Target the Kubernetes CLI to the cluster](cs_cli_install.html#cs_cli_configure). Include the `--admin` option with the `ibmcloud ks cluster-config` command, which is used to download the certificates and permission files. This download also includes the keys to access your infrastructure portfolio and run Calico commands on your worker nodes.
    ```
    ibmcloud ks cluster-config <cluster_name> --admin
    ```
    {: pre}

To create a Calico policy to log denied traffic:

1. Create or use an existing Kubernetes or Calico network policy that blocks or limits incoming traffic. For example, to control traffic between pods, you might use the following example Kubernetes policy named `access-nginx` that limits access to an NGINX app. Incoming traffic to pods that are labeled "run=nginx" is allowed only from pods with the "run=access" label. All other incoming traffic to the "run=nginx" app pods is blocked.
    ```
    kind: NetworkPolicy
    apiVersion: extensions/v1beta1
    metadata:
      name: access-nginx
    spec:
      podSelector:
        matchLabels:
          run: nginx
      ingress:
        - from:
          - podSelector:
              matchLabels:
                run: access
    ```
    {: codeblock}

2. Apply the policy.
    * To apply a Kubernetes policy:
        ```
        kubectl apply -f <policy_name>.yaml
        ```
        {: pre}
        The Kubernetes policy is automatically converted to a Calico NetworkPolicy so that Calico can apply it as iptables rules.

    * To apply a Calico policy:
        ```
        calicoctl apply -f <policy_name>.yaml --config=<filepath>/calicoctl.cfg
        ```
        {: pre}

3. If you applied a Kubernetes policy, review the syntax of the automatically created Calico policy and copy the value of the `spec.selector` field.
    ```
    calicoctl get policy -o yaml <policy_name> --config=<filepath>/calicoctl.cfg
    ```
    {: pre}

    For example, after being applied and converted, the `access-nginx` policy has the following Calico v3 syntax. The `spec.selector` field has the value `projectcalico.org/orchestrator == 'k8s' && run == 'nginx'`.
    ```
    apiVersion: projectcalico.org/v3
    kind: NetworkPolicy
    metadata:
      name: access-nginx
    spec:
      ingress:
      - action: Allow
        destination: {}
        source:
          selector: projectcalico.org/orchestrator == 'k8s' && run == 'access'
      order: 1000
      selector: projectcalico.org/orchestrator == 'k8s' && run == 'nginx'
      types:
      - Ingress
    ```
    {: screen}

4. To log all the traffic that is denied by the Calico policy that you created earlier, create a Calico NetworkPolicy named `log-denied-packets`. For example, use the following policy to log all packets that were denied by the network policy that you defined in step 1. The log policy uses the same pod selector as the example `access-nginx` policy, which adds this policy to the Calico iptables rule chain. By using a higher order number, such as `3000`, you can ensure that this rule is added to the end of the iptables rule chain. Any request packet from the "run=access" pod that matches the `access-nginx` policy rule is accepted by the "run=nginx" pods.  However, when packets from any other source try to match the low-order `access-nginx` policy rule, they are denied. Those packets then try to match the high-order `log-denied-packets` policy rule. `log-denied-packets` logs any packets that arrive to it, so only packets that were denied by the "run=nginx" pods are logged. After the packets' attempts are logged, the packets are dropped.
    ```
    apiVersion: projectcalico.org/v3
    kind: NetworkPolicy
    metadata:
      name: log-denied-packets
    spec:
      types:
      - Ingress
      ingress:
      - action: Log
        destination: {}
        source: {}
      selector: projectcalico.org/orchestrator == 'k8s' && run == 'nginx'
      order: 3000
    ```
    {: codeblock}

    <table>
    <caption>Understanding the log policy YAML components</caption>
    <thead>
    <th colspan=2><img src="images/idea.png" alt="Idea icon"/> Understanding the log policy YAML components</th>
    </thead>
    <tbody>
    <tr>
     <td><code>types</code></td>
     <td>This <code>Ingress</code> policy applies incoming traffic requests. <strong>Note:</strong> The value <code>Ingress</code> is a general term for all incoming traffic, and does not refer to traffic only from the IBM Ingress ALB.</td>
    </tr>
     <tr>
      <td><code>ingress</code></td>
      <td><ul><li><code>action</code>: The <code>Log</code> action writes a log entry for any requests that match this policy to the `/var/log/syslog` path on the worker node.</li><li><code>destination</code>: No destination is specified because the <code>selector</code> applies this policy to all pods with a certain label.</li><li><code>source</code>: This policy applies to requests from any source.</td>
     </tr>
     <tr>
      <td><code>selector</code></td>
      <td>Replace &lt;selector&gt; with the same selector in the `spec.selector` field that you used in your Calico policy from step 1 or that you found in the Calico syntax for your Kubernetes policy in step 3. For example, by using the selector <code>selector: projectcalico.org/orchestrator == 'k8s' && run == 'nginx'</code>, this policy's rule is added to the same iptables chain as the <code>access-nginx</code> sample network policy rule in step 1. This policy applies only to incoming network traffic to pods that use the same pod selector label.</td>
     </tr>
     <tr>
      <td><code>order</code></td>
      <td>Calico policies have orders that determine when they are applied to incoming request packets. Policies with lower orders, such as <code>1000</code>, are applied first. Policies with higher orders are applied after the lower-order policies. For example, a policy with a very high order, such as <code>3000</code>, is effectively applied last after all the lower-order policies have been applied.</br></br>Incoming request packets go through the iptables rules chain and try to match rules from lower-order policies first. If a packet matches any rule, the packet is accepted. However, if a packet doesn't match any rule, it arrives at the last rule in the iptables rules chain with the highest order. To make sure this is the last policy in the chain, use a much higher order, such as <code>3000</code>, than the policy you created in step 1.</td>
     </tr>
    </tbody>
    </table>

5. Apply the policy.
    ```
    calicoctl apply -f log-denied-packets.yaml --config=<filepath>/calicoctl.cfg
    ```
    {: pre}

6. [Forward the logs](cs_health.html#configuring) from `/var/log/syslog` to {{site.data.keyword.loganalysislong}} or an external syslog server.
