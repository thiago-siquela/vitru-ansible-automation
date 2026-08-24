# Vitru Ansible Automation

Automação integrada ao AWX para inventário e diagnóstico de servidores.

## Escopo atual

Esta primeira entrega lista, a partir do inventário já carregado no AWX,
somente VMs que:

1. pertençam ao grupo de energia configurado;
2. pertençam a pelo menos um grupo Linux permitido;
3. não sejam templates, quando essa regra estiver habilitada;
4. possuam IP, quando essa regra estiver habilitada.

O playbook não acessa as VMs. Todas as tarefas são executadas uma única vez e
delegadas para `localhost`, utilizando apenas `groups` e `hostvars` do
inventário.

## Configuração

A política não fica neste repositório. Ela deve existir nas variáveis do
inventário do AWX sob a chave `vm_inventory_policy`.

## Playbook

```text
playbooks/vmware/list_powered_vms.yaml
```

## Validação local

```bash
ansible-playbook \
  -i tests/inventory.yml \
  playbooks/vmware/list_powered_vms.yaml \
  --syntax-check

ansible-playbook \
  -i tests/inventory.yml \
  playbooks/vmware/list_powered_vms.yaml
```

## Documentação

- [Configuração do primeiro Job Template](docs/awx-first-job.md)
- [Contrato de saída](docs/output-schema.md)
