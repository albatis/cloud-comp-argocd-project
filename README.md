## 🧭 Visão Geral
Este repositório contém os artefatos de um **Helm Chart** (ou similar) que permitem a gestão declarativa de um projeto do **Argo CD** voltado para *Cloud Comp*.  
O objetivo é fornecer um pacote reutilizável, versionável e gerenciável com **GitOps** para o ambiente **Kubernetes/Argo CD**.

---

## 📂 Estrutura de Arquivos
Abaixo está a descrição de cada um dos principais arquivos presentes na raiz do repositório:

| Arquivo | Função | Observações |
|----------|--------|-------------|
| `.gitignore` | Define quais arquivos/pastas devem ser ignorados pelo Git. | Evita o versionamento de artefatos locais, binários e temporários. |
| `Chart.yaml` | Contém metadados do *Helm Chart* (nome, versão, descrição, dependências). | Permite que o pacote seja tratado como um Chart Helm. |
| `values.yaml` | Define valores padrão de configuração para o Chart. | Permite customização de imagem, réplicas, environment, etc. |
| `README.md` | Este arquivo de documentação. | Explica o propósito e instruções de uso do projeto. |

---

## 🧾 Detalhamento dos Arquivos

### `.gitignore`
Arquivos do Helm Chart para baixar as dependências remotas ou renderização e depuração localmente que não devem ser versionados, como:
- /charts
- Chart.lock

### `Chart.yaml`

O Chart.yaml é o arquivo principal de metadados do seu Helm Chart.

Neste caso, ele define as dependências para a publicação de duas aplicações distintas:

1. **Um Job:** Utilizando o Helm Chart cloud-comp-job.

1. **Uma Aplicação Web:** Utilizando o Helm Chart cloud-comp (com o *alias* playlists-recommender-system).

```
apiVersion: v2
name: modulo
description: Chart para publicação do módulo
type: application
version: 0.0.1
dependencies:
  - name: cloud-comp-job
    version: 0.0.11
    repository: https://albatis.github.io/helm-charts/
  - name: cloud-comp
    version: 0.0.36
    repository: https://albatis.github.io/helm-charts/
    alias: playlists-recommender-system
```
---

### values.yaml

Este arquivo define os valores que o Chart/Helm do projeto utilizará. Você pode sobrescrever ou ajustar conforme seu ambiente (produção, homologação, testes etc).

#### Valores principais para configuração do projeto

```yaml
# Para trocar os arquivos troque o name, URL e INPUT_FILE_PATH
cloud-comp-job:
  name: job-rules
  login: alexandrevieira
  image: albatis/generator-rules-processor:latest
  resources:
    cpu: 0
    memory: 2G
  environments:
    URL: "https://homepages.dcc.ufmg.br/~cunha/hosted/cloudcomp-2023s2-datasets/2023_spotify_ds2.csv"
    MIN_SUPPORT: "0.04"
    INPUT_FILE_PATH: /mnt/2023_spotify_ds2.csv
  commandInitContainer: "rm -f /mnt/*; wget --no-check-certificate $URL -P /mnt;"
  volumes:
    - name: project2-pv
      storageClassName: default-storage-class-alexandrevieira
      size: 1Gi
      path: /mnt
      hostPath: /home/alexandrevieira/project2-pv
      accessMode: ReadWriteMany
      ignore: true
      ignorePVC: true
  
playlists-recommender-system:
  name: playlists-recommender-system
  login: alexandrevieira
  image: albatis/playlists-recommender-system:latest
  replicas: 1
  containerPort: 8000
  servicePort: 8080
  nodePort: 30000
  resources:
    cpu: 0
    memory: 2G
  startupProbe:
    httpGet:
      path: /healthz
      port: 8000
    failureThreshold: 30
    periodSeconds: 1
    timeoutSeconds: 5
  livenessProbe:
    httpGet:
      path: /healthz
      port: 8000
    initialDelaySeconds: 10
    periodSeconds: 10
    timeoutSeconds: 5
    failureThreshold: 3
  readinessProbe:
    httpGet:
      path: /healthz
      port: 8000
    initialDelaySeconds: 5
    periodSeconds: 5
    timeoutSeconds: 3
    failureThreshold: 2
  volumes:
    - name: project2-pv
      storageClassName: default-storage-class-alexandrevieira
      size: 1Gi
      path: /mnt
      #hostPath: /home/alexandrevieira/project2-pv
      #accessMode: ReadWriteMany
      ignore: true
```

# Descrição dos campos (values.yaml)

Abaixo está a descrição dos campos presentes no `values.yaml`.

| Campo        | Descrição |
|--------------|------------|
| `name` | Nome da instância do **Job** ou **Deployment** criado (nome da aplicação). |
| `login` | Usuário de login para o registro de imagem do Docker (usado para `imagePullSecrets`). |
| `image` | Imagem Docker a ser utilizada para o container principal (ex: `repo/imagem:tag`). |
| `resources.cpu` | Limite de CPU (em cores ou milicores) alocado para o container (se `0`, sem limite). |
| `resources.memory` | Limite de memória (ex: `2G`, `512Mi`) alocado para o container. |
| `volumes` | Lista de configurações de **Volumes Persistentes (PV/PVC)** para o container. |
| `volumes.name` | Nome do volume que será montado. |
| `volumes.storageClassName` | Nome da **StorageClass** a ser usada pelo PVC. |
| `volumes.size` | Tamanho do Volume Persistente Reivindicado (PVC) (ex: `1Gi`). |
| `volumes.path` | Caminho no container onde o volume será montado (ex: `/mnt`). |
| `volumes.ignore` | Se `true`, ignora a criação do PV (útil se já existir ou não for necessário). |
| `environments.URL` | Variável de ambiente com a **URL do arquivo de dataset** a ser baixado. |
| `environments.MIN_SUPPORT` | Variável de ambiente com o **valor do suporte mínimo** para o algoritmo (ex: `0.04`). |
| `environments.INPUT_FILE_PATH` | Variável de ambiente com o **caminho completo do arquivo** no volume montado. |
| `commandInitContainer` | Comando executado no **Init Container** para inicializar o Job (ex: remover arquivos antigos e baixar o dataset com `wget`). |
| `volumes.hostPath` | Caminho no **nó host** onde o volume persistente está localizado (usado em ambientes de teste ou desenvolvimento). |
| `volumes.accessMode` | Modo de acesso ao volume (ex: `ReadWriteMany`). |
| `volumes.ignorePVC` | Se `true`, ignora a criação do **Persistent Volume Claim (PVC)**. |
| `replicas` | Número de **réplicas do Pod** (instâncias do container) a serem criadas. |
| `containerPort` | Porta interna do **container** onde a aplicação está ouvindo (ex: `8000`). |
| `servicePort` | Porta do **Service** do Kubernetes que expõe a aplicação internamente. |
| `nodePort` | Porta **externa** do Kubernetes para acesso direto (usada quando `service.type` é `NodePort`). |
| `startupProbe` | Configuração da **Startup Probe** (*Sonda de Inicialização*) — verifica se o container iniciou corretamente. |
| `livenessProbe` | Configuração da **Liveness Probe** (*Sonda de Vida*) — verifica se a aplicação está rodando e deve ser mantida. |
| `readinessProbe` | Configuração da **Readiness Probe** (*Sonda de Prontidão*) — verifica se a aplicação está pronta para receber tráfego. |
| `*.httpGet.path` | Caminho HTTP (endpoint) que a sonda deve acessar (ex: `/healthz`). |
| `*.httpGet.port` | Porta do container que a sonda deve acessar. |
| `*.failureThreshold` | Número de tentativas falhas antes de considerar a sonda como falha. |
| `*.periodSeconds` | Frequência (em segundos) com que a sonda deve ser executada. |
| `*.timeoutSeconds` | Tempo limite (em segundos) para aguardar a resposta da sonda. |
| `*.initialDelaySeconds` | Tempo (em segundos) para aguardar antes de iniciar as verificações da sonda (para **Liveness** e **Readiness**). |

