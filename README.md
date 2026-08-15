<div align="center">
  <img src="https://kind.sigs.k8s.io/logo/logo.png" alt="Logo do kind" width="360">

  # Laboratório local de Kubernetes com kind

  Aprenda a criar um cluster Kubernetes local, publicar uma imagem Docker e
  acessar sua aplicação pelo navegador.

  <a href="https://kind.sigs.k8s.io/">Documentação do kind</a>
  ·
  <a href="https://youtube.com/@marialazaradev">Canal da Maria</a>
</div>

## Sobre este laboratório

Este repositório acompanha uma aula prática para quem está dando os primeiros
passos com Docker, Kubernetes e `kind`.

Ao final, você terá uma página HTML rodando dentro de um container Nginx, em um
cluster Kubernetes local, acessível em:

```text
http://localhost:8080
```

O laboratório foi pensado para ser executado no computador do desenvolvedor,
sem precisar publicar a imagem em um registry e sem precisar de uma nuvem.

## Antes de começar

Se você ainda não conhece Docker e Kubernetes, recomendo assistir aos vídeos
abaixo antes de executar o laboratório. A ordem foi pensada para construir os
conceitos gradualmente.

### 1. Aprenda os fundamentos de Docker

Comece entendendo sobre o Docker:

[Assistir à introdução sobre Docker](https://youtu.be/t7HcqfEuBcU)

### 2. Aprenda os fundamentos de Kubernetes

Depois, conheça os conceitos básicos de Kubernetes:

[Assistir à introdução sobre Kubernetes](https://youtu.be/z3hOWY46OMQ)

Com essa base, você conseguirá acompanhar o laboratório com mais tranquilidade
e entender por que a imagem Docker precisa ser carregada nos nodes do kind,
como o Deployment cria os Pods e como o Service encaminha as requisições.

## O que você vai aprender

- Como transformar uma página HTML em uma imagem Docker.
- Como criar um cluster Kubernetes local usando `kind`.
- Por que uma imagem criada no Docker local precisa ser carregada no `kind`.
- Como um `Deployment` cria e mantém Pods em execução.
- Como um `Service` encontra os Pods usando labels.
- Como expor uma aplicação local usando `NodePort`.
- Como acompanhar o estado da aplicação com `kubectl`.

## O resultado final

```text
index.html
    |
    | docker build
    v
kind-lab:local
    |
    | kind load docker-image
    v
Nós do cluster kind
    |
    v
Deployment hello-kind
    |
    v
Dois Pods com Nginx
    |
    v
Service NodePort :30080
    |
    | extraPortMappings
    v
http://localhost:8080
```

## Pre-requisitos

Antes de iniciar, instale e teste estes componentes:

- Docker, com o daemon em execução.
- `kind`.
- `kubectl`.
- Um terminal para executar os comandos.

O `kind` executa os nós do Kubernetes como containers Docker. Portanto, o
Docker precisa estar aberto e funcionando antes de criar o cluster.

### Documentação de instalação

Use a documentação oficial de cada ferramenta para instalar os componentes no
seu sistema operacional:

- [Docker Desktop para macOS](https://docs.docker.com/desktop/setup/install/mac-install/)
- [Docker Desktop para Windows](https://docs.docker.com/desktop/setup/install/windows-install/)
- [Docker Engine para Linux](https://docs.docker.com/engine/install/)
- [kind - Instalação](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)
- [kubectl - Instalação](https://kubernetes.io/docs/tasks/tools/)

No Windows, o Docker Desktop recomenda o uso do backend **WSL 2**. O
laboratório pode ser executado pelo PowerShell ou por um terminal Linux dentro
do WSL 2.

Depois da instalação, confirme que `docker`, `kind` e `kubectl` estão
disponíveis no terminal e que o Docker está em execução.

## Como acompanhar o lab

Execute os passos na ordem. Cada etapa prepara a próxima:

1. Construir a imagem Docker.
2. Criar o cluster kind.
3. Carregar a imagem nos nós.
4. Aplicar o Deployment e o Service.
5. Abrir a aplicação no navegador.

Se você estiver acompanhando a videoaula, mantenha este README aberto e use os
comandos de verificação para comparar o que está acontecendo no cluster.

## 1. Construir a imagem Docker

O `Dockerfile` usa o Nginx como servidor HTTP e copia o `index.html` para a
pasta pública do servidor.

Na raiz do repositório, execute:

```bash
docker build -t kind-lab:local .
```

Confira se a imagem existe:

```bash
docker images kind-lab
```

O nome `kind-lab:local` é apenas uma identificação local. Ele não foi enviado
para Docker Hub ou qualquer outro registry.

## 2. Criar o cluster kind

O arquivo `config.yaml` cria:

- Um node `control-plane`.
- Um mapeamento de `localhost:8080` para a porta `30080` do control-plane.

Os workers estão disponíveis como exemplo comentado no arquivo de configuração,
mas não são necessários para este laboratório.

Crie o cluster:

```bash
kind create cluster --config config.yaml --wait 5m
```

O nome definido na configuração é `kind-lab`. Se necessário, selecione
explicitamente o contexto:

```bash
kubectl config use-context kind-kind-lab
```

Confira os nodes:

```bash
kubectl get nodes
```

Todos devem aparecer com status `Ready`.

### Entendendo o mapeamento de portas

Neste lab, a porta que você acessa no navegador é diferente da porta publicada
pelo Service de propósito. Cada uma pertence a uma camada diferente da rede:

| Porta | Onde existe | Função |
| --- | --- | --- |
| `8080` | Seu computador | Porta usada pelo navegador em `localhost`. |
| `30080` | Node do kind | Porta publicada pelo Service como `NodePort`. |
| `80` | Service Kubernetes | Porta interna e estável do Service. |
| `80` | Container Nginx | Porta em que a aplicação escuta dentro do Pod. |

O caminho completo de uma requisição é:

```text
http://localhost:8080
        |
        | config.yaml: hostPort 8080
        v
Node do kind: containerPort 30080
        |
        | Service: nodePort 30080
        v
Service hello-kind: port 80
        |
        | Service: targetPort http
        v
Pod: containerPort 80
        |
        v
Nginx servindo o index.html
```

#### Por que o navegador usa a porta 8080?

O kind executa cada node como um container Docker. Por padrão, uma porta
existente dentro desse node não fica automaticamente disponível no seu
computador. O bloco abaixo, no `config.yaml`, cria uma ponte entre o host e o
container que representa o control-plane:

```yaml
extraPortMappings:
- containerPort: 30080
  hostPort: 8080
  protocol: TCP
```

Isso significa: quando o computador receber uma requisição em
`localhost:8080`, o Docker encaminha essa requisição para a porta `30080` do
node kind.

O valor `8080` foi escolhido por ser uma porta comum para desenvolvimento local
e poderia ser trocado por outra porta livre, como `8081`. Nesse caso, o acesso
seria `http://localhost:8081`, mas a porta do NodePort continuaria `30080`.

#### Definição detalhada dos campos de porta

Existem campos com nomes parecidos em arquivos diferentes. Eles pertencem a
camadas distintas da rede e não devem ser tratados como a mesma porta.

##### Campos do `config.yaml`

```yaml
extraPortMappings:
- containerPort: 30080
  hostPort: 8080
  listenAddress: "127.0.0.1"
  protocol: TCP
```

- `extraPortMappings`: solicita ao kind um encaminhamento entre o computador e
  um node do cluster.
- `containerPort: 30080`: porta dentro do container Docker que representa o
  node do kind. Neste lab, ela precisa ser igual ao `nodePort` do Service.
- `hostPort: 8080`: porta aberta no computador. E por ela que o navegador
  acessa `http://localhost:8080`.
- `listenAddress: "127.0.0.1"`: limita o acesso ao próprio computador. Com
  `0.0.0.0`, a porta pode ficar disponível em todas as interfaces de rede do
  host e exige mais cuidado.
- `protocol: TCP`: protocolo usado pelo HTTP. UDP e SCTP também existem, mas
  não são usados por esta aplicação.

Neste bloco, `containerPort` se refere ao **container do node kind**. Ele não
se refere diretamente ao Nginx nem ao Pod da aplicação.

##### Campos do `deployment.yaml`

```yaml
containers:
- name: hello-kind
  ports:
  - name: http
    containerPort: 80
```

- `containerPort: 80`: informa que o container Nginx escuta HTTP na porta 80
  dentro do Pod.
- `name: http`: cria um nome para essa porta. O Service usa esse nome em
  `targetPort: http`.

Esse `containerPort` pertence ao **container da aplicação**, e não ao node
Docker criado pelo kind. Por isso ele pode ser `80`, enquanto o
`extraPortMappings.containerPort` e `30080`.

##### Campos do `service.yaml`

```yaml
type: NodePort
ports:
- port: 80
  targetPort: http
  nodePort: 30080
```

- `port: 80`: porta estável do Service dentro do cluster. Outros Pods podem
  acessar `hello-kind:80`.
- `targetPort: http`: porta de destino nos Pods. O nome `http` aponta para o
  `containerPort: 80` definido no Deployment.
- `nodePort: 30080`: porta publicada nos nodes pelo Service. É por isso que o
  `config.yaml` encaminha o `containerPort: 30080` do node para o
  `hostPort: 8080`.
- `type: NodePort`: informa ao Kubernetes que o Service deve ser acessível por
  uma porta dos nodes.

Juntando todos os campos, o fluxo fica assim:

```text
hostPort 8080
  -> extraPortMappings.containerPort 30080
  -> Service.nodePort 30080
  -> Service.port 80
  -> Service.targetPort http
  -> Deployment containerPort 80
  -> Nginx
```

#### Por que o Service usa a porta 30080?

O tipo `NodePort` publica um Service em uma porta dos nodes do Kubernetes. Neste
lab, escolhemos explicitamente a porta `30080`:

```yaml
type: NodePort
ports:
- port: 80
  targetPort: http
  nodePort: 30080
```

A faixa normalmente usada para NodePorts é `30000` a `32767`. Por isso, `30080`
é uma porta válida e fácil de reconhecer como uma porta do Kubernetes. Ela não
precisa ser igual a porta do navegador nem a porta do container.

#### O que significam `port` e `targetPort`?

- `port: 80` é a porta estável oferecida pelo Service para outros recursos
  dentro do cluster.
- `targetPort: http` aponta para a porta nomeada `http` no Deployment.
- No Deployment, a porta `http` corresponde a `containerPort: 80`, onde o Nginx
  está escutando.
- `nodePort: 30080` é a porta aberta nos nodes para receber tráfego externo.

Em outras palavras, o Service recebe tráfego na porta `30080` do node, oferece
esse tráfego internamente na porta `80` e encaminha a requisição para a porta
`80` de um dos Pods.

As portas são diferentes porque representam pontos de entrada diferentes. Usar
o mesmo número em todas elas é possível em alguns cenários, mas não é uma
obrigação do Kubernetes e pode dificultar a visualização do caminho da
requisição.

### Se o cluster já existir

Para recriar o ambiente do zero:

```bash
kind delete cluster --name kind-lab
kind create cluster --config config.yaml --wait 5m
```

## 3. Carregar a imagem nos nodes

A imagem existe no Docker local, mas os containers do kind possuem seu próprio
ambiente de imagens. Por isso, o Kubernetes não encontra automaticamente uma
imagem criada fora do cluster.

### Entendendo o comando `kind load`

O comando `kind load` copia uma imagem para dentro dos nodes do cluster. Ele é
útil porque os nodes do kind são containers separados do ambiente Docker onde
você executou o `docker build`.

O comando possui duas formas principais:

```text
kind load docker-image
kind load image-archive
```

#### `kind load docker-image`

Carrega diretamente nos nodes uma imagem que já existe no Docker local. É a
opção usada neste laboratório:

```bash
docker build -t kind-lab:local .
kind load docker-image kind-lab:local --name kind-lab
```

Use essa opção quando você acabou de construir a imagem na mesma máquina onde
o cluster kind está rodando.

#### `kind load image-archive`

Carrega nos nodes uma imagem salva em um arquivo de archive, normalmente um
`.tar` ou `.tar.gz`. Essa opção é útil quando a imagem foi exportada, precisa
ser transferida ou não está disponível diretamente no Docker local:

```bash
kind load image-archive kind-lab.tar --name kind-lab
```

A diferença principal é a origem da imagem:

| Comando | Origem da imagem | Quando usar |
| --- | --- | --- |
| `kind load docker-image` | Imagem armazenada no Docker local | Desenvolvimento cotidiano. |
| `kind load image-archive` | Arquivo `.tar` ou `.tar.gz` | Transferência ou ambiente sem a imagem local disponível. |

Nos dois casos, o resultado é o mesmo: a imagem fica disponível dentro dos
nodes para que o Kubernetes possa iniciar os Pods sem buscar a imagem em um
registry.

Carregue a imagem em todos os nodes:

```bash
kind load docker-image kind-lab:local --name kind-lab
```

O manifest usa `imagePullPolicy: IfNotPresent`. Isso permite que o Kubernetes
use a imagem local carregada, sem tentar baixa-la de um registry.

## 4. Criar os manifestos

Os recursos foram separados em dois arquivos para deixar mais claro o papel de
cada objeto e permitir que sejam aplicados individualmente.

### 4.1 Criar o Deployment

### Deployment

O `Deployment` chamado `hello-kind` declara duas replicas da aplicação. Cada
replica roda um Pod com a imagem `kind-lab:local`.

### Service

O Service chamado `hello-kind` procura os Pods que possuem o label:

```yaml
app: hello-kind
```

Ele publica a aplicação como `NodePort` na porta `30080`.

Aplique primeiro o Deployment:

```bash
kubectl apply -f manifests/deployment.yaml
kubectl rollout status deployment/hello-kind
```

Confira os Pods criados:

```bash
kubectl get pods -o wide
```

### 4.2 Criar o Service

Agora aplique o Service separadamente:

```bash
kubectl apply -f manifests/service.yaml
```

Verifique o que foi criado:

```bash
kubectl get pods -o wide
kubectl get deployment hello-kind
kubectl get service hello-kind
```

O Service deve mostrar algo parecido com:

```text
80:30080/TCP
```

## 5. Acessar a aplicação

Abra o navegador em:

```text
http://localhost:8080
```

Também é possível testar pelo terminal:

```bash
curl http://localhost:8080
```

O caminho da requisição é:

```text
localhost:8080
  -> porta 30080 do node kind
  -> Service hello-kind:80
  -> um dos Pods do Deployment
  -> Nginx servindo index.html
```

Para observar quais Pods estão sendo selecionados pelo Service:

```bash
kubectl get endpoints hello-kind
kubectl describe service hello-kind
```

## 6. Alterar a página

Esse é o ciclo mais importante para desenvolvimento local:

1. Edite o `index.html`.
2. Gere uma nova imagem.
3. Carregue a imagem nos nodes.
4. Reinicie o Deployment.
5. Atualize o navegador.

Comandos:

```bash
docker build -t kind-lab:local .
kind load docker-image kind-lab:local --name kind-lab
kubectl rollout restart deployment/hello-kind
kubectl rollout status deployment/hello-kind
```

Depois, atualize `http://localhost:8080`.

## Troubleshooting

### O Docker não está rodando

Abra o Docker Desktop no macOS ou Windows. No Linux, confira se o servico do
  Docker está ativo. Depois, tente novamente a criação do cluster.

### A imagem não foi encontrada

Confirme se você executou os dois passos abaixo com o mesmo nome de imagem:

```bash
docker build -t kind-lab:local .
kind load docker-image kind-lab:local --name kind-lab
```

Também confira se o manifest possui `image: kind-lab:local`.

### O Pod fica em `ImagePullBackOff`

A imagem provavelmente não foi carregada nos nodes ou o nome da imagem está
diferente. Execute novamente `kind load docker-image` e confira os Pods:

```bash
kubectl describe pod -l app=hello-kind
```

### O navegador não abre `localhost:8080`

Confira se:

- O cluster está em execução.
- Os Pods estão `Running` e `Ready`.
- O Service existe e mostra a porta `80:30080/TCP`.
- O cluster foi criado usando `config.yaml`.

Comandos de diagnóstico:

```bash
kubectl get pods
kubectl get service hello-kind
kubectl get endpoints hello-kind
```

Se o cluster foi criado antes de o mapeamento de porta ser adicionado, remova
e recrie o cluster usando o arquivo atual:

```bash
kind delete cluster --name kind-lab
kind create cluster --config config.yaml --wait 5m
```

### Alterei o HTML, mas a página não mudou

O Kubernetes não percebe sozinho que uma imagem local foi reconstruida. Gere a
imagem, carregue-a novamente nos nodes e reinicie o Deployment:

```bash
docker build -t kind-lab:local .
kind load docker-image kind-lab:local --name kind-lab
kubectl rollout restart deployment/hello-kind
```

Também tente atualizar a página ignorando o cache do navegador.

## Limpeza

Para remover os recursos da aplicação:

```bash
kubectl delete -f manifests/service.yaml
kubectl delete -f manifests/deployment.yaml
```

Para remover o cluster inteiro:

```bash
kind delete cluster --name kind-lab
```

## Estrutura do projeto

```text
.
├── Dockerfile             # Imagem Nginx da aplicação
├── .dockerignore          # Arquivos fora do contexto da imagem
├── config.yaml             # Cluster kind e mapeamento de porta
├── index.html              # Página exibida no navegador
├── assets/
│   └── branding/
│       └── logo-marialazara.png # Logo exibido no rodapé
├── manifests/
│   ├── deployment.yaml     # Deployment da aplicação
│   └── service.yaml        # Service NodePort
└── README.md               # Guia do laboratório
```

## Próximos experimentos

Depois de concluir o lab, tente:

- Alterar o texto e as cores do `index.html`.
- Aumentar o número de replicas no Deployment.
- Alterar o Service para outra porta NodePort.
- Consultar os logs com `kubectl logs`.
- Observar os Pods com `kubectl get pods -w`.
- Criar um novo Deployment com outra imagem.

## Créditos

Laboratório criado por **Maria Lazara** para as aulas e conteúdos do canal:

- YouTube: [@marialazaradev](https://youtube.com/@marialazaradev)
- Repositório: `kind-local-kubernetes-lab`

Logo do kind: [kind.sigs.k8s.io](https://kind.sigs.k8s.io/).

## Referências

- [kind - Quick Start](https://kind.sigs.k8s.io/docs/user/quick-start/)
- [kind - Instalação](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)
- [kind - Carregar uma imagem no cluster](https://kind.sigs.k8s.io/docs/user/quick-start/#loading-an-image-into-your-cluster)
- [kind - Mapeamento de portas](https://kind.sigs.k8s.io/docs/user/configuration/#extra-port-mappings)
- [Kubernetes - Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Kubernetes - Services](https://kubernetes.io/docs/concepts/services-networking/service/)
