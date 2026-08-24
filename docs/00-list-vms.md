# Listagem de VMs Linux ligadas

## Descrição

O playbook `playbooks/vmware/00-list-vms.yaml` gera um relatório CSV com as
máquinas Linux ligadas presentes em um inventário do AWX.

Embora os dados utilizados atualmente sejam fornecidos por uma origem VMware
vCenter, o playbook consulta somente os hosts, grupos e variáveis já carregados
no inventário do AWX. Por isso, ele também pode trabalhar com outras origens de
inventário que forneçam os mesmos grupos e atributos.

Uma máquina é incluída no relatório quando:

- pertence ao grupo configurado em `powered_on_group`;
- pertence a pelo menos um dos grupos configurados em `linux_groups`;
- não é um template, quando `exclude_templates` estiver habilitado;
- possui endereço IP, quando `require_ip` estiver habilitado.

O playbook executa somente em `localhost` e não estabelece conexão SSH com as
máquinas inventariadas.

## Pré-requisitos

- Projeto sincronizado no AWX com este repositório.
- Inventário previamente sincronizado e contendo os grupos de estado e sistema
  operacional.
- Execution Environment com `ansible-core` e o comando `wc`.
- Variáveis de política configuradas em **Inventories > Variables**.

Exemplo:

```yaml
---
vm_inventory_policy:
  brand: nome-da-marca
  powered_on_group: poweredOn
  linux_groups:
    - grupo-linux-01
    - grupo-linux-02
  exclude_templates: true
  require_ip: true
```

### Legenda das variáveis

| Variável | Obrigatória | Padrão | Descrição |
| --- | --- | --- | --- |
| `vm_inventory_policy.brand` | Não | Vazio | Marca associada ao inventário e gravada na coluna `marca`. |
| `vm_inventory_policy.powered_on_group` | Sim | — | Grupo que identifica as máquinas ligadas. |
| `vm_inventory_policy.linux_groups` | Sim | — | Lista de grupos que identificam sistemas Linux. |
| `vm_inventory_policy.exclude_templates` | Não | `true` | Exclui objetos marcados como template. |
| `vm_inventory_policy.require_ip` | Não | `true` | Exclui máquinas sem endereço IP. |
| `csv_batch_size` | Não | `200` | Quantidade de candidatos processados em cada fragmento do CSV. |

> **Atenção:** os grupos precisam existir no inventário do AWX. Na integração
> atual com VMware eles são fornecidos pela origem dinâmica. Para outras
> origens, é necessário criar grupos equivalentes ou ajustar a política do
> inventário.

## Configuração sugerida do Job Template

| Campo | Configuração |
| --- | --- |
| Job Type | `Run` |
| Inventory | Inventário que será analisado |
| Project | Projeto associado a este repositório |
| Playbook | `playbooks/vmware/00-list-vms.yaml` |
| Credentials | Nenhuma credencial de máquina é necessária |
| Limit | Opcional; pode ser habilitado como *Prompt on launch* |
| Variables | `csv_batch_size` pode ser informado para alterar o tamanho do lote |

O inventário pode ser configurado como *Prompt on launch*, permitindo reutilizar
o mesmo Job Template em ambientes diferentes. Cada inventário deve possuir sua
própria `vm_inventory_policy`.

## Processamento

1. Valida a política configurada no inventário.
2. Reúne os hosts pertencentes aos grupos Linux.
3. Mantém apenas os hosts que também pertencem ao grupo de máquinas ligadas.
4. Divide os candidatos em lotes, com 200 hosts por padrão.
5. Gera um fragmento CSV para cada lote.
6. Consolida os fragmentos em um arquivo CSV temporário.
7. Publica no AWX somente os dados resumidos da execução.

## Saída CSV

O arquivo utiliza ponto e vírgula (`;`) como delimitador e contém apenas uma
linha de cabeçalho seguida pelas máquinas selecionadas.

Cabeçalho gerado:

```text
marca;inventario;inventory_hostname;vm_name;hostname;ip;guest_id;instance_uuid;grupos_linux
```

### Legenda das colunas

| Coluna | Descrição | Origem no inventário |
| --- | --- | --- |
| `marca` | Marca associada ao ambiente inventariado. | `vm_inventory_policy.brand` |
| `inventario` | Nome do inventário utilizado na execução. | `awx_inventory_name` |
| `inventory_hostname` | Identificador completo do host dentro do inventário do AWX. | Nome do host no inventário |
| `vm_name` | Nome da máquina virtual. | `config.name` |
| `hostname` | Nome de host informado pelo sistema convidado. | `guest.hostName` |
| `ip` | Endereço IP principal conhecido pelo inventário. | `guest.ipAddress` ou `ansible_host` |
| `guest_id` | Identificador técnico do sistema operacional convidado. | `guest.guestId` ou `config.guestId` |
| `instance_uuid` | UUID da instância virtual. | `config.instanceUuid` |
| `grupos_linux` | Grupos Linux da política aos quais a máquina pertence. | Interseção entre `group_names` e `linux_groups` |

## Resumo publicado no AWX

O `set_stats` não armazena todas as máquinas. Somente os seguintes dados
resumidos são publicados como artefatos do job:

| Artefato | Descrição |
| --- | --- |
| `selected_vm_count` | Quantidade de máquinas gravadas no CSV. |
| `csv_batch_size` | Tamanho do lote utilizado durante o processamento. |
| `csv_batch_count` | Quantidade de lotes gerados. |
| `csv_filename` | Nome atribuído ao arquivo temporário. |

## Como recuperar o CSV no AWX

O arquivo é criado dentro do Execution Environment em um caminho temporário
semelhante a:

```text
/tmp/awx_vm_inventory_<sufixo>/linux_powered_vms_<job_id>.csv
```

Esse caminho não deve ser considerado persistente. Após o encerramento do
Execution Environment, o arquivo pode deixar de existir.

O conteúdo do CSV fica registrado no resultado da tarefa
`Exibir CSV no output do AWX`. Para localizar o evento pela API:

```text
/api/v2/jobs/<JOB_ID>/job_events/?task__icontains=Exibir%20CSV
```

No evento do tipo `runner_on_ok`, o conteúdo está em:

```text
event_data.res.stdout
```

Exemplo de extração com um token pessoal do AWX:

```bash
curl -sS \
  -H "Authorization: Bearer ${AWX_TOKEN}" \
  "https://awx.exemplo/api/v2/job_events/<EVENT_ID>/" \
  | jq -r '.event_data.res.stdout' \
  > "linux_powered_vms_<JOB_ID>.csv"
```

O botão **Download Output** baixa o log completo do job, não um anexo CSV
separado. Para retenção permanente e download direto, será necessário enviar o
arquivo para um armazenamento externo ou disponibilizar um volume persistente
ao Execution Environment.

## Observações

- Se `brand` não estiver configurada, a coluna `marca` será gerada vazia.
- Se um grupo listado em `linux_groups` não existir, ele será ignorado.
- O valor de `guest_id` é técnico, por exemplo um identificador fornecido pelo
  VMware, e pode precisar de normalização antes de ser usado em um inventário
  executivo.
- Diminuir `csv_batch_size` reduz o volume processado por fragmento, mas aumenta
  a quantidade de lotes.
- O filtro `Limit` do Job Template é aplicado antes da lógica do playbook e pode
  ser utilizado para testes com uma parte menor do inventário.
