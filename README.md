# oracle-automation

Automação enterprise para provisionamento e operação de Oracle Database em Linux.
Suporta instalação em filesystem e ASM, Single Instance e RAC.

---

## Pré-requisitos

- Ansible >= 2.14
- Python >= 3.9 no controller
- RHEL/OEL 8 ou 9 nos hosts alvo
- Acesso ao repositório Oracle Linux (para oracle-preinstall)
- Acesso SSH inicial via root ou usuário do SO para bootstrap

---

## Estrutura do Projeto
```
oracle-automation/
├── galaxy.yml
├── requirements.yml
├── .yamllint.yml
├── README.md
├── inventories/
│   ├── dev/
│   │   ├── hosts.yml
│   │   └── group_vars/
│   │       └── dbservers.yml
│   └── prod/
│       ├── hosts.yml
│       └── group_vars/
│           └── dbservers.yml
├── playbooks/
│   ├── bootstrap.yml
│   ├── install_oracle.yml
│   ├── install_oracle_si_fs.yml
│   ├── install_oracle_si_asm.yml
│   └── validate_oracle.yml
├── roles/
│   ├── orahost_bootstrap         ← cria usuário ansible, configura sudo e SSH
│   ├── orahost_common            ← ferramentas base (rlwrap, vim, sysstat...)
│   ├── orahost_os_prereqs        ← oracle-preinstall, THP, HugePages, NTP, firewall
│   ├── orahost_directories       ← diretórios, bash_profile, aliases
│   ├── orahost_storage           ← filesystem: partição, formato, mount
│   ├── orahost_storage_asm       ← ASM: asmlib, asmfd ou udev
│   ├── oragrid_install           ← instalação Grid Infrastructure
│   ├── oragrid_asm_diskgroups    ← criação dos diskgroups ASM
│   ├── oradb_install             ← instalação software RDBMS
│   ├── oradb_create              ← criação do banco via DBCA
│   ├── oradb_listener            ← configuração do listener
│   ├── oradb_config              ← parâmetros, archivelog, FRA, systemd
│   ├── oradb_dataguard           ← configuração Data Guard broker
│   ├── oradb_patching            ← OPatch / OPatchAuto
│   └── oradb_healthcheck         ← validações pós-instalação
└── plugins/
    ├── modules/
    │   ├── oracle_listener_status.py
    │   ├── oracle_patch_inventory.py
    │   ├── oracle_cluster_validate.py
    │   └── oracle_dataguard_validate.py
    └── filter/
        └── oracle_filters.py
```

---

## Convenção de Nomenclatura das Roles

| Prefixo | Responsabilidade |
|---|---|
| `orahost_*` | Sistema operacional: pacotes, kernel, storage físico, diretórios |
| `oragrid_*` | Grid Infrastructure e ASM |
| `oradb_*` | Oracle Database: instalação, criação, configuração |

---

## Como Recriar a Estrutura do Projeto

### 1. Clonar ou criar o diretório base
```bash
mkdir oracle-automation
cd oracle-automation
```

### 2. Instalar dependências das collections
```bash
ansible-galaxy collection install -r requirements.yml
```

### 3. Criar as roles com ansible-galaxy
```bash
cd oracle-automation/roles

ansible-galaxy role init orahost_bootstrap
ansible-galaxy role init orahost_common
ansible-galaxy role init orahost_os_prereqs
ansible-galaxy role init orahost_directories
ansible-galaxy role init orahost_storage
ansible-galaxy role init orahost_storage_asm
ansible-galaxy role init oragrid_install
ansible-galaxy role init oragrid_asm_diskgroups
ansible-galaxy role init oradb_install
ansible-galaxy role init oradb_create
ansible-galaxy role init oradb_listener
ansible-galaxy role init oradb_config
ansible-galaxy role init oradb_dataguard
ansible-galaxy role init oradb_patching
ansible-galaxy role init oradb_healthcheck
```

### 4. Criar estrutura de plugins
```bash
cd oracle-automation
mkdir -p plugins/modules
mkdir -p plugins/filter
```

---

## Ordem de Execução das Roles
```
0. orahost_bootstrap       ← cria oracle_ansible (roda como root)
1. orahost_common          ← ferramentas base
2. orahost_os_prereqs      ← oracle-preinstall, THP, NTP, firewall
3. orahost_directories     ← diretórios, bash_profile, aliases
4. orahost_storage         ← somente se oracle_storage_type = filesystem
5. orahost_storage_asm     ← somente se oracle_storage_type = asm
6. oragrid_install         ← somente se oracle_storage_type = asm
7. oragrid_asm_diskgroups  ← somente se oracle_storage_type = asm
8. oradb_install           ← instalação software RDBMS
9. oradb_create            ← criação do banco
10. oradb_listener         ← configuração listener
11. oradb_config           ← parâmetros e configurações pós-criação
```

---

## Modelo de Autenticação
```
1ª execução  →  bootstrap.yml
                  conecta como root ou usuário inicial do SO
                  └── cria usuário oracle_ansible
                  └── configura sudo
                  └── distribui chave SSH pública

2ª execução em diante → todos os outros playbooks
                  conecta como oracle_ansible
                  └── eleva privilégios via sudo quando necessário
```

---

## Topologias Suportadas

| Topologia | Storage | Playbook |
|---|---|---|
| Single Instance | Filesystem | `install_oracle_si_fs.yml` |
| Single Instance | ASM | `install_oracle_si_asm.yml` |
| RAC | ASM | `install_oracle.yml` |

---

## Execução

### Passo 1 – Bootstrap (primeira execução)
```bash
ansible-playbook playbooks/bootstrap.yml \
  -i inventories/dev \
  -u root \
  --ask-pass
```

### Passo 2 – Instalação completa Single Instance Filesystem
```bash
ansible-playbook playbooks/install_oracle_si_fs.yml \
  -i inventories/dev \
  --ask-vault-pass
```

### Passo 3 – Instalação completa Single Instance ASM
```bash
ansible-playbook playbooks/install_oracle_si_asm.yml \
  -i inventories/dev \
  --ask-vault-pass
```

### Executar role isolada
```bash
# Apenas ferramentas base
ansible-playbook playbooks/install_oracle.yml \
  -i inventories/dev \
  --tags orahost_common

# Apenas pré-requisitos de SO
ansible-playbook playbooks/install_oracle.yml \
  -i inventories/dev \
  --tags orahost_os_prereqs

# Apenas criar diretórios
ansible-playbook playbooks/install_oracle.yml \
  -i inventories/dev \
  --tags orahost_directories

# Apenas configurar ASM
ansible-playbook playbooks/install_oracle.yml \
  -i inventories/dev \
  --tags orahost_storage_asm
```

---

## Segurança

Todas as senhas são gerenciadas via **Ansible Vault**.
```bash
# Criptografar arquivo de variáveis sensíveis
ansible-vault encrypt inventories/dev/group_vars/vault.yml

# Executar com vault
ansible-playbook playbooks/install_oracle.yml \
  -i inventories/dev \
  --ask-vault-pass
```

---

## Compatibilidade

| Oracle Version | RHEL/OEL 8 | RHEL/OEL 9 | Single Instance | RAC |
|---|---|---|---|---|
| 19c (19.3+) | ✅ | ✅ | ✅ | ✅ |
| 21c (21.3+) | ✅ | ✅ | ✅ | ✅ |