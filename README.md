# VM de sandbox com OpenShift GitOps

POC para gerir uma VM no OpenShift Virtualization a partir de um repositório Git. Assume que o cluster está acessível com `oc` e que OpenShift Virtualization e OpenShift GitOps já estão instalados.

## 1. Confirmar o ambiente

```bash
oc whoami
oc get hyperconverged -A
oc get pods -n openshift-cnv
oc get pods -n openshift-gitops
```

## 2. Criar a VM

```bash
oc apply -f bootstrap/namespace.yaml
oc apply -f vms/vm01.yaml
oc get vm,vmi -n sandbox
```

O namespace inclui a label que permite ao OpenShift GitOps gerir os seus recursos. A VM usa um `containerDisk` Fedora, sem armazenamento persistente.

## 3. Publicar os manifests

Configura o remoto `origin` com o URL do teu repositório e substitui `repoURL` em `applications/sandbox-vms.yaml` pelo mesmo URL. Depois:

```bash
git add .gitignore README.md bootstrap vms applications
git commit -m "Add sandbox VM"
git branch -M main
git push -u origin main
```

Se o repositório for privado, configura o acesso no Argo CD antes de criar a Application.

## 4. Ativar o GitOps

```bash
oc apply -f applications/sandbox-vms.yaml
oc get application sandbox-vms -n openshift-gitops
oc get vm,vmi -n sandbox
```

A Application deverá ficar `Synced` e `Healthy`. Em caso de erro, consulta `oc describe application sandbox-vms -n openshift-gitops`.

## 5. Testar sincronização e reconciliação

Altera `cores: 2` para `cores: 4` em `vms/vm01.yaml`, faz commit e push. Confirma a alteração no cluster:

```bash
git add vms/vm01.yaml
git commit -m "Increase VM to 4 CPUs"
git push
oc get vm sandbox-vm01 -n sandbox -o jsonpath='{.spec.template.spec.domain.cpu.cores}{"\n"}'
```

Para testar `selfHeal`, muda temporariamente o valor no cluster e volta a consultá-lo após a reconciliação:

```bash
oc patch vm sandbox-vm01 -n sandbox --type=json -p='[{"op":"replace","path":"/spec/template/spec/domain/cpu/cores","value":2}]'
oc get vm sandbox-vm01 -n sandbox -o jsonpath='{.spec.template.spec.domain.cpu.cores}{"\n"}'
```

O valor deve voltar a `4`. A VMI em execução pode precisar de reinício para utilizar a nova configuração de CPU.
