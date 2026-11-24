<h1 align="center"> 🚀 Ansible </h1>

## 📑 Sumário

- [O que é o Ansible](#-o-que-é-o-ansible)
- [Como funciona](#-como-funciona)
- [Pré-requisitos](#-pré-requisitos)
- [Instalação](#-instalação)
- [Primeiros passos](#-primeiros-passos)
- [Inventário](#-inventário)
- [Comandos ad-hoc](#-comandos-ad-hoc)
- [Playbooks](#-playbooks)
- [Módulos mais usados](#-módulos-mais-usados)
- [Variáveis](#-variáveis)
- [Condicionais e loops](#-condicionais-e-loops)
- [Handlers](#-handlers)
- [Templates (Jinja2)](#-templates-jinja2)
- [Roles](#-roles)
- [Ansible Vault](#-ansible-vault)
- [Ansible Galaxy e Collections](#-ansible-galaxy-e-collections)
- [Comandos essenciais](#-comandos-essenciais)
- [Exemplo completo](#-exemplo-completo)
- [Solução de problemas](#-solução-de-problemas)
- [Boas práticas](#-boas-práticas)
- [Referências](#-referências)

---

## 📖 O que é o Ansible

O [Ansible](https://www.ansible.com/) é uma ferramenta open source (mantida pela Red Hat) de **automação de TI**: gerenciamento de configuração, provisionamento, deploy de aplicações e orquestração de tarefas.

Você descreve o estado desejado dos servidores em arquivos YAML legíveis, e o Ansible aplica essas mudanças em quantas máquinas for necessário.

**Vantagens:**

- ✅ **Sem agente**: usa SSH (Linux) ou WinRM (Windows), sem instalar nada nos servidores gerenciados
- ✅ **Declarativo e legível**: playbooks em YAML
- ✅ **Idempotente**: executar várias vezes gera o mesmo resultado
- ✅ Grande biblioteca de módulos e comunidade ativa
- ✅ Configuração versionável no Git (infraestrutura como código)

---

## ⚙️ Como funciona

```
Control Node  ──(SSH)──▶  Managed Nodes
(Ansible instalado)       (servidores alvo)
   │
   ├── Inventário  (quais máquinas)
   ├── Playbook    (o que fazer)
   └── Módulos     (como fazer)
```

| Conceito | Descrição |
|---|---|
| **Control node** | Máquina onde o Ansible está instalado e de onde os comandos são executados |
| **Managed node** | Servidor gerenciado pelo Ansible |
| **Inventário** | Lista de hosts e grupos |
| **Módulo** | Unidade de trabalho (instalar pacote, copiar arquivo, reiniciar serviço...) |
| **Task** | Chamada a um módulo com seus parâmetros |
| **Play** | Conjunto de tasks aplicado a um grupo de hosts |
| **Playbook** | Arquivo YAML com um ou mais plays |
| **Role** | Forma organizada e reutilizável de empacotar tasks, variáveis e templates |
| **Handler** | Task executada somente quando notificada por uma mudança |
| **Facts** | Informações coletadas automaticamente de cada host |

---

## 📋 Pré-requisitos

**Control node:**

- Linux, macOS ou WSL (o Windows nativo não é suportado como control node)
- Python 3 instalado

**Managed nodes:**

- Acesso SSH (de preferência com chave)
- Python 3 instalado (para a maioria dos módulos)
- Usuário com permissão de `sudo`, quando necessário

---

## 💾 Instalação

### Ubuntu / Debian

```bash
sudo apt update
sudo apt install -y software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install -y ansible
```

### macOS

```bash
brew install ansible
```

### Via pip (qualquer sistema com Python)

```bash
python3 -m pip install --user ansible
```

### Verificando a instalação

```bash
ansible --version
```

---

## 🏁 Primeiros passos

```bash
# 1. Crie a pasta do projeto
mkdir meu-projeto-ansible && cd meu-projeto-ansible

# 2. Crie um inventário
echo "[web]
192.168.56.11
192.168.56.12" > inventory.ini

# 3. Teste a conexão com todos os hosts
ansible all -i inventory.ini -m ping -u vagrant

# 4. Rode seu primeiro comando ad-hoc
ansible web -i inventory.ini -m command -a "uptime" -u vagrant
```

Resposta esperada do `ping`:

```
192.168.56.11 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

---

## 🗂️ Inventário

O inventário define **em quais máquinas** o Ansible vai atuar.

### Formato INI

```ini
[web]
web1 ansible_host=192.168.56.11
web2 ansible_host=192.168.56.12

[db]
db1 ansible_host=192.168.56.21

[producao:children]
web
db

[all:vars]
ansible_user=vagrant
ansible_ssh_private_key_file=~/.ssh/id_rsa
```

### Formato YAML

```yaml
all:
  vars:
    ansible_user: vagrant
  children:
    web:
      hosts:
        web1:
          ansible_host: 192.168.56.11
        web2:
          ansible_host: 192.168.56.12
    db:
      hosts:
        db1:
          ansible_host: 192.168.56.21
```

### Variáveis de conexão mais comuns

| Variável | Descrição |
|---|---|
| `ansible_host` | IP ou hostname real |
| `ansible_user` | Usuário SSH |
| `ansible_port` | Porta SSH |
| `ansible_ssh_private_key_file` | Chave privada |
| `ansible_become` | Usar `sudo` |
| `ansible_python_interpreter` | Caminho do Python no host |

### Arquivo `ansible.cfg`

Evita repetir parâmetros a cada comando:

```ini
[defaults]
inventory = ./inventory.ini
remote_user = vagrant
host_key_checking = False
retry_files_enabled = False

[privilege_escalation]
become = True
```

> ⚠️ `host_key_checking = False` é conveniente em laboratório, mas evite em produção.

---

## ⚡ Comandos ad-hoc

Tarefas rápidas, sem escrever playbook:

```bash
# Testar conectividade
ansible all -m ping

# Executar um comando
ansible web -m command -a "df -h"

# Instalar um pacote
ansible web -m apt -a "name=nginx state=present update_cache=yes" --become

# Reiniciar um serviço
ansible web -m service -a "name=nginx state=restarted" --become

# Copiar um arquivo
ansible web -m copy -a "src=./index.html dest=/var/www/html/index.html" --become

# Coletar facts
ansible web -m setup
```

---

## 📝 Playbooks

Um playbook descreve **o estado desejado** em YAML.

```yaml
---
- name: Configurar servidores web
  hosts: web
  become: true

  tasks:
    - name: Instalar o Nginx
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: true

    - name: Garantir que o Nginx está ativo e habilitado
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: true
```

Execução:

```bash
ansible-playbook playbook.yml
```

Opções úteis:

```bash
ansible-playbook playbook.yml --check          # simula, sem alterar nada
ansible-playbook playbook.yml --diff           # mostra as diferenças
ansible-playbook playbook.yml --limit web1     # restringe a um host/grupo
ansible-playbook playbook.yml --tags nginx     # roda apenas tasks com a tag
ansible-playbook playbook.yml -e "porta=8080"  # passa variáveis extras
ansible-playbook playbook.yml -v               # mais detalhes (-vv, -vvv)
ansible-playbook playbook.yml --syntax-check   # valida a sintaxe
```

---

## 🧩 Módulos mais usados

| Módulo | Para que serve |
|---|---|
| `ansible.builtin.apt` / `dnf` / `yum` | Gerenciar pacotes |
| `ansible.builtin.package` | Gerenciar pacotes de forma genérica |
| `ansible.builtin.service` / `systemd` | Gerenciar serviços |
| `ansible.builtin.copy` | Copiar arquivos para o host |
| `ansible.builtin.template` | Gerar arquivos a partir de templates Jinja2 |
| `ansible.builtin.file` | Criar pastas, links e ajustar permissões |
| `ansible.builtin.lineinfile` | Inserir ou alterar uma linha em arquivo |
| `ansible.builtin.user` / `group` | Gerenciar usuários e grupos |
| `ansible.builtin.command` / `shell` | Executar comandos |
| `ansible.builtin.git` | Clonar repositórios |
| `ansible.builtin.get_url` | Baixar arquivos |
| `ansible.builtin.unarchive` | Extrair arquivos compactados |
| `ansible.builtin.cron` | Gerenciar tarefas agendadas |
| `ansible.builtin.debug` | Exibir mensagens e variáveis |
| `ansible.builtin.wait_for` | Aguardar porta ou condição |
| `ansible.builtin.reboot` | Reiniciar o host |

> 💡 Consulte a documentação de qualquer módulo no terminal: `ansible-doc ansible.builtin.copy`

---

## 🔤 Variáveis

### No playbook

```yaml
- hosts: web
  vars:
    app_nome: meu-app
    app_porta: 8080

  tasks:
    - name: Exibir variáveis
      ansible.builtin.debug:
        msg: "{{ app_nome }} rodando na porta {{ app_porta }}"
```

### Em arquivos

```
group_vars/
  all.yml        # variáveis para todos os hosts
  web.yml        # variáveis do grupo "web"
host_vars/
  web1.yml       # variáveis do host "web1"
```

### Registrando o resultado de uma task

```yaml
- name: Verificar versão do Nginx
  ansible.builtin.command: nginx -v
  register: nginx_versao
  changed_when: false

- name: Mostrar a versão
  ansible.builtin.debug:
    var: nginx_versao.stderr
```

### Facts

```yaml
- name: Mostrar sistema operacional
  ansible.builtin.debug:
    msg: "{{ ansible_facts['distribution'] }} {{ ansible_facts['distribution_version'] }}"
```

---

## 🔀 Condicionais e loops

### `when`

```yaml
- name: Instalar Nginx apenas em Debian/Ubuntu
  ansible.builtin.apt:
    name: nginx
    state: present
  when: ansible_facts['os_family'] == "Debian"
```

### `loop`

```yaml
- name: Instalar vários pacotes
  ansible.builtin.apt:
    name: "{{ item }}"
    state: present
  loop:
    - git
    - curl
    - htop
```

### Loop com dicionários

```yaml
- name: Criar usuários
  ansible.builtin.user:
    name: "{{ item.nome }}"
    shell: "{{ item.shell }}"
  loop:
    - { nome: ana, shell: /bin/bash }
    - { nome: joao, shell: /bin/zsh }
```

---

## 🔔 Handlers

Handlers só executam quando uma task **notifica** que houve mudança, ideal para reiniciar serviços.

```yaml
tasks:
  - name: Atualizar configuração do Nginx
    ansible.builtin.template:
      src: nginx.conf.j2
      dest: /etc/nginx/nginx.conf
    notify: Reiniciar Nginx

handlers:
  - name: Reiniciar Nginx
    ansible.builtin.service:
      name: nginx
      state: restarted
```

---

## 🧱 Templates (Jinja2)

`templates/nginx.conf.j2`:

```jinja
server {
    listen {{ app_porta }};
    server_name {{ ansible_facts['hostname'] }};

    location / {
        root {{ app_diretorio }};
        index index.html;
    }
}
```

Uso na task:

```yaml
- name: Gerar configuração do site
  ansible.builtin.template:
    src: nginx.conf.j2
    dest: /etc/nginx/sites-available/meu-site
    mode: "0644"
```

---

## 📦 Roles

Roles organizam o projeto em partes reutilizáveis.

```bash
ansible-galaxy init roles/nginx
```

Estrutura gerada:

```
roles/nginx/
├── defaults/main.yml    # variáveis padrão (menor precedência)
├── files/               # arquivos estáticos
├── handlers/main.yml    # handlers
├── meta/main.yml        # metadados e dependências
├── tasks/main.yml       # tarefas principais
├── templates/           # templates Jinja2
├── vars/main.yml        # variáveis da role
└── tests/
```

Usando a role em um playbook:

```yaml
- hosts: web
  become: true
  roles:
    - role: nginx
      vars:
        app_porta: 8080
```

Estrutura recomendada de projeto:

```
projeto/
├── ansible.cfg
├── inventory.ini
├── site.yml
├── group_vars/
├── host_vars/
└── roles/
    ├── common/
    ├── nginx/
    └── app/
```

---

## 🔐 Ansible Vault

Protege senhas, tokens e outros segredos criptografando arquivos ou valores.

```bash
# Criar um arquivo criptografado
ansible-vault create group_vars/all/vault.yml

# Editar
ansible-vault edit group_vars/all/vault.yml

# Criptografar um arquivo existente
ansible-vault encrypt secrets.yml

# Descriptografar
ansible-vault decrypt secrets.yml

# Visualizar
ansible-vault view secrets.yml

# Trocar a senha
ansible-vault rekey secrets.yml

# Criptografar apenas um valor
ansible-vault encrypt_string 'minha-senha' --name 'db_password'
```

Executando um playbook com Vault:

```bash
ansible-playbook site.yml --ask-vault-pass
# ou
ansible-playbook site.yml --vault-password-file ~/.vault_pass
```

> ⚠️ Nunca versione o arquivo com a senha do Vault.

---

## 🌍 Ansible Galaxy e Collections

```bash
# Instalar uma role da comunidade
ansible-galaxy role install geerlingguy.docker

# Instalar uma collection
ansible-galaxy collection install community.general

# Instalar tudo a partir de um arquivo de dependências
ansible-galaxy install -r requirements.yml
```

`requirements.yml`:

```yaml
---
roles:
  - name: geerlingguy.docker
collections:
  - name: community.general
  - name: ansible.posix
```

---

## 🧰 Comandos essenciais

| Comando | O que faz |
|---|---|
| `ansible all -m ping` | Testa a conexão com os hosts |
| `ansible <grupo> -m <módulo> -a "<args>"` | Executa um comando ad-hoc |
| `ansible-playbook <arquivo>` | Executa um playbook |
| `ansible-playbook <arquivo> --check --diff` | Simula e mostra diferenças |
| `ansible-inventory --list` | Exibe o inventário completo |
| `ansible-inventory --graph` | Exibe o inventário em árvore |
| `ansible-doc <módulo>` | Documentação de um módulo |
| `ansible-doc -l` | Lista todos os módulos |
| `ansible-galaxy init <role>` | Cria a estrutura de uma role |
| `ansible-galaxy install <role>` | Instala uma role |
| `ansible-vault <ação>` | Gerencia segredos |
| `ansible-config dump --only-changed` | Mostra configurações alteradas |
| `ansible-lint <playbook>` | Verifica boas práticas (requer `ansible-lint`) |
| `ansible --version` | Mostra a versão e caminhos de configuração |

---

## 🧪 Exemplo completo

Um playbook que prepara um servidor web com Nginx e publica uma página:

`inventory.ini`:

```ini
[web]
web1 ansible_host=192.168.56.11

[web:vars]
ansible_user=vagrant
```

`site.yml`:

```yaml
---
- name: Provisionar servidor web
  hosts: web
  become: true

  vars:
    app_porta: 80
    app_diretorio: /var/www/meu-site

  tasks:
    - name: Atualizar cache de pacotes
      ansible.builtin.apt:
        update_cache: true
        cache_valid_time: 3600

    - name: Instalar pacotes necessários
      ansible.builtin.apt:
        name:
          - nginx
          - git
          - curl
        state: present

    - name: Criar diretório do site
      ansible.builtin.file:
        path: "{{ app_diretorio }}"
        state: directory
        owner: www-data
        group: www-data
        mode: "0755"

    - name: Publicar página inicial
      ansible.builtin.copy:
        content: "<h1>Servidor configurado com Ansible 🚀</h1>"
        dest: "{{ app_diretorio }}/index.html"
        mode: "0644"

    - name: Gerar configuração do site
      ansible.builtin.template:
        src: site.conf.j2
        dest: /etc/nginx/sites-available/meu-site
        mode: "0644"
      notify: Recarregar Nginx

    - name: Ativar o site
      ansible.builtin.file:
        src: /etc/nginx/sites-available/meu-site
        dest: /etc/nginx/sites-enabled/meu-site
        state: link
      notify: Recarregar Nginx

    - name: Remover site padrão
      ansible.builtin.file:
        path: /etc/nginx/sites-enabled/default
        state: absent
      notify: Recarregar Nginx

    - name: Garantir que o Nginx está ativo
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: true

  handlers:
    - name: Recarregar Nginx
      ansible.builtin.service:
        name: nginx
        state: reloaded
```

`templates/site.conf.j2`:

```jinja
server {
    listen {{ app_porta }};
    server_name _;
    root {{ app_diretorio }};
    index index.html;
}
```

Execução:

```bash
ansible-playbook site.yml --syntax-check
ansible-playbook site.yml --check --diff
ansible-playbook site.yml
```

---

## 🩺 Solução de problemas

| Problema | Possível solução |
|---|---|
| `UNREACHABLE! Permission denied (publickey)` | Confira o usuário, a chave (`ansible_ssh_private_key_file`) e se a chave pública está no `authorized_keys` do host |
| `Failed to connect to the host via ssh` | Verifique IP, porta, firewall e se o SSH está ativo |
| `Missing sudo password` | Use `--ask-become-pass` ou configure `sudo` sem senha para o usuário |
| `/usr/bin/python: not found` | Instale o Python no host ou defina `ansible_python_interpreter=/usr/bin/python3` |
| Erro de *host key verification* | Adicione o host ao `known_hosts` ou, em laboratório, use `host_key_checking = False` |
| Erro de sintaxe YAML | Cheque a indentação (use espaços, nunca tabs) e rode `--syntax-check` |
| Task sempre mostra `changed` | Use módulos nativos em vez de `command`/`shell` ou defina `changed_when` |
| Variável indefinida | Verifique a precedência e use `default()`: `{{ var \| default('valor') }}` |
| Módulo não encontrado | Instale a collection necessária com `ansible-galaxy collection install` |

Para depurar:

```bash
ansible-playbook site.yml -vvv
```

---

## ✅ Boas práticas

- 📁 Organize o projeto com **roles**, `group_vars` e `host_vars`
- 🔁 Mantenha tudo **idempotente**; prefira módulos nativos a `shell`/`command`
- 🏷️ Dê **nomes descritivos** às tasks (`name:`) e use **tags**
- 🔐 Guarde segredos com **Ansible Vault**, nunca em texto puro
- 🧪 Sempre rode `--check --diff` antes de aplicar em produção
- 🧹 Use o `ansible-lint` para padronizar o código
- 🔢 **Fixe versões** de roles e collections no `requirements.yml`
- 🌿 Versione o projeto no Git e separe inventários por ambiente (dev, homologação, produção)
- 🔑 Use chaves SSH e o menor privilégio possível (`become` só quando necessário)

---

## 📚 Referências

- [Documentação oficial do Ansible](https://docs.ansible.com/)
- [Ansible Galaxy](https://galaxy.ansible.com/)
- [Índice de módulos](https://docs.ansible.com/ansible/latest/collections/index_module.html)
- [Ansible Lint](https://ansible.readthedocs.io/projects/lint/)

---

## Contatos

[![Github Badge](https://img.shields.io/badge/-Github-000?style=flat-square&logo=Github&logoColor=white&link=https://github.com/cleibp)](https://github.com/cleibp)
[![Linkedin Badge](https://img.shields.io/badge/-LinkedIn-blue?style=flat-square&logo=Linkedin&logoColor=white&link=https://www.linkedin.com/in/cleitonpaiva/)](https://www.linkedin.com/in/cleitonpaiva/)
[![Whatsapp Badge](https://img.shields.io/badge/-Whatsapp-4CA143?style=flat-square&labelColor=4CA143&logo=whatsapp&logoColor=white&link=https://api.whatsapp.com/send?phone=5516988368457&text=Ola!)](https://api.whatsapp.com/send?phone=5516988368457&text=Ola!)
[![Gmail Badge](https://img.shields.io/badge/-Gmail-c14438?style=flat-square&logo=Gmail&logoColor=white&link=mailto:cleibp@gmail.com)](mailto:cleibp@gmail.com)

[Pague-me um ☕](https://www.buymeacoffee.com/cleibp)

Feito com muito ❤️☕👨🏻‍💻 por Cleiton Paiva
