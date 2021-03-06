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
{:tsSymptoms: .tsSymptoms}
{:tsCauses: .tsCauses}
{:tsResolve: .tsResolve}


# Utilizzo di {{site.data.keyword.containerlong_notm}} con {{site.data.keyword.Bluemix_notm}} privato
{: #hybrid_iks_icp}

Se hai un account {{site.data.keyword.Bluemix}} privato, puoi usarlo con i servizi {{site.data.keyword.Bluemix_notm}} di selezione, incluso {{site.data.keyword.containerlong}}. Per ulteriori informazioni, consulta il blog sull'argomento [Hybrid experience across {{site.data.keyword.Bluemix_notm}} Private and IBM Public Cloud![Icona link esterno](../icons/launch-glyph.svg "Icona link esterno")](http://ibm.biz/hybridJune2018).
{: shortdesc}

Hai compreso le offerte [{{site.data.keyword.Bluemix_notm}}](cs_why.html#differentiation). Ora puoi [connettere il tuo cloud pubblico e privato](#hybrid_vpn) e [riutilizzare i tuoi pacchetti privati per i contenitori pubblici](#hybrid_ppa_importer).

## Connessione del tuo cloud pubblico e privato con la VPN strongSwan
{: #hybrid_vpn}

Stabilisci una connettività VPN tra il tuo cluster Kubernetes pubblico e la tua istanza {{site.data.keyword.Bluemix}} privato per consentire una comunicazione bidirezionale.
{: shortdesc}

1.  [Crea un cluster in {{site.data.keyword.Bluemix}} privato![Icona link esterno](../icons/launch-glyph.svg "Icona link esterno")](https://www.ibm.com/support/knowledgecenter/SSBS6K_2.1.0.3/installing/installing.html).

2.  Crea un cluster standard con {{site.data.keyword.containerlong}} in {{site.data.keyword.Bluemix}} pubblico oppure usane uno esistente. Per creare un cluster, scegli tra le seguenti opzioni:  
    - [Crea un cluster standard dalla GUI](cs_clusters.html#clusters_ui). 
    - [Crea un cluster standard dalla CLI](cs_clusters.html#clusters_cli). 
    - [Usa CAM (Cloud Automation Manager) per creare un cluster utilizzando un template predefinito![Icona link esterno](../icons/launch-glyph.svg "Icona link esterno")](https://www.ibm.com/support/knowledgecenter/SS2L37_2.1.0.3/cam_deploy_IKS.html). Quando distribuisci un cluster con CAM, viene installato automaticamente il tiller Helm. 

3.  Nel tuo cluster {{site.data.keyword.Bluemix}} privato, distribuisci il servizio VPN IPSec strongSwan. 

    1.  [Completa le soluzioni temporanee della VPN IPSec strongSwan ![Icona link esterno](../icons/launch-glyph.svg "Icona link esterno")](https://www.ibm.com/support/knowledgecenter/SS2L37_2.1.0.3/cam_strongswan.html). 

    2.  [Installa il grafico Helm della VPN strongSwan![Icona link esterno](../icons/launch-glyph.svg "Icona link esterno")](https://www.ibm.com/support/knowledgecenter/SSBS6K_2.1.0.3/app_center/create_release.html) nel tuo cluster privato.

4.  Ottieni l'indirizzo IP pubblico del gateway VPN {{site.data.keyword.Bluemix}} privato. L'indirizzo IP faceva parte della tua [configurazione preliminare del cluster privato![Icona link esterno](../icons/launch-glyph.svg "Icona link esterno")](https://www.ibm.com/support/knowledgecenter/SSBS6K_2.1.0.3/installing/prep_cluster.html).

5.  Nel tuo cluster {{site.data.keyword.Bluemix}} pubblico, [distribuisci il servizio VPN IPSec strongSwan](cs_vpn.html#vpn-setup). Usa l'indirizzo IP pubblico del passo precedente e assicurati di configurare il tuo gateway VPN in {{site.data.keyword.Bluemix}} pubblico per la [connessione in uscita](cs_vpn.html#strongswan_3) in modo che la connessione VPN venga avviata dal cluster in {{site.data.keyword.Bluemix}} pubblico. 

6.  [Verifica la connessione VPN](cs_vpn.html#vpn_test) tra i cluster.

7.  Ripeti questi passi per ciascun cluster che vuoi collegare.  


## Esecuzione delle immagini di {{site.data.keyword.Bluemix_notm}} privato in contenitori Kubernetes pubblici
{: #hybrid_ppa_importer}

Puoi eseguire i prodotti IBM su licenza di selezione forniti per {{site.data.keyword.Bluemix_notm}} privato in un cluster in {{site.data.keyword.Bluemix_notm}} pubblico.  
{: shortdesc}

Il software su licenza è disponibile in [IBM Passport Advantage ![Icona link esterno](../icons/launch-glyph.svg "Icona link esterno")](https://www-01.ibm.com/software/passportadvantage/index.html). Per usare questo software in un cluster in {{site.data.keyword.Bluemix_notm}} pubblico, devi scaricarlo, estrarre l'immagine e caricarla nel tuo spazio dei nomi in {{site.data.keyword.registryshort}}. Indipendentemente dall'ambiente in cui intendi usare il software, devi ottenere innanzitutto la licenza richiesta per il prodotto.  

La seguente tabella costituisce una panoramica dei prodotti {{site.data.keyword.Bluemix_notm}} privato disponibili che puoi usare nel tuo cluster in {{site.data.keyword.Bluemix_notm}} pubblico.

| Nome prodotto| Versione | Numero parte|
| --- | --- | --- |
| IBM Db2 Direct Advanced Edition Server | 11.1 | CNU3TML |
| IBM Db2 Advanced Enterprise Server Edition Server | 11.1 | CNU3SML |
| IBM MQ Advanced | 9.0.5 | CNU1VML |
| IBM WebSphere Application Server Liberty | 16.0.0.3 | immagine Docker Hub |
{: caption="Tabella. Prodotti {{site.data.keyword.Bluemix_notm}} privato supportati da usare in {{site.data.keyword.Bluemix_notm}} pubblico." caption-side="top"}

Prima di iniziare: 
- [Installa il plugin della CLI {{site.data.keyword.registryshort}} (`ibmcloud cr`)](/docs/services/Registry/registry_setup_cli_namespace.html#registry_cli_install). 
- [Configura lo spazio dei nomi in {{site.data.keyword.registryshort}}](/docs/services/Registry/registry_setup_cli_namespace.html#registry_namespace_add) o recupera il tuo spazio dei nomi esistente eseguendo `ibmcloud cr namespaces`. 
- [Indirizza la tua CLI `kubectl` al tuo cluster](/docs/containers/cs_cli_install.html#cs_cli_configure). 
- [Installa la CLI Helm e imposta tiller nel tuo cluster](/docs/containers/cs_integrations.html#helm). 

Per distribuire un'immagine di {{site.data.keyword.Bluemix_notm}} privato in un cluster in {{site.data.keyword.Bluemix_notm}} pubblico:

1.  Segui i passi presenti nella [documentazione di {{site.data.keyword.registryshort}}](/docs/services/Registry/ts_index.html#ts_ppa) per scaricare il software su licenza da IBM Passport Advantage, invia l'immagine al tuo spazio dei nomi e installa il grafico Helm nel tuo cluster. 

    **Per IBM WebSphere Application Server Liberty**:
    
    1.  Invece di ottenere l'immagine da IBM Passport Advantage, usa l'[immagine Docker Hub ![Icona link esterno](../icons/launch-glyph.svg "Icona link esterno")](https://hub.docker.com/_/websphere-liberty/). Per istruzioni su come ottenere una licenza, vedi [Upgrading the image from Docker Hub to a production image![Icona link esterno](../icons/launch-glyph.svg "Icona link esterno")](https://github.com/WASdev/ci.docker/tree/master/ga/production-upgrade).
    
    2.  Segui le [istruzioni del grafico Helm Liberty![Icona link esterno](../icons/launch-glyph.svg "Icona link esterno")](https://www.ibm.com/support/knowledgecenter/en/SSEQTP_liberty/com.ibm.websphere.wlp.doc/ae/rwlp_icp_helm.html). 

2.  Verifica che lo **STATO** del grafico Helm mostri `DISTRIBUITO`. Nel caso in cui non lo sia, attendi alcuni minuti e ritenta.
    ```
    helm status <helm_chart_name>
    ```
    {: pre}
   
3.  Consulta la documentazione specifica del prodotto per ulteriori informazioni su come configurare e usare il prodotto con il tuo cluster.  

    - [IBM Db2 Direct Advanced Edition Server ![Icona link esterno](../icons/launch-glyph.svg "Icona link esterno")](https://www.ibm.com/support/knowledgecenter/en/SSEPGG_11.1.0/com.ibm.db2.luw.licensing.doc/doc/c0070181.html) 
    - [IBM MQ Advanced ![Icona link esterno](../icons/launch-glyph.svg "Icona link esterno")](https://www.ibm.com/support/knowledgecenter/en/SSFKSJ_9.0.0/com.ibm.mq.helphome.v90.doc/WelcomePagev9r0.html)
    - [IBM WebSphere Application Server Liberty ![Icona link esterno](../icons/launch-glyph.svg "Icona link esterno")](https://www.ibm.com/support/knowledgecenter/en/SSEQTP_liberty/as_ditamaps/was900_welcome_liberty.html)
