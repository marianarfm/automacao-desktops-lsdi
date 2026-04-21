# Automação de Desktops Ubuntu para o LSDi
Esse repositório implementa uma configuração Ansible para desktops Ubuntu do Laboratório de Sistemas Distribuídos (LSDi) com base em dois playbooks:
1. `playbooks/desktop.yml`
   Aplica o estado de software desejado a um host. Instala software ausente, atualiza pacotes gerenciados e configura software de fornecedores distribuído via APT, arquivos `.deb` diretos, arquivos compactados, instaladores de shell e ambientes virtuais Python
2. `playbooks/bootstrap_pull.yml`
   Instala o `ansible` e o `git` no host e configura um temporizador `systemd` para o `ansible-pull`, possibilitando a manutenção autônoma de desktops durante janelas de tempo agendadas

## Pacotes gerenciados
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

### Observações
As aplicações **Android Studio** e **MySQL Workbench Community** estão temporariamente desativadas do gerenciamento de pacotes
- O fluxo de download do Android Studio exige a aceitação dos termos do fornecedor
- O repositório APT da Oracle para o MySQL Workbench apresentou instabilidade no Ubuntu 24.04 durante os testes

Esses problemas dificultam a instalação/atualização automatizada por downloads HTTP

## Como implementar a configuração
### 1. Edite o inventário
Atualize `inventories/lab/hosts.yml` com os IPs ou nomes de host reais dos desktops e os usuários SSH usados ​​no ambiente

### 2. Revise o catálogo de pacotes
Se necessário, edite o arquivo `inventories/lab/group_vars/ubuntu_desktops.yml` para refletir a política atual do laboratório

Em particular, revise:
- listas de pacotes
- URLs de download do fornecedor
- pacotes do SDK do Android selecionados
- `desktop_pull_repo_url`

### 3. Teste a partir do nó de controle

Execute o playbook principal no grupo de desktops:

```bash
ansible-playbook playbooks/desktop.yml --limit ubuntu_desktops --ask-pass
```

Se você quiser fazer uma simulação (nenhum pacote será modificado):

```bash
ansible-playbook playbooks/desktop.yml --limit ubuntu_desktops --check --diff --ask-pass
```

Se você quiser testar em VMs com armazenamento limitado (instala o perfil de teste com apenas alguns pacotes):

```bash
ansible-playbook playbooks/desktop_vm_test.yml --limit ubuntu_desktops --ask-pass
```

### 4. Limitar a depuração de aplicativos arquivados

Para depurar aplicativos instalados por meio de arquivos compactados mais rapidamente, você pode limitar a execução a nomes de aplicativos específicos:

```bash
ansible-playbook playbooks/desktop.yml --limit ubuntu-vm-01 --ask-pass \
  -e '{"desktop_selected_archive_apps":["arduino-ide","intellij-idea-community"]}'
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

## Plano de testes sugerido
Etapas de validação recomendadas:
1. Execute `playbooks/desktop_vm_test.yml` em VMs pequenas ou recém-instaladas
2. Execute `playbooks/desktop.yml` em apenas uma VM inicialmente
3. Confirme se os principais aplicativos com interface gráfica abrem corretamente
4. Execute o mesmo playbook novamente e confirme se a segunda execução registrou a maioria das saídas como `ok`
5. Teste `playbooks/bootstrap_pull.yml` somente após o playbook do modo push estar estável

## Checklist
Estado atual do repositório:
- [x] A manutenção agendada do `ansible-pull` está ativa através do `systemd`
- [x] As atualizações do sistema Ubuntu podem ser executadas automaticamente através de `desktop_upgrade_all_system_packages`
- [x] Atualizações de software gerenciadas pelo APT são automáticas
- [x] Aplicativos Python em ambiente virtual são atualizados automaticamente
- [x] A instalação de pacotes do SDK do Android pela linha de comando é automatizada
- [ ] O Android Studio é totalmente automatizado no modo pull
- [ ] O MySQL Workbench Community é totalmente automatizado no modo pull
- [ ] Aplicativos instalados por arquivo são atualizados automaticamente sem a necessidade de editar variáveis ​​de versão
- [ ] Aplicativos instalados por shell são atualizados automaticamente sem a necessidade de editar variáveis ​​de versão
- [ ] Aplicativos `.deb` diretos são atualizados automaticamente sem a necessidade de editar variáveis ​​de versão
- [ ] Os demais aplicativos não-APT são totalmente automatizados, desde a instalação até as atualizações