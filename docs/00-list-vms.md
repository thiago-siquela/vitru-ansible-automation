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

Nesta primeira etapa do inventário, o relatório contém somente as informações
que já estão disponíveis diretamente no inventário dinâmico e são utilizadas
na planilha: identificação da máquina, hostname e endereço IP. Os demais campos
da planilha serão preenchidos posteriormente por playbooks e integrações
específicos.

Cabeçalho gerado:

```text
id_da_maquina;hostname;ip
```

### Legenda das colunas

| Coluna | Descrição | Origem no inventário |
| --- | --- | --- |
| `id_da_maquina` | Identificação da máquina usada na coluna `ID da máquina` da planilha. Se `config.name` não existir, utiliza o nome do host no inventário. | `config.name` ou nome do host no inventário |
| `hostname` | Nome de host informado pelo sistema convidado. | `guest.hostName` |
| `ip` | Endereço IP principal conhecido pelo inventário. | `guest.ipAddress` ou `ansible_host` |

### Campos da planilha fora do escopo deste playbook

Os campos abaixo não são exportados pelo playbook atual:

- Marca;
- Ambiente;
- FQDN;
- Localização/datacenter/cloud;
- Sistema operacional e versão;
- Situação da integração com o Active Directory;
- Origem da elevação de privilégios;
- Data da coleta.

Essas informações exigem regras organizacionais, coleta dentro do sistema
operacional ou consulta a outras fontes. Elas devem ser tratadas em playbooks
ou integrações independentes, evitando misturar responsabilidades nesta
primeira coleta.

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

- Se um grupo listado em `linux_groups` não existir, ele será ignorado.
- Diminuir `csv_batch_size` reduz o volume processado por fragmento, mas aumenta
  a quantidade de lotes.
- O filtro `Limit` do Job Template é aplicado antes da lógica do playbook e pode
  ser utilizado para testes com uma parte menor do inventário.
