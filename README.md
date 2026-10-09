# Sandbox com OpenShift GitOps

Nesta POC, uso GitOps num cluster OpenShift Local (CRC) para gerir uma VM CirrOS e uma página web. O OpenShift GitOps acompanha a branch `main` do meu repositório e aplica as alterações no namespace `sandbox`.

## O que está no repositório

| Caminho | Função |
| --- | --- |
| `bootstrap/namespace.yaml` | Cria `sandbox` e permite que o Argo CD faça a gestão dos recursos. |
| `bootstrap/gitops-operator.yaml` | Instala o operador OpenShift GitOps. |
| `bootstrap/virtualization-operator.yaml` | Instala o operador OpenShift Virtualization. |
| `bootstrap/hyperconverged.yaml` | Ativa o OpenShift Virtualization depois da instalação do operador. |
| `vms/vm01.yaml` | Define a VM `sandbox-vm01`. |
| `apps/web/web.yaml` | Define a página, o Deployment, o Service e a Route `sandbox-web`. |
| `applications/` | Contém as Applications `sandbox-vms` e `sandbox-web`. |

A VM que criei usa um `containerDisk` CirrOS, 1 vCPU e 128 MiB de memória. O disco não é persistente. Sirvo a página web com nginx a partir de um ConfigMap.

## Recriar o CRC no Windows

Uso **PowerShell no Windows** para recriar o CRC. Antes de executar `crc delete`, guardo fora do cluster qualquer dado que queira conservar: esse comando apaga o cluster e todos os recursos nele. O `containerDisk` da VM não guarda dados entre recriações.

```powershell
crc stop
crc delete
crc config set preset openshift
crc config set cpus 6
crc config set memory 20480
crc config set disk-size 80
crc setup
crc start
```

O `crc start` pede o pull secret da minha conta Red Hat. Os valores acima reproduzem o tamanho que usei nesta POC: 6 vCPUs, 20 GiB de memória e 80 GiB de disco. Para mudar os recursos da instância, paro o CRC primeiro.

Para executar a VM da POC, exponho as extensões de virtualização à máquina `crc` no Hyper-V. Faço isto no **PowerShell como administrador**, com o CRC parado e depois de a máquina `crc` ter sido criada:

```powershell
crc stop
Set-VMProcessor -VMName crc -ExposeVirtualizationExtensions $true
crc start
```

Este uso de virtualização aninhada no CRC é experimental e pode depender do hardware. Se o nome da máquina no Hyper-V for diferente de `crc`, consulto-o com `Get-VM` antes de executar `Set-VMProcessor`.

Depois ativo o cliente `oc` no PowerShell e consulto as credenciais do cluster novo. Uso o contexto de administrador criado pelo CRC; a senha do cluster anterior deixa de valer depois de `crc delete`.

```powershell
& crc oc-env | Invoke-Expression
crc console --credentials
oc config use-context crc-admin
oc whoami
```

Executo os comandos seguintes na raiz deste repositório. No WSL, uso a função `occrc` no lugar de `oc` para chamar o cliente do Windows. Antes de instalar os operadores, confirmo com `oc whoami` que estou ligado ao cluster novo como administrador.

## Instalar os operadores

Começo pelo OpenShift Virtualization:

```bash
oc apply -f bootstrap/virtualization-operator.yaml
oc get csv -n openshift-cnv
```

Quando o CSV do operador chega a `Succeeded`, crio o HyperConverged e espero que os componentes fiquem prontos:

```bash
oc apply -f bootstrap/hyperconverged.yaml
oc get hyperconverged -n openshift-cnv
oc get pods -n openshift-cnv
oc describe node crc
```

No resultado do nó, confirmo que `devices.kubevirt.io/kvm` tem capacidade disponível. Se o valor for zero, a VM não consegue arrancar.

Depois instalo o GitOps e crio o namespace da demonstração:

```bash
oc apply -f bootstrap/gitops-operator.yaml
oc apply -f bootstrap/namespace.yaml
oc get csv -n openshift-gitops-operator
oc get pods -n openshift-gitops
```

Espero até que o CSV do GitOps esteja em `Succeeded` e os pods de `openshift-gitops` estejam prontos. O operador cria esse namespace e a instância padrão do Argo CD.

## Ativar as Applications

As duas Applications que criei apontam para `https://github.com/flmora/aapautomation.git`, na branch `main`. O repositório está público. Se mudar de repositório, altero `repoURL` nos dois ficheiros de `applications/` e publico os manifests no Git antes de os aplicar.

```bash
oc apply -f applications/sandbox-vms.yaml
oc apply -f applications/sandbox-web.yaml
oc get applications -n openshift-gitops
oc get vm,vmi -n sandbox
oc get route sandbox-web -n sandbox
```

Espero ver as Applications como `Synced` e `Healthy`. Abro a Route com `http://` para ver a página. Se uma Application falhar, consulto `oc describe application sandbox-vms -n openshift-gitops` ou substituo o nome por `sandbox-web`.

Se tornar o repositório privado, registo-o em **Settings → Repositories** no Argo CD com credenciais GitHub de leitura. O Client ID e o Client secret da minha conta de serviço Red Hat não dão acesso ao GitHub.

## Aceder à VM por SSH

Para aceder à `sandbox-vm01` a partir do meu computador, abro um PowerShell no Windows e encaminho a porta SSH da VM para `127.0.0.1:22322`:

```powershell
$pod = oc get pod -n sandbox -l kubevirt.io/domain=sandbox-vm01 -o jsonpath='{.items[0].metadata.name}'
oc port-forward -n sandbox "pod/$pod" 22322:22
```

Mantenho essa janela aberta. Noutro PowerShell, entro na VM:

```powershell
ssh -p 22322 cirros@127.0.0.1
```

A senha padrão desta imagem de teste é `gocubsgo`. Para terminar o túnel, uso `Ctrl+C` na primeira janela. Se reiniciar a VM, executo o primeiro bloco novamente para obter o nome do pod atual. A ligação fica apenas no meu computador; não crio uma porta pública no CRC.

### Se o cluster estiver remoto

Num cluster OpenShift remoto, uso o mesmo método se a minha rede conseguir chegar à API e a minha conta tiver permissões para aceder à VM. Faço `oc login` nesse cluster e abro o túnel no meu terminal:

```bash
pod=$(oc get pod -n sandbox -l kubevirt.io/domain=sandbox-vm01 -o jsonpath='{.items[0].metadata.name}')
oc port-forward -n sandbox "pod/$pod" 22322:22
```

Noutro terminal, executo `ssh -p 22322 cirros@127.0.0.1`. A ligação SSH passa pela API do OpenShift; não preciso de acesso direto ao IP da VM. Uso este método para acesso pontual e mantenho o `port-forward` aberto durante a sessão.

Se precisar de acesso regular pela rede, posso associar a VM a um `Service` do tipo `NodePort` ou `LoadBalancer` interno. O `NodePort` exige acesso de rede aos nós; o `LoadBalancer` depende da infraestrutura do cluster. Antes de expor o SSH dessa forma, configuro uma chave pública na VM e deixo de usar a senha padrão da CirrOS. As alternativas estão descritas na [documentação do OpenShift Virtualization](https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html/virtualization/managing-vms).

## Entrar no Argo CD

Obtenho o endereço do Argo CD e a senha da conta local `admin`:

```bash
oc get route openshift-gitops-server -n openshift-gitops -o jsonpath='https://{.spec.host}{"\n"}'
oc get secret openshift-gitops-cluster -n openshift-gitops -o jsonpath='{.data.admin\.password}' | base64 -d; echo
```

Na página de entrada, escolho **Username/Password** e uso o utilizador `admin` com a senha apresentada. Esta conta pertence ao Argo CD; a conta `kubeadmin` do OpenShift não mostra automaticamente as Applications no Argo CD.

## Fazer uma alteração

Quando altero o HTML em `apps/web/web.yaml`, faço commit e push para `main`. A Application `sandbox-web` sincroniza o ConfigMap e a página é atualizada sem eu voltar a executar `oc apply`:

```bash
git add apps/web/web.yaml
git commit -m "Update sandbox page"
git push
oc get application sandbox-web -n openshift-gitops
```

O mesmo fluxo vale para `vms/vm01.yaml`. Se mudar `cores: 1` para `cores: 2`, o Argo CD atualiza a definição da VM. A instância em execução mantém a CPU anterior até eu reiniciar a VM em **Virtualization → VirtualMachines → sandbox-vm01 → Actions → Restart**. Comparo os valores com:

```bash
oc get vm sandbox-vm01 -n sandbox -o jsonpath='{.spec.template.spec.domain.cpu.cores}{"\n"}'
oc get vmi sandbox-vm01 -n sandbox -o jsonpath='{.spec.domain.cpu.cores}{"\n"}'
```

Configurei as Applications com sincronização automática, `selfHeal` e `prune`. Se eu alterar manualmente um recurso gerido pelo Argo CD, ele repõe o conteúdo do Git. Se remover um manifesto do Git, o recurso correspondente também é removido do cluster.

## Acrescentar recursos ou outra Application

Quando acrescento um manifesto em `vms/`, ele entra na Application `sandbox-vms`. Coloco os recursos da página em `apps/web/`. Depois de fazer commit e push, o Argo CD aplica as alterações.

Para gerir uma pasta nova, crio um manifesto em `applications/` com outro nome e com `spec.source.path` a apontar para essa pasta. Publico os ficheiros no Git e aplico a nova Application uma vez com `oc apply -f applications/NOME.yaml`. A partir daí, as alterações dessa pasta seguem o mesmo fluxo de commit e push.
