https://www.guide2wsl.com/k3s/

# Instale k3s em nossa distribuição WSL 2

## Informar a versão do K3s
```shell
K3S_VERSION="v1.23.4+k3s1"
```

## Identificação do arm 
```shell
archSuffix=""
if test "$(uname -m)" = "aarch64"
then
    archSuffix="-arm64"
fi
```

## Download do K3s
```shell
wget -q "https://github.com/k3s-io/k3s/releases/download/${K3S_VERSION}/k3s${archSuffix}" -O /usr/local/bin/k3s
```

## Permissão para o K3s
```shell
chmod u+x ~/.local/bin/k3s
```

## Exibir a versão do K3s
```shell
k3s --version
```

## Iniciar o K3s
```shell
sudo $(which k3s) server
```

## Abra uma nova aba do Wsl e execute para conferir o status
```shell
k3s check-config
```

## Caso seja necessário analisar os logs
```shell
tail -F /var/log/k3s.log
```

Assim finalizamos a instalação do K3s

# Instalação kubectl e configuração com o K3s

## Download Kubectl
```shell
wget -q https://storage.googleapis.com/kubernetes-release/release/v1.16.0/bin/linux/amd64/kubectl -O /tmp/kubectl
```

## Permissão para o Kubectl
```shell
chmod +x ./kubectl
```

## Configuração para ./kube/config
```shell
sudo $(which k3s) kubectl config rename-context default k3s
sudo $(which k3s) kubectl config view --raw > /tmp/k3s.yaml
if test -f ~/.kube/config
then
    cp -p ~/.kube/config ~/.kube/config.bak
    KUBECONFIG=~/.kube/config.bak:/tmp/k3s.yaml kubectl config view --flatten > ~/.kube/config
else
    mkdir ~/.kube
    install -m 400 /tmp/k3s.yaml ~/.kube/config
fi
```

## Baixe e execute o script de instalação do Helm e verifique se ele funciona:
```shell
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
```
```shell
helm version
```

## Kubernetes não funciona bem com nftables, que agora é o padrão no Ubuntu, então vamos reverter para o modo legado do iptables:
```shell
sudo update-alternatives --set iptables /usr/sbin/iptables-legacy
```

# Instalação Rancher no Wsl e K3s

## Criar os namespace necessários
```shell
kubectl create ns cattle-system
kubectl create ns cert-manager
```

## Instalar o cert-manager
```shell
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.2.0-alpha.2/cert-manager.crds.yaml
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager
```

## Depois check confira se os pods estão rodando
```shell
k3s kubectl get pods -n cert-manager
```

O Rancher obriga que tenhamos um certificado para aplicação, neste caso, usaremos a função de certificado autoassinado do Rancher, da qual seu navegador reclamará, mas poderá ser testado.

## Instalação do Rancher
```shell
helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname=rancher.local
```

Será instalado a última versão usando Helm.
Estamos definido o hostname rancher.local, que mais tarde, deveremos adicionar no /etc/hosts

## Aguardando o deployment
```shell
kubectl -n cattle-system get deploy rancher -w
```

Acredite, dependendo da configuração da sua máquina, pode demorar bastante

## Configuração de rede
```shell
echo "127.0.0.1 rancher.local" >> /etc/hosts
```

Agora adicionamos o host tanto no WSL e no Windows, para que funcione perfeitamente.

```shell
ip addr show eth0 | grep -oP '(?<=inet\s)\d+(\.\d+){3}'
```

Obtenha seu endereço IP de distribuição WSL 2. Isso é redefinido toda vez que você reinicia o contêiner, portanto, a próxima etapa precisará ser repetida sempre que sua distribuição WSL for reiniciada.

Abra agora o hosts no Windows e também adicione o host.
<WSL 2 IP address> rancher.local

```shell
powershell.exe Start-Process notepad.exe 'c:\Windows\System32\Drivers\etc\hosts' -Verb runAs
```

## Conclusão
Agora você deve conseguir abrir rancher.local em um navegador Windows (ou WSL) e configurar.
