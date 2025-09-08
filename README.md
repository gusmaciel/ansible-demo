````markdown
# 🛠️ Demonstração Guiada com Ansible + Role `webserver`

> **Objetivo:** montar e executar um playbook Ansible com a role `webserver`, que:
>
> 1. Atualiza os pacotes da VM
> 2. Instala pacotes via loop
> 3. Instala o Apache
> 4. Publica um `index.html` via template Jinja2
> 5. Reinicia o Apache via handler

---

## 🔧 1. Subir a VM (rápido)

Se a VM ainda não estiver ativa, em outro terminal:

```bash
cd ~/aulas/ansible-demo
vagrant init debian/bullseye64   # ou outra box Debian
vagrant up
````

Depois, pegue as informações necessárias:

```bash
vagrant ssh-config   # Mostra HostName, Port, IdentityFile
```

Anote:

* `HostName` (IP da VM)
* `User` (geralmente `vagrant`)
* `IdentityFile` (chave privada SSH)

Essas informações serão usadas no inventário do Ansible.

---

## 📁 2. Criar a estrutura do projeto

Execute no terminal (host):

```bash
mkdir -p ansible-demo/roles/webserver/{tasks,handlers,templates,vars,files}
cd ansible-demo
```

---

## 📦 3. Arquivo de inventário (`inventory`)

Crie o arquivo `ansible-demo/inventory` com o conteúdo:

```ini
[web]
192.168.56.10 ansible_user=vagrant ansible_private_key_file=/caminho/para/private_key
```

> ⚠️ No Windows, use o caminho completo da chave gerada pelo Vagrant, por exemplo:
> `C:\\Users\\SeuUsuario\\caminho\\ate\\.vagrant\\machines\\default\\virtualbox\\private_key`

---

## ⚙️ 4. `ansible.cfg` (opcional, mas recomendado)

Crie um arquivo `ansible.cfg` no diretório do projeto:

```ini
[defaults]
inventory = ./inventory
host_key_checking = False
retry_files_enabled = False
```

---

## 📜 5. Playbook principal (`playbook.yml`)

```yaml
- name: Provisionar webserver (role webserver)
  hosts: web
  become: true
  roles:
    - webserver
```

---

## 🔢 6. Variáveis da role (`roles/webserver/vars/main.yml`)

```yaml
pacotes_basicos:
  - htop
  - tree
  - unzip

usuario_dev: devops
senha_dev: "Senai@123"  # ⚠️ Apenas para demonstração. Use Vault em produção.
```

---

## 🔨 7. Tarefas da role (`roles/webserver/tasks/main.yml`)

```yaml
- name: Atualizar cache e pacotes
  apt:
    update_cache: yes
    upgrade: dist
  tags: always

- name: Instalar pacotes básicos (loop)
  apt:
    name: "{{ item }}"
    state: present
  loop: "{{ pacotes_basicos }}"

- name: Instalar Apache
  apt:
    name: apache2
    state: present

- name: Criar usuário devops
  user:
    name: "{{ usuario_dev }}"
    password: "{{ senha_dev | password_hash('sha512') }}"
    state: present
    shell: /bin/bash

- name: Publicar página inicial via template
  template:
    src: index.html.j2
    dest: /var/www/html/index.html
  notify: restart apache
```

---

## 🔁 8. Handler (`roles/webserver/handlers/main.yml`)

```yaml
- name: restart apache
  service:
    name: apache2
    state: restarted
```

---

## 🧩 9. Template HTML (`roles/webserver/templates/index.html.j2`)

```html
<!doctype html>
<html>
  <head>
    <meta charset="utf-8"/>
    <title>Servidor do {{ usuario_dev }}</title>
  </head>
  <body>
    <h1>Bem-vindo, {{ usuario_dev }}!</h1>
    <p>Pacotes instalados:</p>
    <ul>
    {% for p in pacotes_basicos %}
      <li>{{ p }}</li>
    {% endfor %}
    </ul>
    <p>Gerado por Ansible (template Jinja2).</p>
  </body>
</html>
```

---

## 🚀 10. Executar o playbook

### 1. Testar conectividade com o host remoto:

```bash
ansible all -m ping -i inventory -u vagrant --private-key=/caminho/para/private_key
```

Ou, se estiver usando `ansible.cfg`:

```bash
ansible web -m ping
```

✔️ Saída esperada:

```bash
192.168.56.10 | SUCCESS => {
    "ping": "pong"
}
```

---

### 2. Rodar o playbook:

```bash
ansible-playbook playbook.yml -i inventory -u vagrant --private-key=/caminho/para/private_key
```

✔️ Saída esperada (resumo):

```
PLAY RECAP *********************************************************************
192.168.56.10            : ok=6    changed=4    unreachable=0    failed=0
```

---

## ✅ Resultado

Após a execução:

* O Apache estará instalado e rodando.
* Um usuário `devops` será criado.
* A página padrão será publicada com base no template.
* O Apache será reiniciado automaticamente via handler, se necessário.

---

> 💡 **Dica final:**
> Para reutilizar esse projeto, basta alterar o inventário e ajustar as variáveis na role. Tudo é modular e reutilizável via Ansible.

---