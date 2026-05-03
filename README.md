# Automação para Desktops do LSDi

<div align="center">
  <img src="assets/lsdi_logo.png" alt="Logo do LSDi">
</div>

Esse repositório implementa uma configuração Ansible para desktops Ubuntu do Laboratório de Sistemas Distribuídos Inteligentes (LSDi). O objetivo é:

- definir um estado desejado de software para os desktops
- aplicar esse estado inicialmente a partir de um nó de controle
- depois manter os hosts atualizados de forma autônoma com `ansible-pull`

O fluxo do projeto é baseado em dois playbooks:

1. `playbooks/desktop.yml`
   
   Aplica o estado de software desejado a um host. Instala software ausente, atualiza pacotes gerenciados e configura software de fornecedores distribuído via APT, arquivos `.deb` diretos, arquivos compactados, instaladores de shell e ambientes virtuais Python

2. `playbooks/bootstrap_pull.yml`
   
   Instala o `ansible` e o `git` no host e configura um temporizador `systemd` para o `ansible-pull`, possibilitando a manutenção autônoma de desktops durante janelas de tempo agendadas

## Software gerenciado
- Desenvolvimento mobile
  - Android Studio
  - Ferramentas de linha de comando do Android SDK e pacotes selecionados do SDK
- Linguagens e toolchains
  - Python
  - OpenJDK 17
  - Node.js e npm
  - GCC e G++
  - .NET SDK 8.0
- IDEs
  - IntelliJ IDEA Community
  - PyCharm Community
  - CLion
  - Eclipse IDE
  - VS Code
  - Arduino IDE
- Bancos de dados e ferramentas relacionadas
  - PostgreSQL
  - pgAdmin 4
  - MySQL Server
  - MySQL Workbench Community
  - MongoDB Community
- Outras ferramentas
  - Docker
  - Esper
  - Postman
  - Anaconda
  - Jupyter Notebook
  - Git
  - TeXstudio
  - Pacotes TeX Live relacionados ao LaTeX e Beamer
  - MQTTX
  - LibreOffice
  - Google Chrome
  - Krita

## Configuração inicial
### 1. Prepare o nó de controle
Use qualquer máquina Linux que tenha `ansible` e conectividade SSH com os desktops. Para futuros mantenedores, o fluxo recomendado é:
- fazer um fork deste repositório
- clonar o fork localmente no nó de controle
- editar o repositório localmente com as variáveis, hosts e políticas do seu ambiente
- alterar `desktop_pull_repo_url` em `inventories/lab/group_vars/ubuntu_desktops.yml` para apontar para o seu fork público, por exemplo:
  
  ```yml
  desktop_pull_repo_url: https://github.com/seu-usuario/seu-fork-desse-repositorio.git
  ```
- fazer commit e push das mudanças no seu próprio fork

Depois que `playbooks/bootstrap_pull.yml` for executado, os hosts passarão a buscar exatamente o repositório configurado em `desktop_pull_repo_url`. Portanto, se você será o mantenedor desse ambiente, os hosts devem depender do seu fork e não deste repositório original

### 2. Edite o inventário
Atualize `inventories/lab/hosts.yml` com:
- IPs ou nomes dos desktops
- usuário SSH de cada host

### 3. Revise o catálogo de software
Se necessário, edite `inventories/lab/group_vars/ubuntu_desktops.yml`. Em especial, revise:
- `desktop_pull_repo_url`
- `desktop_pull_repo_branch`
- listas de pacotes APT
- versões, URLs e checksums dos aplicativos não-APT mantidos manualmente
- pacotes selecionados do Android SDK

### 4. Aplique o estado inicial a partir do nó de controle
Execute o playbook principal no grupo de desktops:
```bash
ansible-playbook playbooks/desktop.yml --limit ubuntu_desktops --ask-pass
```
**Comandos opcionais:**

Para fazer uma simulação (nenhum pacote será modificado):
```bash
ansible-playbook playbooks/desktop.yml --limit ubuntu_desktops --check --diff --ask-pass
```

Para testar em VMs com armazenamento limitado (instala o perfil de teste com apenas alguns pacotes):
```bash
ansible-playbook playbooks/desktop_vm_test.yml --limit ubuntu_desktops --ask-pass
```

Para depurar aplicativos instalados por meio de arquivos compactados mais rapidamente (limita a execução a nomes de aplicativos específicos):
```bash
ansible-playbook playbooks/desktop.yml --limit ubuntu-vm-01 --ask-pass -e '{"desktop_selected_archive_apps":["arduino-ide","intellij-idea-community"]}'
```

### 5. Habilitar a manutenção do modo pull
Após a configuração base estar funcionando como desejado, inicialize o `ansible-pull`:

```bash
ansible-playbook playbooks/bootstrap_pull.yml --limit ubuntu_desktops --ask-pass
```

Isso instala e habilita:

- `ansible-pull.service`
- `ansible-pull.timer`
- `/usr/local/sbin/ansible-pull-run`

## Como atualizar aplicativos depois
Para aplicativos mantidos por APT, normalmente não é necessário editar versões manualmente. Basta manter os repositórios configurados e os hosts se atualizam sozinhos no próximo `ansible-pull`

Para aplicativos não-APT com versões explícitas, a manutenção normal é:

1. editar `inventories/lab/group_vars/ubuntu_desktops.yml`
2. atualizar versão, URL, nome de arquivo, checksum ou diretório de release conforme o app
3. fazer commit e push para o branch usado em `desktop_pull_repo_branch`
4. aguardar o próximo `ansible-pull` dos hosts ou dispará-lo manualmente

## Como remover aplicativos
Para remover um aplicativo gerenciado por este repositório, a forma recomendada é manter a entrada no inventário e marcar o app como desabilitado primeiro. No próximo `ansible-pull`, o host removerá o aplicativo. Só depois dessa execução é seguro apagar a entrada do repositório

Fluxo recomendado:
1. altere o app desejado na seção `desktop_managed_apps_enabled` do inventário para `false`
2. faça commit e push
3. aguarde a próxima execução do `ansible-pull` nos hosts
4. confirme que o aplicativo foi removido
5. só então remova a entrada definitivamente do repositório, se desejar

Observações:
- todos os itens listados na seção de [software gerenciado](#software-gerenciado) devem expor um caminho de manutenção via `enabled`, mesmo quando forem pacotes APT
- para apps não-APT, `enabled: false` agora significa remover launchers, links e diretórios instalados por este projeto
- para os apps APT controlados em `desktop_optional_apt_apps`, `enabled: false` remove o pacote na próxima execução

### O que editar em cada app com manutenção manual
Android Studio
- `desktop_android_studio.version`
- `desktop_android_studio.build`
- `desktop_android_studio.linux_archive_file`
- `desktop_android_studio.linux_checksum`

Anaconda
- `desktop_anaconda.version`

MQTTX
- `desktop_mqttx.version`

Eclipse IDE
- `desktop_eclipse_ide.release`
- `desktop_eclipse_ide.linux_archive_file`

CLion
- `desktop_archive_apps` -> entrada `clion`:
  `url`, `archive_file`, `extracted_dir_name`

Arduino IDE
- `desktop_archive_apps` -> entrada `arduino-ide`:
  `url`, `archive_file`, `extracted_dir_name`

IntelliJ IDEA Community
- `desktop_archive_apps` -> entrada `intellij-idea-community`:
  `url`, `archive_file`, `extracted_dir_name`

PyCharm Community
- `desktop_archive_apps` -> entrada `pycharm-community`:
  `url`, `archive_file`, `extracted_dir_name`

Esper
- `desktop_archive_apps` -> entrada `esper`:
  `url`, `archive_file`, `extracted_dir_name`

Android SDK cmdline-tools
- `desktop_android_sdk.cmdline_tools_version`
- `desktop_android_sdk.cmdline_tools_url`, se necessário
- `desktop_android_sdk.cmdline_tools_filename`
- `desktop_android_sdk.cmdline_tools_release_dir`
- `desktop_android_sdk.cmdline_tools_checksum`
- `desktop_android_sdk.packages`, se quiser alterar os componentes instalados
