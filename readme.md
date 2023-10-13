## Kubernetes(K8S)

Orquestrador de containers, utilizando verifica se todos estão rodando(up), permite criação dinâmica de containers dada as configurações. Passou de "x"% de utilização de máquina sobe mais 2 contianers e faz o balanceamento da carga.

Recursos do K8S: - Variáveis de ambiente - Gerenciamento de senhas/secrets - Escolha de recursos computacionais - Health check - Load balancing - SSL / TLS - Domínio - Estratégia de deploy - Storage - Service Discovery com DNS.

### POD

Funcionamento do K8S: Ele envolve o container com um POD(menor unidade do kube), normalmente um pod roda um container, mas pode rodar mais de um container em situações especificas.

Criando um pod com manifesto(uma receita).

```yml
apiVersion: v1              --> versão da API do kube
kind: Pod                   --> Qual o tipo de recurso estamos criando
metadata:                   --> Alguns dados do nosso pod
  name: "nginx"                 --> Qual o nome do nosso pod
  namespace: default            --> Qual namespace ele faz parte
  labels:
    app: "nginx"                --> Label utilizada para buscar o pod em consultas
spec:                       --> Especificações do pod, como containers, volumes, restart do pod e etc.
  containers:                   --> Quais containers fazem parte desse pod
  - name: nginx                     --> Nome primeiro container
    image: "nginx:latest"           --> Qual a imagem
    ports:                      --> Configs de porta do container
    - containerPort:  80
  restartPolicy: Always
```

Para executarmos esse manifest no nosso k8s realizamos o seguinte comando: `kubectl apply -f podExample.yml`

---

### Replica Set

Ele gerencia nossos POD's, ele por exemplo fica "observando" os pods e se o pod não estiver no ar ele recria o pod e executa ele. Nele escolhemos a quantidade de pods que queremos executar. Então caso realizemos a deleção de um pod com o comando de delete, o próprio replicaset, por verificar que um dos pods que possuem a label esperada não está mais up, ele já automáticamente sobe outro.

Abaixo temos o replicaset.yml, nosso manifest de um Replica Set do k8s:

```yml
apiVersion: apps/v1           --> versão da API do kube
kind: ReplicaSet              --> Qual o tipo de recurso estamos criando
metadata:                     --> Dados do rs
  name: nginx                     --> Nome do rs
  labels:                         --> Label utilizada para buscar o rs em consultas
    app: nginx
spec:                         --> Detalhes do rs
  replicas: 3                     --> Quantidade de pods que ele vai criar/gerenciar
  selector:                       --> Seletor de pods
    matchLabels:                      --> Nome da label dos pods que ele vai gerenciar
      app: nginx
  template:                       --> Template nada mais é que a configuração dos containers que vão estar nos pods que o rs gerencia
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
          - containerPort:  80
```

Executando `kubectl apply -f replicaset.yml` temos a criação do nosso Replica Set.

Executando o comando `kubectl get rs` temos o seguinte retorno:

```
NAME    DESIRED   CURRENT   READY   AGE
nginx   3         3         3       53s
```

Listando os pods temos 3 pods rodando, se realizarmos o comando de exclusão de um deles, o rs já sobe outro.

Para acessar a aplicação que está rodando no pod precisamos executar o seguinte comando `kubectl port-forward pod/nome_pod 8080:80`

⚠Atenção!⚠ lembrando que o replica set não fica "olhando" para alterações realizadas e aplicadas no replicaset.yml, por exemplo ir lá trocar a imagem de executar o manifest novamente, ele apenas olha se a quantidade de pods está batendo com a quantidade esperada. Para poder refletir a nova atualização realizada no manifest nos pods, é necessário realizar a deleção de cada um, o que realmente não é muito prático. Para isso temos o `Deployment`.

---

### Deployment

Ele vai automátizar a criação/gerenciamento dos replicaset's de acordo com uma especificação, de modo que quando tiver alguma alteração, por exemplo, na imagem da especificação do deployment ele vai esvaziar o rs atual(jogar a zero pods) e criar um novo com os novos pods.

Abaixo temos um exemplo de yml de deployment, note que é muito parecido com o do rs:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
          - containerPort:  80
```

E executando o comando `kubectl get deployment` temos um retorno similar a:
NAME READY UP-TO-DATE AVAILABLE AGE
nginx-deployment 3/3 3 3 9s

Com isso também temos a criação de um replicaset com um nome randomico e a criação dos 3 pods. Logo entendemos o que cada um gerencia nessa estrutura do K8S, o deployment gerencia o replicaset, que por sua vez gerencia os pods, tendo assim uma estrutura similar a imagem abaixo:

![k8s-layers](https://github.com/Sandrolaxx/frostNext/assets/61207420/a88f930f-7f7b-472d-bd01-26976755c277)

---

### Service - Balanceamento de Carga

Agora temos um problema, o acesso e balanceamento de carga entre essas três replicas, vimos que podemos realizar o port-forward para ter acesso a um pod, mas como vamos acessar nossas três replicas ao mesmo tempo? Para isso temos o Service, que realiza o balanceamento da carga das nossas requests entre as nossas multiplas replicas, esse balanceamento é baseado no algoritmo de balanceamento [Round Robin](https://somosagility.com.br/balanceamento-de-carga-baseado-no-metodo-round-robin/).

Abaixo temos um exemplo de yml de service:

```yml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: default
spec:
  selector:
    app: nginx
  type: ClusterIP
  ports:
    - name: nginx-service
      port: 80
      targetPort: 80
```

Para executar `kubectl apply -f service.yml`. Vale lembrar que ele faz o balanceamento de acordo com os pods que contenham a label setada em spec -> selector -> app.

Realizando o comando `kubectl get svc` teremos de retorno:

```
NAME            TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
kubernetes      ClusterIP   10.96.0.1     <none>        443/TCP   13h
nginx-service   ClusterIP   10.98.27.57   <none>        80/TCP    14s
```

Após isso podemos agora realizar o port-forward para o serviço crido, com `kubectl port-forward svc/nginx-service 8080:80`.

Com isso temos algo como o diagrama abaixo, onde o service recebe essa request:
![k8s-service](https://github.com/Sandrolaxx/recarga-api/assets/61207420/a837502d-508d-4e09-8a60-755ebaa0da29)

Mas ok, como faço para disponibilizar minha aplicação na web? Sem ter de ficar fazendo port-forward. Precisamos apenas trocar o type do nosso service, de `ClusterIP` para `LoadBalancer`, localmente não temos como ter esse IP externo, mas em cloud providers(ASW, Oracle, Azure) ele gera um IP para poder acessar.

---

Comandos úteis:
`kubectl get nodes`: Mostra quantas máquinas temos no nosso cluster.
`kubectl apply -f podExample.yml`: Realiza a execução de um manifesto, -f de file e o arquivo do manifesto.
`kubectl get pods`: Retorna todos os pods que estão no kube. Apresenta o nome, prontidão, status, quantidade de restart, tempo de execução.
`kubectl delete pod nginx`: Remover um pod específico.
`kubectl get nodes`: Retorna todos os nossos replica set's. Apresenta o nome, qtd desejada, qtd atual, qtd prontos e tempo de execução.
`kubectl port-forward pod/nome_pod 8080:80`: Realiza um mapeamento de uma porta local para a porta do pod.

---

MiniKube:
alias kubectl="minikube kubectl --" ---> setar o comando do minikube para um alias
minikube kubectl -- get po -A ---> Acessar nosso novo cluster
