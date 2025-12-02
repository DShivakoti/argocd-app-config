### ArgoCD gitops demo

Originally from Nana Janashia

Original gitlab repo: https://gitlab.com/nanuchi/argocd-app-config.git
YouTube Video: https://www.youtube.com/watch?v=MeU5_k9ssrs


#### Commands

```bash
# install ArgoCD in k8s
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# access ArgoCD UI
kubectl get svc -n argocd
kubectl port-forward svc/argocd-server 8080:443 -n argocd

# login with admin user and below token (as in documentation):
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 --decode && echo

# you can change and delete init password

```
</br>

#### Links

* Config repo: [https://gitlab.com/nanuchi/argocd-app-config](https://gitlab.com/nanuchi/argocd-app-config)

* Docker repo: [https://hub.docker.com/repository/docker/nanajanashia/argocd-app](https://hub.docker.com/repository/docker/nanajanashia/argocd-app)

* Install ArgoCD: [https://argo-cd.readthedocs.io/en/stable/getting_started/#1-install-argo-cd](https://argo-cd.readthedocs.io/en/stable/getting_started/#1-install-argo-cd)

* Login to ArgoCD: [https://argo-cd.readthedocs.io/en/stable/getting_started/#4-login-using-the-cli](https://argo-cd.readthedocs.io/en/stable/getting_started/#4-login-using-the-cli)

* ArgoCD Configuration: [https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/)




## Architecture Overview


```mermaid
flowchart TD
    subgraph OCPAPP["OpenShift"]
        OCPApp["**Applications**"]
        OCPdals(OpenShift Cluster Log Forwarder)
        OCPpils(Logback TCP)
    end

    subgraph ADCMAPP["AD-CMD"]
        ADCMApp["**Applications**"]
        ADCMdals(Filebeat)
        ADCMpils(Filebeat)
        ADCMpils1(Filebeat)
    end

    subgraph Pipelines["**Grafana Loki in OpenShift**"]
        LogsPipeline
    end

    subgraph LogsPipeline[" "]
        vector[**Vector Logs Processor**<br> Enriches logs by adding labels]
        Loki[**Loki Distributor**<br>Routes logs to tenants based on system name.]

        %% Tenants
        AppTenant[**Application Tenant**:<br>tenant:<*system*>-app<br>tenant = pin-app<br>log type = DALS]

        PayloadTenant[**Payload Tenant**:<br>tenant:<*system*>-payload<br>tenant = pin-payload<br>log type = PILS]

        InfraTenant[**Infrastcuture Tenant**:<br>tenant:<*system*>-infra<br>tenant = pin-infra<br>log type = PILS]

        LokiIngester(**Loki Ingester**)

        LokiQueryFrontend(**Loki Query Frontend**)
        LokiCompactor(**Loki Compactor**<br>Compaction & Retention<br>based on log type PILS & DALS)

    end




    subgraph Storage[**External S3 Storage**]
        S3["**S3 Logs Bucket**"]
    end


    OCPApp -->|stdout applicattion logs| OCPdals
    OCPApp -->|payload logs| OCPpils

    OCPdals -->|tenant = <*system*>-app <br>tenant = pin-app| vector
    OCPpils -->|tenant = <*system*>-payload<br>tenant = pin-payload| vector


    ADCMApp -->|application logs| ADCMdals
    ADCMApp -->|payload logs| ADCMpils
    ADCMApp -->|infra logs| ADCMpils1

    ADCMdals -->|tenant = <*system*>-app<br>tenant = pin-app| vector
    ADCMpils1 -->|tenant = <*system*>-infra<br>tenant = pin-infra| vector
    ADCMpils -->|tenant = <*system*>-payload<br>tenant = pin-payload| vector



    vector --- |Forward Logs| Loki
    Loki --> |Route logs to dedicated tenant| AppTenant --> LokiIngester
    Loki --> |Route logs to dedicated tenant| PayloadTenant --> LokiIngester
    Loki --> |Route logs to dedicated tenant| InfraTenant --> LokiIngester

    LokiIngester L0_Grafana_Animation_0@ <-- Fetch Logs --> LokiQueryFrontend



    %% Visualization
    Viz@{ label: "<img src=\" /services/observability/references/images/users.svg\" alt="Users" width="30" height="30">Users" } --> Grafana


    subgraph Visualization[**Visualization**]
        Grafana@{ label: "**Grafana UI** <img src="https://grafana.apps.os-pci-global.finods.com/public/build/static/img/grafana_icon.1e0deb6b.svg" alt="Grafana" width="30" height="30">\n        **Logs**\n📄 📊" }

        Grafana ---> |"OpenShift oauth login"| Oauth[""OpenShift Oauth]
    end

    LokiIngester -- Stores Logs --> S3
    LokiQueryFrontend L1_Grafana_Animation_0@ <-- Fetch Logs --> S3



    %% Compactor Flow
    LokiCompactor -->|Compacts logs| S3
    LokiCompactor -->|Applies retention| S3


    Grafana L2_Grafana_Animation_0@ <-- Query Logs --> LokiQueryFrontend



    Viz@{ shape: rounded}
    OCPApp@{ shape: processes}
    ADCMApp@{ shape: processes}
    S3@{ shape: lin-cyl}
    Grafana@{ shape: rounded}
    Oauth@{ shape: rounded}

    %% color %%
    OCPApp:::application
    ADCMApp:::application


    vector:::loki
    Loki:::loki
    LokiIngester:::loki
    LokiQueryFrontend:::loki
    LokiCompactor:::loki
    AppTenant:::tenant
    PayloadTenant:::tenant
    InfraTenant:::tenant
    S3:::storage
    Grafana:::vizFlow
    Oauth:::vizFlow
    Viz:::users
    Storage:::storage

    classDef loki stroke:#f39c12,stroke-width:1px
    classDef tenant stroke:#27ae60,stroke-width:1px
    classDef vizFlow stroke:#8e44ad,stroke-width:1px
    classDef application stroke:#587275,stroke-width:1px
    classDef storage stroke:#c9ca4f,stroke-width:1px
    classDef users stroke:#978776,stroke-width:1px

    L0_Grafana_Animation_0@{ animation: slow }
    L1_Grafana_Animation_0@{ animation: slow }
    L2_Grafana_Animation_0@{ animation: slow }

    click Grafana href "../../grafana/references/grafana-instances.md" "Grafana Instances"
    click Oauth href "../../../deploy/argocd/how-to/from-container-to-env/index.md/#getting-access-to-the-openshift-cluster" "OpenShift Instances"

```



# Loki Monitoring & Performance Dashboard

## Loki Troubleshooting Flowchart

When investigating Loki performance issues, follow this systematic approach:

```mermaid
flowchart TD
    Start([Start Investigation]) --> LocalOps
    
    subgraph LocalOps[Local Operational Checks]
        direction LR
        L1[Dashboard]
        L2[General health check]
        L3[Request rates]
        L4[Latency overview]
    end
    
    LocalOps --> ReadPathIssue{Read Path Issue?}
    
    ReadPathIssue -->|Yes| ReadPath
    ReadPathIssue -->|No| WritePathIssue{Write Path Issue?}
    
    subgraph ReadPath[Read Path Investigation]
        direction TB
        RP1[Loki Reads Dashboard]
        RP2[<b>Query performance</b><br/>- Query-related metrics<br/>- Querier metrics]
    end
    
    subgraph WritePath[Write Path Investigation]
        direction TB
        WP1[Loki Writers Dashboard]
        WP2[<b>Push performance</b><br/>- Distributor metrics<br/>- Ingester metrics]
    end
    
    WritePathIssue -->|Yes| WritePath
    WritePathIssue -->|No| ResourceIssue{Resource Issue?}
    
    ReadPath --> ResourceIssue
    WritePath --> ResourceIssue
    
    ResourceIssue -->|Yes| Resources
    ResourceIssue -->|No| ComponentLogs{Need Component Logs?}
    
    subgraph Resources[Resource Investigation]
        direction LR
        Res1[Read Path]
        Res2[Write Path]
        
        subgraph Res1Sub[Loki Reads Resources]
            R1a[CPU, memory usage]
            R1b[Query path components]
        end
        
        subgraph Res2Sub[Loki Writers Resources]
            R2a[CPU, memory usage]
            R2b[Write path components]
        end
    end
    
    Resources --> ComponentLogs
    
    ComponentLogs -->|Yes| Logs
    ComponentLogs -->|No| StorageIssue{Cloud/Storage Issue?}
    
    subgraph Logs[Logs Investigation]
        Log1[Loki Logs Dashboard]
        Log2[<b>Component logs</b><br/>- Process metrics]
    end
    
    Logs --> StorageIssue
    
    StorageIssue -->|Yes| Storage
    StorageIssue -->|No| CompactorIssue{Compactor Issue?}
    
    subgraph Storage[Storage Investigation]
        Store1[Loki Chunks Dashboard]
        Store2[<b>Chunk metrics</b><br/>- Flush statistics<br/>- Utilization]
    end
    
    Storage --> CompactorIssue
    
    CompactorIssue -->|Yes| Compactor
    CompactorIssue -->|No| End([Continue Monitoring])
    
    subgraph Compactor[Compactor Investigation]
        Comp1[Loki Compactor Dashboard]
        Comp2[<b>Compactor metrics</b><br/>- Compaction requests]
    end
    
    Compactor --> End
    
    %% Styling
    style Start fill:#4CAF50,stroke:#388E3C,stroke-width:2px,color:white
    style End fill:#4CAF50,stroke:#388E3C,stroke-width:2px,color:white
    style ReadPathIssue fill:#FF9800,stroke:#F57C00,stroke-width:2px,color:white
    style WritePathIssue fill:#FF9800,stroke:#F57C00,stroke-width:2px,color:white
    style ResourceIssue fill:#FF9800,stroke:#F57C00,stroke-width:2px,color:white
    style ComponentLogs fill:#FF9800,stroke:#F57C00,stroke-width:2px,color:white
    style StorageIssue fill:#FF9800,stroke:#F57C00,stroke-width:2px,color:white
    style CompactorIssue fill:#FF9800,stroke:#F57C00,stroke-width:2px,color:white
    
    style LocalOps fill:#E3F2FD,stroke:#2196F3,stroke-width:2px
    style ReadPath fill:#F3E5F5,stroke:#9C27B0,stroke-width:2px
    style WritePath fill:#F3E5F5,stroke:#9C27B0,stroke-width:2px
    style Resources fill:#FFF3E0,stroke:#FF9800,stroke-width:2px
    style Logs fill:#E8F5E8,stroke:#4CAF50,stroke-width:2px
    style Storage fill:#FFEBEE,stroke:#F44336,stroke-width:2px
    style Compactor fill:#E1BEE7,stroke:#7B1FA2,stroke-width:2px
```

## Dashboard Links

- **Loki Reads Dashboard**: `/grafana/d/loki-reads`
- **Loki Writers Dashboard**: `/grafana/d/loki-writes`  
- **Loki Reads Resources**: `/grafana/d/loki-reads-resources`
- **Loki Writers Resources**: `/grafana/d/loki-writes-resources`
- **Loki Logs Dashboard**: `/grafana/d/loki-logs`
- **Loki Chunks Dashboard**: `/grafana/d/loki-chunks`
- **Loki Compactor Dashboard**: `/grafana/d/loki-compactor`

## Quick Reference Commands

```bash
# Check current query performance
histogram_quantile(0.95, rate(loki_request_duration_seconds_bucket[5m]))

# Check ingestion rate
sum(rate(loki_distributor_received_bytes_total[5m]))

# Check active streams
loki_ingester_memory_streams

# Check errors
sum(rate(loki_request_failures_total[5m])) by (route, status_code)
```
