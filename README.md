# kube-study
Para usar esse repositio voce precisa ter o VirtualBox Instalado em sua maquina e possui no minimo 8Gb de mem para uso e 60Gb de espaço livre.
## Criar Cluster
Para poder praticar voce precisara criar 3 VM's sendo 1 Master e 2 Worker para isso basta executar o script vagrant dentro da pasta **template** que ele irá criar as 3 VM's necessárias
Segue um Cheat Sheet dos principais comandos do vagrant para usar nesse repo:
|   Command  | Description             |
|:----------:|-------------------------|
|     up     | Criar ou inicia as vm's |
| ssh <:name> | Realiza o ssh          |
|    halt    | Desliga as vm's         |
|   destroy  | Remove a vm             |
Ao executar o comando **up** serão criada as 3 VM's abaixo

|  VM name  | S.O.         	| IP         	|
|:------:	|--------------	|------------	|
| master 	| Ubuntu 20.04 	| 10.17.1.10 	|
|  node1 	| Ubuntu 20.04 	| 10.17.1.11 	|
|  node2 	| ubuntu 20.04 	| 10.17.1.12 	|
## Container Runtime
Eu particularmente prefiro usar o **containerd** como runtime do cluster mas fica a seu critério usar o CRI-O ou o Docker *(lembrando que o docker deixara de ser suportado)* faça isso em todas as vm's
1. Acesse as vms e entre com o usuario **wb**
````
sudo su - wb
````
2. Vamos criar e carregar as confgs que o **containerd** utiliza:
````
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
````
 - Carregue a confg
````
sudo modprobe overlay
sudo modprobe br_netfilter
````
- Adicione os parametros SYSCTL na inicializacao do sistema
````
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf  
net.bridge.bridge-nf-call-iptables = 1  
net.ipv4.ip_forward = 1  
net.bridge.bridge-nf-call-ip6tables = 1  
EOF
````
- Reinicie o sistema
````
sudo reboot
````
3. Instale o **containerd**
````
sudo apt update && sudo apt install -y containerd
````
4. Crie uma pasta para o **containerd** por padrao eu crio a pasta no /etc mas pode ser em qualquer lugar (por tanto que nao seja efemero)
````
sudo mkdir -p /etc/containerd
````
5. Defina o **containerd** como runtime padrao do sistema
````
sudo containerd config default | sudo tee /etc/containerd/config.toml
````
6. Inicie o serviço
````
sudo systemctl enable containerd
sudo systemctl start containerd
````
## S.O.
Para que o Cluster funcione bem, precisamos mudar algumas coisas a nivel de S.O. 
- SWAP
````
sudo sed -Ei 's/(.*swap.*)/#\1/g' /etc/fstab
sudo swapoff -a
````
- IPTABLES
````
sudo update-alternatives --set iptables /usr/sbin/iptables-legacy
sudo update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy
sudo update-alternatives --set arptables /usr/sbin/arptables-legacy
sudo update-alternatives --set ebtables /usr/sbin/ebtables-legacy
````
- Firewall
````
sudo ufw status 

Se estiver habilitado desabilite
````
## Kubernetes
Irei usar a versao 1.19.3 pois mais para frente irei mostar como atualizar a versao **MAJOR** e **MINOR** do Cluster
1. Baixe a chave publica do repo kube
````
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
````
2. Configure um repo instavel do Ubuntu
````
cat << EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list  
deb https://apt.kubernetes.io/ kubernetes-xenial main  
EOF
````
3. Faça um update
````
sudo apt update
````
4. Instale as ferramentas do Kubernetes
````
sudo apt install -y kubelet=1.19.3-00 kubeadm=1.19.3-00 kubectl=1.19.3-00
````
5. Mark os pacotes para nao atualizarem automaticamente ***(ISSO E MUITO IMPORTANTE)***
````
sudo apt-mark hold kubelet kubeadm kubectl
````
6. Como estou usando o **containerd** precisa adicionar o SOCK
````
sudo vim /etc/systemd/system/kubelet.service.d/0-containerd.conf  
[Service]  
Environment="KUBELET_EXTRA_ARGS=--container-runtime=remote --runtime-request-timeout=15m --container-runtime-endpoint=unix:///run/containerd/containerd.sock"
````
7. Reinicie o daemon e o kubelet
````
sudo systemctl daemon-reload
sudo systemctl restart kubelet
````
8. Reinicie a VM
````
sudo reboot
````
## Iniciando o Kubernetes 
Após a instalação e configurar de todos os pre-reqs, vamos instalar a imagem do Kubernetes, ***utilize apenas no Master!***
1. Use o comando abaixo para baixar a imagem do Kubernetes na versão 1.19.3
````
sudo kubeadm config images pull --kubernetes-version=1.19.3
````
2. Configure o kubeadmin
````
sudo kubeadm init --pod-network-cidr=192.168.0.0/16 \
--apiserver-advertise-address=10.17.1.10 --kubernetes-version=1.19.3 \
--ignore-preflight-errors=all
````
Essa é a saida do comando
````
[init] Using Kubernetes version: v1.19.3
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your
internet connection
[preflight] You can also perform this action in beforehand using ’kubeadm
config images pull’
[kubelet-start] Writing kubelet environment file with flags to file "/var/
lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/
config.yaml"
[kubelet-start] Activating the kubelet service
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [master
kubernetes kubernetes.default kubernetes.default.svc kubernetes.
default.svc.cluster.local] and IPs [10.96.0.1 172.16.1.100]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
````
3. Crie uma pasta **kube** na raiz para acessar as configs do cluster de forma rapida
````
mkdir -p $HOME/.kube
````
4. Copie o arquivo de configuração para a pasta criada
````
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
````
5. Arrume as permissões de user e group
````
sudo chown $(id -u):$(id -g) $HOME/.kube/config
````
6. Para que a comunicacao com o cluster funcione precisamos instalar um plugin de rede (de acordo com a doc do kube nao ha um plugin especifico que seja "melhor") aqui usarei o calico

 Use o manifesto disponibilizado pela project calico para criar a arquitetura necessaria
````
kubectl apply -f https://docs.projectcalico.org/v3.9/manifests/calico.yaml
````
7. Veja se foi criado
````
kubectl get pods -n kube-system
````
Caso o pod valico-node-XPTO esteja dando restart execute o comando abaixo para adicionar uma var de ambiente para seu daemonset
````
kubectl -n kube-system set env daemonset/calico-node FELIX_IGNORELOOSERPF=true

````
Veja se os pods estão com status de  ***running***
8. Veja agora se o master esta ***ready*** 
````
kubectl get nodes 
````
## Tokens de acesso do Cluster
Como descrito no inicio desse documento, temos 3 vm's para o nosso cluster, agora iremos criar e gerenciar os tokens de acesso para que todas as vms possam se comunicar como um cluster
1. Crie um token com tempo de expiracao de 10 minutos e saida do comando para adicionar os nodes workers no cluster
````
sudo kubeadm token create --print-join-command --ttl 10m --description="Cluster
Kubernetes
````
2. Listar os tokens
````
sudo kubeadm token list
````

3. Ingressando node no cluster
````
sudo kubeadm join 172.16.1.100:6443 --token <TOKEN_GERADO> \
--discovery-token-ca-cert-hash sha256:<DISCOVERY_TOKEN_GERADO> --ignore
-preflight-errors=all
````
Basta copiar a saida do comando no passo 1 e adicionar a flag --ignore
-preflight-errors=all

## Tks
Prontinho :D o Cluster foi instalado e configurado 

Agora da uma olhada nas pastas chamadas 'labs' e vai seguindo o passo a passo la que eh sucesso!

Qualquer coisa da um ping nas issues pois se eu puder ajudar estarei a disposicao :D

# Wallace Bruno Gentil