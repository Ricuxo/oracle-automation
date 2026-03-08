# orahost_os_prereqs

Role responsável por preparar o sistema operacional para instalação do Oracle Database.

## Funcionalidades

Esta role executa as seguintes configurações:

- Instala o pacote de pré-requisitos fornecido pela Oracle
- Configura sincronização de tempo com chrony
- Define SELinux como permissive
- Desativa o serviço firewalld

## Variáveis

| Variável | Descrição | Default |
|--------|-----------|--------|
| oracle_preinstall_package | Pacote de prerequisitos Oracle | oracle-database-preinstall-19c |

Exemplo:

oracle_preinstall_package: oracle-database-preinstall-21c

## Observações

Esta role utiliza configurações simplificadas para facilitar automação e ambientes de laboratório.

Dependendo da política de segurança da organização, pode ser necessário ajustar:

- Configuração do SELinux
- Regras de firewall
- Configuração de NTP/chrony

Antes de utilizar em produção, recomenda-se revisar essas configurações conforme os padrões de segurança da organização.