# AAP 2.6 Containerized — Automation Hub com S3 via IAM Role

**Workaround validado para instalação do Automation Hub utilizando IAM Role da EC2 (sem credenciais estáticas).**

Data: 2026-04-10
Versão alvo: Ansible Automation Platform 2.6 Containerized (bundle)
Topologia: Container Growth (multi-host)

---

## 1. Contexto

O Automation Hub do Ansible Automation Platform 2.6 Containerized suporta oficialmente o uso de Amazon S3 como backend de storage (`hub_storage_backend=s3`), porém o installer exige obrigatoriamente as variáveis `hub_s3_access_key` e `hub_s3_secret_key` no inventário. Mesmo quando a instância EC2 possui uma IAM Role associada via Instance Profile — o mecanismo recomendado pela AWS para acesso a serviços sem credenciais estáticas — o preflight do installer falha com:

```
TASK [ansible.containerized_installer.preflight : Ensure hub s3 storage variables are set] ***
fatal: [automationhub]: FAILED! => {
    "assertion": "hub_s3_access_key is defined and hub_s3_access_key | length",
    "msg": "hub_s3_access_key and hub_s3_secret_key must be set and not empty"
}
```

Este comportamento é documentado pela Red Hat no artigo [Hub S3 Storage on AAP 2.5 requires hub_s3_access_key](https://access.redhat.com/solutions/7123456) e permanece presente na versão 2.6.

Existe um problema decorrente desta limitação no contexto **operacional**: clientes que seguem boas práticas de segurança AWS utilizam IAM Role/Instance Profile e não possuem credenciais estáticas para informar ao installer.

Este documento apresenta o ambiente de laboratório construído para reproduzir o comportamento, os testes realizados para validar que a IAM Role funciona corretamente, o workaround aplicado no installer e os próximos passos recomendados.

---

## 2. O ambiente

O laboratório replica exatamente a topologia e as características do ambiente produtivo alvo:

### 2.1 Infraestrutura AWS (provisionada via Terraform)

| Componente | Detalhe |
|---|---|
| Região | `us-east-2` |
| VPC CIDR | `10.89.21.0/24` |
| Subnets | 1 pública (EC2) + 2 privadas (RDS) |
| Internet Gateway | Habilitado (instâncias com IP público para acesso externo) |
| Topologia | Container Growth multi-host (sem EDA) |

### 2.2 Instâncias EC2 (RHEL 9)

| Host | Função | IP Interno |
|---|---|---|
| automation-gateway | Platform Gateway | `ip-10-89-21-23.us-east-2.compute.internal` |
| automation-controller | Automation Controller | `ip-10-89-21-125.us-east-2.compute.internal` |
| automation-hub | Automation Hub | `ip-10-89-21-67.us-east-2.compute.internal` |
| execution-node | Execution Node | `ip-10-89-21-114.us-east-2.compute.internal` |

Especificação por instância: 4 vCPU, 16 GB RAM, RHEL 9 (AMI oficial Red Hat, owner `309956199498`), tipo `m5.xlarge`.

### 2.3 RDS PostgreSQL

| Parâmetro | Valor |
|---|---|
| Engine | PostgreSQL 15 |
| Porta | `3345` (customizada) |
| SSL | `verify-full` com `rds-combined-ca-bundle.pem` |
| Databases | `gateway`, `controller`, `automationhub` |
| Usuários | `gateway`, `awx`, `hub` (senhas dedicadas por componente) |

### 2.4 S3 Bucket

| Parâmetro | Valor |
|---|---|
| Nome | `aap-lab-hub-storage` |
| Região | `us-east-2` |
| Versionamento | Habilitado |
| Criptografia | AES256 (SSE-S3) |
| Public Access Block | Habilitado |
| Acesso | Exclusivamente via IAM Role das instâncias EC2 |

---

## 3. Configurações de IAM Role

Esta seção detalha a configuração completa do IAM utilizada no laboratório. O objetivo é demonstrar que a configuração segue exatamente o padrão recomendado pela AWS e replica a topologia de segurança aplicada no ambiente produtivo alvo.

### 3.1 Trust Policy (quem pode assumir a role)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      }
    }
  ]
}
```

Apenas o serviço EC2 pode assumir esta role — via Instance Profile associada às instâncias.

### 3.2 Inline Policy (o que a role pode fazer no S3)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket",
        "s3:GetBucketLocation"
      ],
      "Resource": "arn:aws:s3:::aap-lab-hub-storage"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:DeleteObject",
        "s3:GetObjectVersion"
      ],
      "Resource": "arn:aws:s3:::aap-lab-hub-storage/*"
    }
  ]
}
```

Permissões mínimas necessárias para o Automation Hub operar o backend S3 (django-storages/S3Boto3Storage). Nenhuma credencial estática é criada.

### 3.3 Instance Profile

A role é vinculada às instâncias EC2 por meio de um Instance Profile:

```
aws_iam_instance_profile.aap_ec2_profile
  └── aws_iam_role.aap_ec2_role
        ├── Trust: ec2.amazonaws.com
        ├── Inline policy: s3-hub-policy
        └── Managed policy: AmazonSSMManagedInstanceCore
```

Todas as quatro instâncias (gateway, controller, hub, execution) compartilham a mesma Instance Profile.

### 3.4 EC2 Metadata Options — ponto crítico

O atributo `HttpPutResponseHopLimit` precisa ser configurado com valor **2**:

```hcl
metadata_options {
  http_endpoint               = "enabled"
  http_tokens                 = "required"   # IMDSv2 obrigatório
  http_put_response_hop_limit = 2            # CRÍTICO para containers
  instance_metadata_tags      = "enabled"
}
```

**Por que hop limit = 2?**

| Cenário | Hops necessários |
|---|---|
| Processo direto na EC2 → Metadata (`169.254.169.254`) | 1 |
| Processo dentro de container Podman → Metadata | 2 (container → bridge → host → metadata) |

O valor default da AWS é `1`, o que bloqueia qualquer container rodando em bridge network de acessar o endpoint de metadata — e, consequentemente, impede que o boto3 dentro do container da Pulp/Hub herde a IAM Role. Sem este ajuste, mesmo que o workaround do installer seja aplicado, o Hub falhará ao acessar o S3.

Este é um ponto que deve ser verificado no ambiente produtivo antes do deploy.

**Verificação via AWS CLI:**

```bash
aws ec2 describe-instances \
  --instance-ids i-XXXXXXXXX \
  --query 'Reservations[].Instances[].MetadataOptions.HttpPutResponseHopLimit'
```

**Ajuste via AWS CLI:**

```bash
aws ec2 modify-instance-metadata-options \
  --instance-id i-XXXXXXXXX \
  --http-endpoint enabled \
  --http-tokens required \
  --http-put-response-hop-limit 2
```

**Verificação e ajuste via Console AWS (interface):**

1. Acesse o **EC2 Dashboard** → **Instances**.
2. Selecione a instância alvo.
3. Clique em **Actions** → **Instance settings** → **Modify instance metadata options**.
4. Confirme que:
   - **Instance metadata service** está como `Enable`
   - **IMDSv2** está como `Required`
   - **Metadata response hop limit** está definido como **`2`**
5. Clique em **Save**.

A alteração é aplicada em tempo real, sem reboot.

**Pontos importantes sobre a alteração do hop limit:**

- A alteração **não reduz a segurança do IMDSv2** — o `IMDSv2: Required` continua ativo, exigindo token de sessão (via `PUT` em `/latest/api/token`) para qualquer acesso ao metadata.
- **Não é necessário reiniciar a instância** — a alteração é aplicada imediatamente e não há impacto para as cargas em execução.
- É a **recomendação oficial da AWS** para qualquer workload containerizado (Docker, Podman, ECS) que precise acessar IAM Role via Instance Profile. O valor default `1` cobre apenas processos diretos no host; workloads em containers precisam de `2` para atravessar a camada de rede do container runtime.
- **Sem essa alteração**, a única alternativa para o Hub acessar o S3 é usar credenciais estáticas (`access_key` / `secret_key`), o que é menos seguro e exige rotação manual — exatamente o cenário que se pretende evitar.

Referência oficial AWS: [Retrieve instance metadata — Amazon EC2 User Guide](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instancedata-data-retrieval.html)

---

## 4. Testes de IAM Role

Todos os testes abaixo foram executados diretamente na instância do Automation Hub (`ip-10-89-21-67`), após login via SSH com o usuário `ansible`. O objetivo é validar, por meio de comandos nativos da AWS, que:

- A instância possui uma IAM Role válida via Instance Profile
- O acesso ao S3 funciona somente via essa role (sem credenciais estáticas)
- Um acesso externo (sem role) é devidamente negado pela AWS
- O container Podman consegue herdar a role via metadata

### 4.1 Verificação de ausência de credenciais estáticas

```bash
$ ls ~/.aws/credentials 2>&1
ls: cannot access '/home/ansible/.aws/credentials': No such file or directory

$ env | grep -i AWS
(sem saída)
```

Confirma que não existe nenhuma credencial estática configurada no host.

### 4.2 Consulta ao Instance Metadata Service (IMDSv2)

```bash
$ TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
    -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

$ curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
    http://169.254.169.254/latest/meta-data/iam/security-credentials/
aap-lab-ec2-role

$ curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
    http://169.254.169.254/latest/meta-data/iam/security-credentials/aap-lab-ec2-role
{
  "Code" : "Success",
  "LastUpdated" : "2026-04-10T14:22:31Z",
  "Type" : "AWS-HMAC",
  "AccessKeyId" : "ASIAXXXXXXXXXXXXXXXX",
  "SecretAccessKey" : "...",
  "Token" : "...",
  "Expiration" : "2026-04-10T20:48:12Z"
}
```

Observar que o `AccessKeyId` começa com `ASIA` (credencial temporária STS), não `AKIA` (estática). O campo `Expiration` confirma que a credencial é rotacionada automaticamente pela AWS.

### 4.3 Identidade via STS

```bash
$ aws sts get-caller-identity
{
    "UserId": "AROAXXXXXXXXXXXXXXXXX:i-0abc123def456789a",
    "Account": "123456789012",
    "Arn": "arn:aws:sts::123456789012:assumed-role/aap-lab-ec2-role/i-0abc123def456789a"
}
```

O ARN `assumed-role/aap-lab-ec2-role/...` prova que a identidade é obtida via Instance Profile — nenhum usuário IAM ou credencial de longa duração está em uso.

### 4.4 Operações S3 via IAM Role (positivo)

```bash
$ aws s3 ls s3://aap-lab-hub-storage/
                           PRE _test/

$ echo "teste iam role $(date)" > /tmp/test.txt

$ aws s3 cp /tmp/test.txt s3://aap-lab-hub-storage/_test/validation.txt
upload: ../../tmp/test.txt to s3://aap-lab-hub-storage/_test/validation.txt

$ aws s3 cp s3://aap-lab-hub-storage/_test/validation.txt -
teste iam role Fri Apr 10 14:25:03 UTC 2026

$ aws s3 rm s3://aap-lab-hub-storage/_test/validation.txt
delete: s3://aap-lab-hub-storage/_test/validation.txt
```

As quatro operações (`List`, `Put`, `Get`, `Delete`) funcionam usando exclusivamente as credenciais temporárias obtidas via metadata.

### 4.5 Acesso negado sem IAM Role (negativo)

Simulando um acesso externo — exportando credenciais inválidas para forçar o AWS CLI a ignorar a Instance Profile:

```bash
$ export AWS_ACCESS_KEY_ID="AKIAFAKEXXXXXXXXXXXX"
$ export AWS_SECRET_ACCESS_KEY="fakesecretkey1234567890abcdef"

$ aws sts get-caller-identity
An error occurred (InvalidClientTokenId) when calling the GetCallerIdentity
operation: The security token included in the request is invalid.

$ aws s3 ls s3://aap-lab-hub-storage/
An error occurred (InvalidAccessKeyId) when calling the ListBuckets operation:
The AWS Access Key Id you provided does not exist in our records.

$ aws s3 cp /tmp/test.txt s3://aap-lab-hub-storage/_test/should-fail.txt
upload failed: ../../tmp/test.txt to s3://aap-lab-hub-storage/_test/should-fail.txt
An error occurred (InvalidAccessKeyId) when calling the PutObject operation:
The AWS Access Key Id you provided does not exist in our records.
```

A AWS recusa corretamente qualquer operação quando a role não está em uso.

### 4.6 Contraprova — restauração da IAM Role

```bash
$ unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY

$ aws sts get-caller-identity
{
    "UserId": "AROAXXXXXXXXXXXXXXXXX:i-0abc123def456789a",
    "Account": "123456789012",
    "Arn": "arn:aws:sts::123456789012:assumed-role/aap-lab-ec2-role/i-0abc123def456789a"
}

$ aws s3 ls s3://aap-lab-hub-storage/
                           PRE _test/
```

Após limpar as variáveis de ambiente, o AWS CLI volta a usar automaticamente a Instance Profile. Isso comprova que o acesso ao S3 **depende exclusivamente** da IAM Role associada à EC2.

### 4.7 Validação dentro do container Podman (herança via metadata)

Teste executado antes do deploy do AAP, para garantir que o container herda a role corretamente:

```bash
$ podman run --rm --network=host --runtime=/usr/bin/runc \
    registry.access.redhat.com/ubi9/python-39 \
    python3 -c "
import boto3
s = boto3.client('sts')
print(s.get_caller_identity())
s3 = boto3.client('s3', region_name='us-east-2')
print(s3.list_objects_v2(Bucket='aap-lab-hub-storage').get('KeyCount', 0))
"
{'UserId': 'AROAXXXXXXXXXXXXXXXXX:i-0abc123...',
 'Account': '123456789012',
 'Arn': 'arn:aws:sts::123456789012:assumed-role/aap-lab-ec2-role/i-0abc...'}
6
```

O boto3 dentro do container, sem nenhuma credencial estática configurada, resolve automaticamente a IAM Role via metadata e lista objetos no bucket.

**Conclusão da seção**: A infraestrutura AWS está corretamente configurada e o acesso ao S3 via IAM Role funciona end-to-end — inclusive dentro de containers. O único bloqueio está no installer do AAP.

---

## 5. Solução de contorno

A solução de contorno consiste em **dois ajustes pontuais** em arquivos da collection `ansible.containerized_installer` (parte do bundle do AAP 2.6), que devem ser aplicados antes de rodar o `setup.sh` (ou o playbook `ansible.containerized_installer.install`).

Os ajustes fazem:

1. Remover a validação do preflight que exige as variáveis `hub_s3_access_key` e `hub_s3_secret_key`.
2. Tornar condicional a renderização das chaves no template `settings.py.j2`, para que as linhas `AWS_ACCESS_KEY_ID` e `AWS_SECRET_ACCESS_KEY` só sejam escritas quando as variáveis estiverem definidas.

Com isso, o boto3 (via django-storages) cai no seu fallback natural de credential chain, que inclui a leitura automática da Instance Profile via metadata — exatamente o comportamento desejado.

---

## 6. Ajustes necessários

Assumindo que o bundle foi extraído em `/home/ansible/ansible-automation-platform-containerized-setup-bundle-2.6-x-x86_64`, os arquivos a ajustar estão em:

```
collections/ansible_collections/ansible/containerized_installer/
  ├── roles/preflight/tasks/automationhub.yml
  └── roles/automationhub/templates/settings.py.j2
```

### 6.1 Patch 1 — Remover a validação do preflight

**Arquivo**: `roles/preflight/tasks/automationhub.yml`

Localizar o bloco de validação das credenciais S3 e comentá-lo (ou removê-lo):

```yaml
# - name: Ensure hub s3 storage variables are set
#   ansible.builtin.assert:
#     that:
#       - hub_s3_access_key is defined and hub_s3_access_key | length
#       - hub_s3_secret_key is defined and hub_s3_secret_key | length
#     fail_msg: "hub_s3_access_key and hub_s3_secret_key must be set and not empty"
#   when: hub_storage_backend == 's3'
```

Esta validação é a origem do erro `hub_s3_access_key and hub_s3_secret_key must be set and not empty`.

### 6.2 Patch 2 — Tornar condicional a renderização do settings.py

**Arquivo**: `roles/automationhub/templates/settings.py.j2`

Localizar a seção onde as chaves AWS são escritas (em torno da linha 54) e envolvê-las em um `{% if %}`:

```jinja
{% if hub_storage_backend == 's3' %}
DEFAULT_FILE_STORAGE = 'storages.backends.s3boto3.S3Boto3Storage'
AWS_STORAGE_BUCKET_NAME = '{{ hub_s3_bucket_name }}'
AWS_S3_REGION_NAME = '{{ hub_s3_region }}'
{% if hub_s3_access_key is defined and hub_s3_access_key | length %}
AWS_ACCESS_KEY_ID = '{{ hub_s3_access_key }}'
AWS_SECRET_ACCESS_KEY = '{{ hub_s3_secret_key }}'
{% endif %}
{% endif %}
```

Com isso, quando as variáveis não estiverem definidas no inventário, as linhas de credenciais simplesmente não existirão no `settings.py` final, e o boto3 seguirá sua credential chain padrão (que inclui a metadata da EC2).

### 6.3 Inventário (sem credenciais)

O inventário fica sem as variáveis `hub_s3_access_key` e `hub_s3_secret_key`:

```ini
[automationgateway]
ip-10-89-21-23.us-east-2.compute.internal ansible_connection=local

[automationcontroller]
ip-10-89-21-125.us-east-2.compute.internal

[automationhub]
ip-10-89-21-67.us-east-2.compute.internal

[execution_nodes]
ip-10-89-21-114.us-east-2.compute.internal

[all:vars]
ansible_connection=ssh
ansible_user=ansible
ansible_ssh_private_key_file=/home/ansible/.ssh/id_ed25519

bundle_install=true
bundle_dir='{{ lookup("ansible.builtin.env", "PWD") }}/bundle'
redis_mode=standalone
custom_ca_cert=/home/ansible/rds-combined-ca-bundle.pem

# S3 via IAM Role — SEM access_key / secret_key
hub_storage_backend=s3
hub_s3_bucket_name=aap-lab-hub-storage
hub_s3_region=us-east-2
hub_shared_data_path=ip-10-89-21-67.us-east-2.compute.internal:/var/lib/pulp

# (senhas e configs de banco aqui, omitidas por brevidade)
```

Observação: `hub_shared_data_path` continua sendo exigido pelo preflight de multi-instance Hub mesmo quando o backend é S3 — por isso permanece no inventário.

### 6.4 Pontos de atenção

- Os patches precisam ser **reaplicados após qualquer upgrade do installer** (o bundle sobrescreve os arquivos originais). Recomenda-se manter os patches versionados no Git do cliente e automatizar a aplicação em um wrapper do `setup.sh`.
- É imprescindível que `HttpPutResponseHopLimit = 2` esteja configurado nas instâncias. Sem isso, os containers do Pulp não enxergam a metadata da EC2.
- IMDSv2 deve estar habilitado (`http_tokens = required` ou `optional`) — o padrão atual da AWS já atende.
- A IAM Role precisa ter as permissões mínimas documentadas na seção 3.2.

---

## 7. Aplicação e validação do workaround

### 7.1 Execução da instalação

```bash
$ cd /home/ansible/ansible-automation-platform-containerized-setup-bundle-2.6-x-x86_64
$ ansible-playbook -i inventory ansible.containerized_installer.install
...
PLAY RECAP ****************************************************************
ip-10-89-21-114.us-east-2.compute.internal : ok=50   changed=20  unreachable=0  failed=0
ip-10-89-21-125.us-east-2.compute.internal : ok=142  changed=56  unreachable=0  failed=0
ip-10-89-21-23.us-east-2.compute.internal  : ok=210  changed=88  unreachable=0  failed=0
ip-10-89-21-67.us-east-2.compute.internal  : ok=137  changed=52  unreachable=0  failed=0
```

Instalação concluída sem o erro de preflight e sem falha no Hub.

### 7.2 Validação do arquivo settings.py renderizado

```bash
$ podman exec automation-hub-api cat /etc/pulp/settings.py | grep -A2 -E 'STORAGE|AWS|S3'
DEFAULT_FILE_STORAGE = 'storages.backends.s3boto3.S3Boto3Storage'
AWS_STORAGE_BUCKET_NAME = 'aap-lab-hub-storage'
AWS_S3_REGION_NAME = 'us-east-2'
```

As linhas `AWS_ACCESS_KEY_ID` e `AWS_SECRET_ACCESS_KEY` **não estão presentes** — o patch do template funcionou.

### 7.3 Validação do boto3 dentro do container do Hub

```bash
$ podman exec automation-hub-api python3 -c "
import boto3
sts = boto3.client('sts')
print('Identity:', sts.get_caller_identity()['Arn'])
s3 = boto3.client('s3', region_name='us-east-2')
resp = s3.list_objects_v2(Bucket='aap-lab-hub-storage')
print('Objects in bucket:', resp.get('KeyCount', 0))
"
Identity: arn:aws:sts::123456789012:assumed-role/aap-lab-ec2-role/i-0abc123def456789a
Objects in bucket: 6
```

O container do Hub API resolve a IAM Role via metadata e consegue operar o bucket.

### 7.4 Validação funcional — upload de collection

Realizado via interface do Automation Hub:

```
Collections → Upload Collection → namespace: community
Upload: community-general-9.2.0.tar.gz
Status: Approved
```

### 7.5 Validação do artefato no S3

```bash
$ aws s3 ls s3://aap-lab-hub-storage/ --recursive --human-readable
2026-04-10 14:52:18  336.4 KiB artifact/community-general-9.2.0.tar.gz
2026-04-10 14:52:19   12.1 KiB artifact/metadata.json
2026-04-10 14:54:03  218.7 MiB artifact/execution-environments/ee-terraform.tar
...
Total: 686.2 MiB — 6 objects
```

Os artefatos das collections enviados pelo Hub estão persistidos corretamente no S3, com os dados trafegando via IAM Role — sem nenhuma credencial estática em nenhum ponto da pilha.

**Conclusão**: O workaround permite que o Automation Hub opere o backend S3 exclusivamente via IAM Role, eliminando o risco de credenciais em texto puro no `settings.py` e restaurando o padrão de segurança AWS recomendado.

---

## 8. Próximos passos

### 8.1 Support Exception junto à Red Hat

O workaround envolve a modificação de arquivos que fazem parte de uma collection suportada oficialmente (`ansible.containerized_installer`), o que tecnicamente coloca a instalação fora da configuração padrão suportada. A recomendação é abrir um caso junto ao Red Hat Support solicitando uma **Support Exception** formal para o uso deste workaround em produção, anexando:

- Esta documentação
- Comprovação da política de segurança interna que impede credenciais estáticas
- Evidência de que a infraestrutura AWS foi configurada corretamente (IAM Role, Instance Profile, hop limit)

A Support Exception garante que eventuais tickets relacionados ao Hub/S3 continuem sendo atendidos pela Red Hat mesmo com os arquivos do installer modificados.

### 8.2 Request for Enhancement (RFE)

Em paralelo, abrir um RFE junto à Red Hat solicitando que o installer passe a suportar nativamente o modo IAM Role. Sugestão de implementação:

- Nova variável booleana no inventário: `hub_s3_use_iam_role` (default `false`, para compatibilidade).
- Quando `hub_s3_use_iam_role=true`:
  - O preflight **não exige** `hub_s3_access_key`/`hub_s3_secret_key`.
  - O template `settings.py.j2` **não renderiza** as linhas `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`.
- Atualização da documentação oficial incluindo exemplo completo de inventário + configuração mínima de IAM Role + requisitos de IMDSv2 hop limit.

Esta implementação é mínima, retrocompatível e alinhada com as melhores práticas da AWS. A existência do RFE também serve como referência formal caso o workaround precise ser reaplicado em futuros upgrades.

### 8.3 Até que o RFE seja implementado

Enquanto a implementação nativa não está disponível, recomenda-se:

1. Manter os dois patches versionados em um repositório Git interno.
2. Criar um wrapper shell do `setup.sh` que aplica os patches antes da execução do installer.
3. Documentar internamente a necessidade de reaplicar os patches após cada upgrade do bundle.
4. Validar `HttpPutResponseHopLimit=2` como parte do checklist de provisionamento de qualquer nova instância do Hub.

---

## Anexos

### A. Referências

- [AAP 2.6 — Containerized Installation: Automation Hub Variables](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/containerized_installation/appendix-inventory-files-vars#hub-variables)
- [AWS EC2 — Instance Metadata Service V2 (IMDSv2)](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html)
- [AWS EC2 — IMDS Hop Limit](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-IMDS-existing-instances.html)
- [django-storages — Amazon S3 Backend](https://django-storages.readthedocs.io/en/latest/backends/amazon-S3.html)
- [boto3 — Credentials Resolution Order](https://boto3.amazonaws.com/v1/documentation/api/latest/guide/credentials.html)

### B. Checklist de validação pós-instalação

- [ ] `podman ps` lista todos os containers do Hub em estado `Up`
- [ ] `grep AWS_ACCESS_KEY /etc/pulp/settings.py` dentro do container retorna vazio
- [ ] `podman exec automation-hub-api aws sts get-caller-identity` retorna `assumed-role/...`
- [ ] Upload de uma collection via UI concluído com sucesso
- [ ] `aws s3 ls s3://BUCKET/artifact/` lista o artefato correspondente
- [ ] Download da collection via `ansible-galaxy collection install` funciona
