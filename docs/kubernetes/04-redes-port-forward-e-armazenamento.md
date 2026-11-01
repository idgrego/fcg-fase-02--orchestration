# Redes no Kubernetes, Port-Forward e Persistência de Dados

No quarto módulo do guia de **Kubernetes (K8s)**, você aprenderá como os Pods conversam entre si através da rede interna do cluster, como o **CoreDNS** viabiliza a descoberta de serviços sem acoplamento de IPs, como utilizar o **`kubectl port-forward`** para testar e depurar aplicações da sua máquina local (Windows) e como funciona a **persistência de dados** (PV, PVC, StorageClass) em contraste com o modelo de microsserviços *Stateless* conectados ao **Azure Database**.

---

## 📑 Sumário

- [1. O Modelo de Redes do Kubernetes (Pod Networking)](#1-o-modelo-de-redes-do-kubernetes-pod-networking)
  - [1.1. Os Três Princípios Fundamentais da Rede K8s](#11-os-três-princípios-fundamentais-da-rede-k8s)
  - [1.2. O Conceito de Rede Plana (Flat Network) sem NAT Interno](#12-o-conceito-de-rede-plana-flat-network-sem-nat-interno)
  - [1.3. A Interface CNI (Container Network Interface)](#13-a-interface-cni-container-network-interface)
- [2. Descoberta de Serviços e DNS Interno (CoreDNS)](#2-descoberta-de-serviços-e-dns-interno-coredns)
  - [2.1. O Nome de Domínio Totalmente Qualificado (FQDN)](#21-o-nome-de-domínio-totalmente-qualificado-fqdn)
  - [2.2. Resolução no Mesmo Namespace vs. Namespaces Diferentes](#22-resolução-no-mesmo-namespace-vs-namespaces-diferentes)
- [3. O Poder do `kubectl port-forward` (Túnel Local)](#3-o-poder-do-kubectl-port-forward-túnel-local)
  - [3.1. Por que o Port-Forward é Indispensável para o Desenvolvedor?](#31-por-que-o-port-forward-é-indispensável-para-o-desenvolvedor)
  - [3.2. Como Criar Túneis para os Serviços do FIAP Cloud Games](#32-como-criar-túneis-para-os-serviços-do-fiap-cloud-games)
  - [3.3. Testando as Interfaces do Scalar e o Painel do RabbitMQ](#33-testando-as-interfaces-do-scalar-e-o-painel-do-rabbitmq)
- [4. Persistência de Dados e Armazenamento no Cluster](#4-persistência-de-dados-e-armazenamento-no-cluster)
  - [4.1. Volumes Efêmeros (`emptyDir`)](#41-volumes-efêmeros-emptydir)
  - [4.2. PersistentVolume (PV) e PersistentVolumeClaim (PVC)](#42-persistentvolume-pv-e-persistentvolumeclaim-pvc)
  - [4.3. Provisionamento Dinâmico com `StorageClass`](#43-provisionamento-dinâmico-com-storageclass)
- [5. Decisão Arquitetural: Microsserviços Stateless e Azure Database](#5-decisão-arquitetural-microsserviços-stateless-e-azure-database)
  - [5.1. Por que as APIs não Usam Discos Locais no Cluster?](#51-por-que-as-apis-não-usam-discos-locais-no-cluster)
  - [5.2. A Persistência Delegada ao SGBD PaaS na Nuvem](#52-a-persistência-delegada-ao-sgbd-paas-na-nuvem)
- [6. Navegação](#6-navegação)

---

## 1. O Modelo de Redes do Kubernetes (Pod Networking)

Diferente do Docker tradicional ([📖 Ver Módulo de Redes no Docker](../docker/04-volumes-e-redes.md)), onde contêineres em hosts diferentes exigem mapeamentos complexos de portas para se enxergarem, o Kubernetes implementa um modelo de rede limpo e altamente padronizado.

### 1.1. Os Três Princípios Fundamentais da Rede K8s

1. **Todo Pod possui um IP único e exclusivo:** Não importa em qual nó físico o Pod esteja executando, ele ganha um endereço IP real na rede interna do cluster.
2. **Comunicação Direta Pod-to-Pod sem NAT:** Todos os Pods conseguem enviar pacotes TCP/UDP para qualquer outro Pod do cluster diretamente, sem tradução de endereços de rede (NAT).
3. **Agentes do Nó conversam com todos os Pods:** Os processos do nó (como o `kubelet`) comunicam-se com todos os Pods daquela máquina usando o IP do Pod.

### 1.2. O Conceito de Rede Plana (Flat Network) sem NAT Interno

```mermaid
graph LR
    subgraph Node1 ["Worker Node 1 (IP Host: 192.168.1.10)"]
        Pod1["📦 users-api-1<br/>(IP: 10.244.1.5)"]
        Pod2["📦 catalog-api-1<br/>(IP: 10.244.1.6)"]
    end

    subgraph Node2 ["Worker Node 2 (IP Host: 192.168.1.11)"]
        Pod3["📦 payments-api-1<br/>(IP: 10.244.2.8)"]
        Pod4["📦 rabbitmq-1<br/>(IP: 10.244.2.9)"]
    end

    Pod1 -->|"Roteamento Direto sem NAT<br/>(10.244.1.5 -> 10.244.2.9)"| Pod4
    Pod3 -->|"Roteamento Direto sem NAT<br/>(10.244.2.8 -> 10.244.1.6)"| Pod2
```

O contêiner dentro do Pod enxerga a si mesmo exatamente com o mesmo endereço IP que o resto do cluster utiliza para se comunicar com ele. Isso elimina os problemas clássicos de portas estáticas duplicadas.

### 1.3. A Interface CNI (Container Network Interface)
O Kubernetes não implementa o driver de rede por conta própria; ele delega essa função para um plugin que implementa a especificação **CNI**:
- Em clusters locais como o Docker Desktop ou Minikube, o CNI padrão vem pré-configurado com roteamento bridge transparente.
- Em clusters de produção em nuvem corporativa (como o **Azure AKS**), adota-se o **Azure CNI** (onde cada Pod ganha um IP real da Virtual Network - VNet do Azure) ou soluções de alta performance baseadas em eBPF como o **Cilium** ou **Calico**.

---

## 2. Descoberta de Serviços e DNS Interno (CoreDNS)

Como os IPs dos Pods mudam a cada novo deploy, os microsserviços **nunca** devem se comunicar utilizando IPs numéricos. Toda a comunicação utiliza o **Service Discovery** provido pelo serviço interno **`CoreDNS`**.

### 2.1. O Nome de Domínio Totalmente Qualificado (FQDN)

Sempre que um manifesto de `Service` ([📖 Ver Aula 02](02-manifestos-e-tipos-de-configuracao.md)) é criado no cluster, o CoreDNS registra automaticamente um registro de DNS interno seguindo o padrão:

$$\text{\bf <nome-do-service>.<namespace>.svc.cluster.local}$$

### 2.2. Resolução no Mesmo Namespace vs. Namespaces Diferentes

```mermaid
graph TD
    ClientPod["📦 Pod: catalog-api (Namespace: default)"]

    DNS["🔍 CoreDNS Server (kube-system)"]

    SvcLocal["🌐 Service: rabbitmq-service (default)"]
    SvcOutro["🌐 Service: auth-service (producao)"]

    ClientPod -->|"1. Busca 'rabbitmq-service'"| DNS
    DNS -->|"2. Resolução Curta no Mesmo Namespace"| SvcLocal

    ClientPod -->|"3. Busca 'auth-service.producao'"| DNS
    DNS -->|"4. Resolução Cross-Namespace"| SvcOutro
```

- **No mesmo Namespace (Padrão):**  
  A `UsersAPI` e a `CatalogAPI` conversam com o broker simplesmente utilizando o nome curto:  
  `amqp://guest:guest@rabbitmq-service:5672`
- **Entre Namespaces Diferentes:**  
  Se um serviço estiver no namespace `dev` e precisar chamar outro no namespace `shared`:  
  `http://meu-servico.shared.svc.cluster.local:80`

---

## 3. O Poder do `kubectl port-forward` (Túnel Local)

### 3.1. Por que o Port-Forward é Indispensável para o Desenvolvedor?

Por padrão, nossos microsserviços utilizam o tipo `ClusterIP` em seus manifestos `04-service.yaml`. Isso significa que eles estão **isolados dentro do cluster** e seu navegador no Windows não consegue acessá-los diretamente digitando `localhost`.

Em vez de alterar o manifesto para `NodePort` ou instalar um complexo Ingress Controller local, o comando **`kubectl port-forward`** cria um **túnel TCP seguro diretamente entre uma porta da sua máquina host (Windows) e o Service ou Pod do cluster**.

### 3.2. Como Criar Túneis para os Serviços do FIAP Cloud Games

Abra uma janela de terminal e execute o túnel para o serviço que deseja testar:

```powershell
# 1. Túnel para a UsersAPI (Mapeia a porta 8080 do Windows para a porta 80 do Service)
kubectl port-forward svc/users-api-service 8080:80

# 2. Túnel para a CatalogAPI (Porta 8082 do Windows)
kubectl port-forward svc/catalog-api-service 8082:80

# 3. Túnel para a PaymentsAPI (Porta 8083 do Windows)
kubectl port-forward svc/payments-api-service 8083:80

# 4. Túnel para o Painel Web Management do RabbitMQ (Porta 15672)
kubectl port-forward svc/rabbitmq-service 15672:15672
```

> 💡 **Como funciona a sintaxe:**  
> `kubectl port-forward svc/<nome-do-service> <porta-no-windows>:<porta-do-service>`  
> O terminal permanecerá aberto mantendo o túnel ativo. Para encerrar o túnel, pressione `Ctrl + C`.

---

### 3.3. Testando as Interfaces do Scalar e o Painel do RabbitMQ

Com os túneis ativos, abra o navegador da sua máquina no Windows e acesse instantaneamente:

- **UsersAPI (Scalar OpenAPI):** [http://localhost:8080/scalar/v1](http://localhost:8080/scalar/v1)
- **CatalogAPI (Scalar OpenAPI):** [http://localhost:8082/scalar/v1](http://localhost:8082/scalar/v1)
- **PaymentsAPI (Scalar OpenAPI):** [http://localhost:8083/scalar/v1](http://localhost:8083/scalar/v1)
- **RabbitMQ Management UI:** [http://localhost:15672](http://localhost:15672) *(Login: `guest` / `guest`)*

Você pode disparar requisições HTTP pelo Scalar ou Postman da sua máquina física, e o tráfego será encapsulado através do túnel até as réplicas em execução no Kubernetes!

---

## 4. Persistência de Dados e Armazenamento no Cluster

No Kubernetes, assim como no Docker ([📖 Ver Módulo de Volumes](../docker/04-volumes-e-redes.md)), o sistema de arquivos de um contêiner é efêmero. Para dados que precisam persistir além da vida de um Pod, o K8s oferece abstrações dedicadas.

### 4.1. Volumes Efêmeros (`emptyDir`)
- Criado quando o Pod é agendado em um nó e existe apenas enquanto aquele Pod estiver vivo.
- Compartilhado entre múltiplos contêineres dentro do mesmo Pod.
- **Caso de Uso:** Cache temporário em memória RAM ou armazenamento transitório de arquivos temporários durante uma requisição.

---

### 4.2. PersistentVolume (PV) e PersistentVolumeClaim (PVC)

O Kubernetes separa a alocação de armazenamento em duas responsabilidades distintas:

```mermaid
graph LR
    subgraph StorageInfra ["Infraestrutura de TI / Nuvem"]
        PV["💾 PersistentVolume (PV)<br/>(Disco Real de 50GB no Azure)"]
    end

    subgraph DevLayer ["Desenvolvedor / Aplicação"]
        PVC["📋 PersistentVolumeClaim (PVC)<br/>('Preciso de 10GB ReadWriteOnce')"]
        Pod["📦 Pod do RabbitMQ"]
    end

    PVC -->|"1. Solicita e Faz Binding"| PV
    Pod -->|"2. Monta em /var/lib/rabbitmq"| PVC
```

- **`PersistentVolume` (PV):** É o pedaço físico de armazenamento provisionado no cluster (um disco rígido virtual do Azure Disk, NFS ou AWS EBS). Ele tem ciclo de vida próprio e **não pertence a nenhum namespace**.
- **`PersistentVolumeClaim` (PVC):** É o "pedido de compra" feito pelo desenvolvedor. O desenvolvedor declara: *"Minha aplicação precisa de 10 GB de disco com acesso de leitura/escrita exclusivo (ReadWriteOnce)"*. O Kubernetes encontra um PV compatível e conecta os dois (*Binding*).

---

### 4.3. Provisionamento Dinâmico com `StorageClass`
Em nuvens corporativas como o **Azure**, ninguém cria PVs manualmente. Utiliza-se um **`StorageClass`**:
- O desenvolvedor cria apenas o PVC;
- O `StorageClass` contata a API do Microsoft Azure automaticamente, aloca um novo disco gerenciado (*Azure Managed Disk - SSD*) e cria o PV correspondente nos bastidores de forma transparente.

---

## 5. Decisão Arquitetural: Microsserviços Stateless e Azure Database

### 5.1. Por que as APIs não Usam Discos Locais no Cluster?

Observe uma decisão de design corporativo fundamental do projeto **FIAP Cloud Games**:  
Nos manifestos `03-deployment.yaml` da `UsersAPI`, `CatalogAPI`, `PaymentsAPI` e `NotificationsAPI`, **não há nenhuma montagem de volume de disco (PVC)**.

**Por que adotamos essa arquitetura?**
- Nossos microsserviços seguem rigorosamente os princípios de **Aplicações Doze-Fatores (*Twelve-Factor App - VI. Processos Stateless*)**.
- Processos sem estado (*Stateless*) podem ser criados, destruídos, reiniciados e escalados para 100 réplicas em segundos sem nenhum risco de corrupção de arquivos em disco ou conflitos de concorrência (*File Locking*).

---

### 5.2. A Persistência Delegada ao SGBD PaaS na Nuvem

Toda a persistência de dados das aplicações é delegada para o **Azure Database** (`fcg-server.database.windows.net`), com bancos relacionais isolados por microsserviço (*Database-per-Service*):
- `UsersAPI` $\rightarrow$ `fcg-02-usersdb`
- `CatalogAPI` $\rightarrow$ `fcg-02-catalogdb`
- `PaymentsAPI` $\rightarrow$ `fcg-02-paymentsdb`

```mermaid
graph TD
    subgraph K8sCluster ["☸️ Cluster Kubernetes (Pods Stateless)"]
        Pod1["👤 users-api (Réplica 1)"]
        Pod2["👤 users-api (Réplica 2)"]
        Pod3["👤 users-api (Réplica 3)"]
    end

    subgraph AzurePaaS ["☁️ Microsoft Azure (Banco Gerenciado PaaS)"]
        AzureDB[("☁️ Azure Database<br/>Server: fcg-server.database.windows.net:1433<br/>Database: fcg-02-usersdb")]
    end

    Pod1 -->|"Conexão Segura TLS/SSL (Criptografada)"| AzureDB
    Pod2 -->|"Conexão Segura TLS/SSL (Criptografada)"| AzureDB
    Pod3 -->|"Conexão Segura TLS/SSL (Criptografada)"| AzureDB
```

Essa abordagem combina o melhor de dois mundos:
1. **Elasticidade Extrema no Kubernetes:** As APIs escalam horizontalmente no cluster sob demanda.
2. **Confiabilidade e Governança na Nuvem:** O banco de dados usufrui de alta disponibilidade, replicação geográfica, backups automáticos de ponto no tempo (*Point-in-Time Restore*) e criptografia de ponta a ponta sem onerar a equipe de sustentação do cluster Kubernetes.

---

## 6. Navegação

- ⬅️ **[Voltar para a Aula 03: Atualização de Código e Ciclo de Vida (`03-atualizacao-de-codigo-rollout-e-ciclo-de-vida.md`)](03-atualizacao-de-codigo-rollout-e-ciclo-de-vida.md)**
- 📑 **[Ir para o Índice do Guia de Kubernetes (`index.md`)](index.md)**
- ➡️ **[Avançar para a Aula 05: Recursos Avançados e Guia de Comandos (`05-recursos-avancados-e-guia-de-comandos.md`)](05-recursos-avancados-e-guia-de-comandos.md)**
