# VM de sandbox com OpenShift GitOps

POC para gerir uma VM no OpenShift Virtualization a partir de um repositório Git. Assume que o cluster está acessível com `oc` e que OpenShift Virtualization já está instalado.

## 1. Confirmar o ambiente

```bash
oc whoami
oc get hyperconverged -A
oc get pods -n openshift-cnv
```

## 2. Criar a VM

```bash
oc apply -f bootstrap/namespace.yaml
oc apply -f vms/vm01.yaml
oc get vm,vmi -n sandbox
```

O namespace inclui a label que permite ao OpenShift GitOps gerir os seus recursos. A VM usa um `containerDisk` CirrOS, 2 vCPUs e 128 MiB de memória, sem armazenamento persistente.

## 3. Publicar os manifests

Configura o remoto `origin` com o URL do teu repositório e substitui `repoURL` em `applications/sandbox-vms.yaml` pelo mesmo URL. Depois:

```bash
git add .gitignore README.md bootstrap vms applications
git commit -m "Add sandbox VM"
git branch -M main
git push -u origin main
```

## 4. Instalar o GitOps

Se o operador ainda não estiver instalado, aplica a subscrição e espera pela instância padrão do Argo CD:

```bash
oc apply -f bootstrap/gitops-operator.yaml
oc get csv -n openshift-gitops-operator
oc get pods -n openshift-gitops
```

O namespace `openshift-gitops` é criado pelo operador. Aguarda até que os pods estejam prontos antes de criar a Application.
No CRC, os pods podem ficar `Pending` por falta de CPU; este laboratório precisou de 6 vCPUs.

Para abrir o Argo CD e obter a senha inicial:

```bash
oc get route openshift-gitops-server -n openshift-gitops -o jsonpath='https://{.spec.host}{"\n"}'
oc get secret openshift-gitops-cluster -n openshift-gitops -o jsonpath='{.data.admin\.password}' | base64 -d; echo
```

Abre o endereço apresentado e entra nos campos **Username/Password** com o utilizador `admin` e a senha do segundo comando. Esta é a conta local do Argo CD, diferente do `kubeadmin` do OpenShift; não uses **Log in via OpenShift**. No WSL, se usas a função `occrc` para aceder ao CRC, substitui `oc` por `occrc` nestes comandos.

Se o repositório for privado, abre a interface do Argo CD (`oc get route openshift-gitops-server -n openshift-gitops`) e adiciona o repositório em **Settings → Repositories**, com um token GitHub de leitura. Faz isso antes de criar a Application. O Client ID e o Client secret da Red Hat não dão acesso ao GitHub.

## 5. Ativar o GitOps

```bash
oc apply -f applications/sandbox-vms.yaml
oc get application sandbox-vms -n openshift-gitops
oc get vm,vmi -n sandbox
```

A Application deverá ficar `Synced` e `Healthy`. Em caso de erro, consulta `oc describe application sandbox-vms -n openshift-gitops`.

## 6. Testar sincronização e reconciliação

Altera `cores: 2` para `cores: 1` em `vms/vm01.yaml`, faz commit e push. Confirma a alteração no cluster:

```bash
git add vms/vm01.yaml
git commit -m "Set VM to 1 CPU"
git push
oc get vm sandbox-vm01 -n sandbox -o jsonpath='{.spec.template.spec.domain.cpu.cores}{"\n"}'
```

A VM fica com a configuração nova, mas a instância em execução conserva a CPU antiga até ser reiniciada. Reinicia-a na consola do OpenShift em **Virtualization → VirtualMachines → sandbox-vm01 → Actions → Restart**. Depois, confirma com `oc get vmi sandbox-vm01 -n sandbox -o jsonpath='{.spec.domain.cpu.cores}{"\n"}'`.

Para testar `selfHeal`, muda temporariamente o valor no cluster e volta a consultá-lo após a reconciliação:

```bash
oc patch vm sandbox-vm01 -n sandbox --type=json -p='[{"op":"replace","path":"/spec/template/spec/domain/cpu/cores","value":2}]'
oc get vm sandbox-vm01 -n sandbox -o jsonpath='{.spec.template.spec.domain.cpu.cores}{"\n"}'
```

O valor deve voltar a `1`.
