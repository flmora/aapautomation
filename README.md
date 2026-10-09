# Sandbox com OpenShift GitOps

POC de GitOps num cluster OpenShift Local (CRC). O repositório contém uma VM CirrOS e uma página web. O OpenShift GitOps acompanha a branch `main` e aplica as alterações no namespace `sandbox`.

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

A VM usa um `containerDisk` CirrOS, 1 vCPU e 128 MiB de memória. O disco não é persistente. A página web é servida por nginx a partir de um ConfigMap.

## Recriar o CRC no Windows

Os próximos comandos desta secção são para **PowerShell no Windows**. `crc delete` apaga o cluster atual e todos os recursos que estão nele. Executa-o apenas depois de guardar o que quiseres manter fora do Git. O `containerDisk` da VM não guarda dados entre recriações.

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

O `crc start` pede o pull secret da tua conta Red Hat. Os valores acima reproduzem o tamanho usado nesta POC: 6 vCPUs, 20 GiB de memória e 80 GiB de disco. O CRC tem de estar parado para mudares os recursos da instância.

Para executar a VM desta POC, expõe as extensões de virtualização à máquina `crc` no Hyper-V. Faz isto no **PowerShell como administrador**, com o CRC parado, depois de a VM `crc` ter sido criada:

```powershell
crc stop
Set-VMProcessor -VMName crc -ExposeVirtualizationExtensions $true
crc start
```

Este uso de virtualização aninhada no CRC é experimental e pode depender do hardware. Se a VM do Hyper-V tiver outro nome, confirma-o com `Get-VM` antes de executar `Set-VMProcessor`.

Ativa o cliente `oc` no PowerShell e obtém as credenciais atuais do cluster novo. Usa o login de administrador apresentado por `crc console --credentials`; a senha antiga deixa de valer depois de `crc delete`.

```powershell
& crc oc-env | Invoke-Expression
crc console --credentials
oc config use-context crc-admin
oc whoami
```

Os comandos seguintes são executados na raiz deste repositório. No WSL, se usas a função `occrc` para chamar o `oc` do Windows, troca `oc` por `occrc`. Confirma que estás ligado ao cluster novo com permissões de administrador antes de instalar os operadores.

## Instalar os operadores

Instala primeiro o OpenShift Virtualization:

```bash
oc apply -f bootstrap/virtualization-operator.yaml
oc get csv -n openshift-cnv
```

Quando o CSV do operador estiver em `Succeeded`, cria o HyperConverged e espera até que os componentes estejam prontos:

```bash
oc apply -f bootstrap/hyperconverged.yaml
oc get hyperconverged -n openshift-cnv
oc get pods -n openshift-cnv
oc describe node crc
```

No resultado do nó, confirma que `devices.kubevirt.io/kvm` tem capacidade disponível. Se esse valor for zero, a VM desta POC não conseguirá arrancar.

Instala o GitOps e cria o namespace da demonstração:

```bash
oc apply -f bootstrap/gitops-operator.yaml
oc apply -f bootstrap/namespace.yaml
oc get csv -n openshift-gitops-operator
oc get pods -n openshift-gitops
```

Espera até que o CSV do GitOps esteja em `Succeeded` e os pods de `openshift-gitops` estejam prontos. O operador cria esse namespace e a instância padrão do Argo CD.

## Ativar as Applications

As duas Applications apontam para `https://github.com/flmora/aapautomation.git`, na branch `main`. Este repositório está público. Publica primeiro os dois manifests novos de `bootstrap/` no GitHub para que a reconstrução futura parta do repositório completo. Se usares outro repositório, altera `repoURL` nos dois ficheiros de `applications/` e publica os manifests no Git antes de os aplicar.

```bash
oc apply -f applications/sandbox-vms.yaml
oc apply -f applications/sandbox-web.yaml
oc get applications -n openshift-gitops
oc get vm,vmi -n sandbox
oc get route sandbox-web -n sandbox
```

As Applications devem aparecer como `Synced` e `Healthy`. Abre a Route com `http://` para ver a página. Se uma Application falhar, consulta `oc describe application sandbox-vms -n openshift-gitops` ou usa o nome `sandbox-web`.

Se o repositório passar a ser privado, regista-o em **Settings → Repositories** no Argo CD com credenciais GitHub de leitura. O Client ID e o Client secret de uma conta de serviço Red Hat não dão acesso ao GitHub.

## Aceder à VM por SSH

Para aceder à `sandbox-vm01` a partir do próprio computador, abre um PowerShell no Windows e encaminha a porta SSH da VM para `127.0.0.1:22322`:

```powershell
$pod = oc get pod -n sandbox -l kubevirt.io/domain=sandbox-vm01 -o jsonpath='{.items[0].metadata.name}'
oc port-forward -n sandbox "pod/$pod" 22322:22
```

Deixa essa janela aberta. Noutro PowerShell, entra na VM:

```powershell
ssh -p 22322 cirros@127.0.0.1
```

A senha padrão desta imagem de teste é `gocubsgo`. O túnel termina com `Ctrl+C` na primeira janela. Se reiniciares a VM, executa o primeiro bloco novamente para obter o nome do pod atual. Esta ligação fica apenas no teu computador; não cria uma porta pública no CRC.

## Entrar no Argo CD

Obtém o endereço e a senha da conta local `admin`:

```bash
oc get route openshift-gitops-server -n openshift-gitops -o jsonpath='https://{.spec.host}{"\n"}'
oc get secret openshift-gitops-cluster -n openshift-gitops -o jsonpath='{.data.admin\.password}' | base64 -d; echo
```

Na página de entrada, usa **Username/Password** com o utilizador `admin` e a senha apresentada. Esta conta pertence ao Argo CD; `kubeadmin` é uma conta do OpenShift e não mostra automaticamente as Applications no Argo CD.

## Fazer uma alteração

Altera o HTML em `apps/web/web.yaml`, faz commit e push para `main`. A Application `sandbox-web` sincroniza o ConfigMap e a página é atualizada sem voltar a executar `oc apply`:

```bash
git add apps/web/web.yaml
git commit -m "Update sandbox page"
git push
oc get application sandbox-web -n openshift-gitops
```

O mesmo fluxo vale para `vms/vm01.yaml`. Se mudares `cores: 1` para `cores: 2`, o Argo CD atualiza a definição da VM. A instância que já está a correr mantém a CPU anterior até reiniciares a VM em **Virtualization → VirtualMachines → sandbox-vm01 → Actions → Restart**. Podes comparar os valores com:

```bash
oc get vm sandbox-vm01 -n sandbox -o jsonpath='{.spec.template.spec.domain.cpu.cores}{"\n"}'
oc get vmi sandbox-vm01 -n sandbox -o jsonpath='{.spec.domain.cpu.cores}{"\n"}'
```

As Applications têm sincronização automática, `selfHeal` e `prune`. Uma alteração manual num recurso gerido pelo Argo CD é revertida para o conteúdo do Git; a remoção de um manifesto do Git também remove o recurso correspondente do cluster.

## Acrescentar recursos ou outra Application

Um manifesto novo em `vms/` entra na Application `sandbox-vms`. Recursos relacionados com a página podem ser acrescentados em `apps/web/`. Depois de fazer commit e push, o Argo CD aplica essas alterações.

Para gerir uma pasta nova, cria um manifesto em `applications/` com outro nome e com `spec.source.path` a apontar para essa pasta. Publica ambos os ficheiros no Git e aplica a nova Application uma vez com `oc apply -f applications/NOME.yaml`. A partir daí, as alterações nessa pasta seguem o mesmo fluxo de commit e push.
