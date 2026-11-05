<div align="center">

# logcentralizer

**Centralizador de logs para Linux com linha de base, detecção de anomalias, sugestão de soluções e backup íntegro.**

[![Go](https://img.shields.io/badge/Go-1.24%2B-00ADD8?logo=go&logoColor=white)](https://go.dev/)
[![Ubuntu](https://img.shields.io/badge/Ubuntu-26.04%20LTS-E95420?logo=ubuntu&logoColor=white)](https://ubuntu.com/)
[![Plataforma](https://img.shields.io/badge/arquitetura-amd64%20%7C%20arm64-informational)](#-requisitos)
[![Dependências](https://img.shields.io/badge/depend%C3%AAncias-zero-success)](#-arquitetura)
[![Licença](https://img.shields.io/badge/licen%C3%A7a-MIT-blue)](#-licença)

[Visão geral](#-visão-geral) •
[Instalação](#-instalação) •
[Uso](#-uso) •
[Configuração](#%EF%B8%8F-configuração) •
[Detecções](#-o-que-é-detectado) •
[Backups](#-backups) •
[FAQ](#-solução-de-problemas)

</div>

---

## 📌 Visão geral

O **logcentralizer** é uma ferramenta de linha de comando, escrita em Go, que reúne **todos os principais
logs do sistema operacional numa única análise**. Ele aprende o comportamento habitual da máquina
(a *linha de base*) e indica com clareza o que fugiu do padrão.

Cada execução de `logcentralizer analisar` segue este fluxo:

```
┌──────────────┐   ┌──────────────────┐   ┌───────────────────┐   ┌──────────────────┐   ┌─────────────┐
│ 22 coletores │──▶│ Comparação com a │──▶│ Anomalias +       │──▶│ Relatório TXT +  │──▶│ Backup      │
│ em paralelo  │   │ linha de base    │   │ soluções (local e │   │ histórico das    │   │ .tar.gz     │
│ (logs, /proc,│   │ (o "habitual")   │   │ pesquisa na web)  │   │ análises         │   │ + SHA-256   │
│ hardware...) │   └──────────────────┘   └───────────────────┘   └──────────────────┘   └─────────────┘
└──────────────┘
```

### Destaques

- 🔍 **Visão completa.** Cobre journald/systemd, syslog, autenticação (SSH, sudo/sudo-rs, su, PAM), kernel,
  logins (wtmp/wtmpdb, btmp, lastlog), apt/dpkg, UFW, auditd, AppArmor, cron e timers, fail2ban,
  Nginx, Apache, Docker, MySQL/MariaDB e PostgreSQL. Inclui também **hardware** (temperatura, SMART,
  ECC, bateria), **processos do sistema**, **integridade de arquivos críticos** e **logs de aplicações
  próprias**.
- 📈 **Linha de base estatística.** Compara cada hora com a mesma hora do dia em semanas anteriores,
  guarda modelos de mensagens de erro e mantém o inventário de IPs, usuários, portas, executáveis e
  dispositivos USB já vistos.
- 🚨 **Mais de 80 regras de detecção.** Entre elas: força bruta SSH seguida de login bem-sucedido,
  conta com UID 0, malware *fileless* (memfd), logs adulterados, varredura de portas, ataques web,
  falhas de disco e de hardware.
- 🛠️ **Soluções acionáveis.** Cada anomalia vem com passos e comandos de correção e referências
  oficiais, mais uma pesquisa na internet (DuckDuckGo sem chave ou Brave Search API).
- 🧾 **Relatório em português** que explica **por que** algo é anômalo, com evidências, veredito e
  tendência das últimas análises.
- 💾 **Backup completo a cada análise**, com manifesto SHA-256, verificação de integridade e
  criptografia **AES-256-GCM** opcional.
- 🧰 **Verificação de ferramentas.** Detecta o que está ou não instalado e instala o que falta com um
  comando.
- 📦 **Binário estático único**, sem dependências externas, só com a biblioteca padrão do Go.

---

## 📋 Requisitos

| Item | Requisito |
|---|---|
| Sistema operacional | **Ubuntu 26.04 LTS** (alvo principal). Também funciona no 22.04/24.04 e em distribuições com systemd |
| Arquitetura | `amd64` (x86-64) ou `arm64` |
| Permissão | `root` (todos os comandos de análise usam `sudo`) |
| Go | Apenas para compilar o código-fonte: **Go 1.24+** (o Ubuntu 26.04 traz o Go 1.26 no pacote `golang-go`) |
| Internet | Opcional. Usada somente para pesquisar soluções |

---

## 🚀 Instalação

### Opção 1: instalador automático (recomendado)

```bash
git clone https://github.com/SEU-USUARIO/logcentralizer.git
cd logcentralizer
sudo ./scripts/install.sh
```

> Se você baixou o `.zip`, basta rodar `unzip logcentralizer-1.0.0.zip && cd logcentralizer`.

O instalador:

1. detecta a arquitetura e instala o binário pré-compilado em `/usr/local/bin/logcentralizer`;
2. cria `/etc/logcentralizer/` (configuração), `/var/lib/logcentralizer/` (linha de base, histórico e
   relatórios) e `/var/backups/logcentralizer/` (backups);
3. gera a configuração padrão em `/etc/logcentralizer/config.json`;
4. **verifica as ferramentas de log** e oferece instalar as que faltam (rsyslog, auditd, ufw,
   fail2ban, smartmontools, acct…);
5. oferece ativar o firewall UFW com registro de bloqueios e regras básicas de auditoria;
6. pergunta se os backups devem ser **criptografados**.

| Opção | Efeito |
|---|---|
| `--sim` | responde "sim" a todas as perguntas (instalação completa e não interativa) |
| `--nao` | instala apenas o logcentralizer, sem pacotes extras |
| `--compilar` | compila a partir do código-fonte em vez de usar o binário pronto |

### Opção 2: instalação manual

```bash
# Binários (use linux-arm64 em máquinas ARM)
sudo install -m 0755 bin/linux-amd64/logcentralizer           /usr/local/bin/
sudo install -m 0755 bin/linux-amd64/logcentralizer-simulador /usr/local/bin/

# Configuração e diretórios
sudo logcentralizer config init
sudo install -d -m 0700 /var/lib/logcentralizer /var/backups/logcentralizer

# Ferramentas de monitoramento
sudo logcentralizer verificar             # mostra o que falta
sudo logcentralizer verificar --instalar  # instala o que falta
```

### Opção 3: compilar a partir do código-fonte

```bash
sudo apt install golang-go make
make test     # testes unitários
make build    # gera build/logcentralizer e build/logcentralizer-simulador
make dist     # gera binários estáticos amd64 e arm64 em bin/
sudo ./scripts/install.sh --compilar
```

### Ferramentas recomendadas

Quanto mais fontes ativas, mais completa a análise:

```bash
sudo apt install rsyslog auditd ufw fail2ban smartmontools acct apparmor apparmor-utils lm-sensors rasdaemon
sudo ufw enable && sudo ufw logging on
sudo systemctl enable --now auditd acct fail2ban smartmontools rsyslog
```

| Pacote | O que adiciona à análise |
|---|---|
| `rsyslog` | arquivos de texto `syslog`, `auth.log`, `kern.log` e `ufw.log` |
| `auditd` | auditoria do kernel (há regras sugeridas em `configs/auditd-logcentralizer.rules`) |
| `ufw` | registro de bloqueios e detecção de varredura de portas |
| `fail2ban` | bloqueio automático de força bruta |
| `smartmontools` | saúde dos discos (SMART/NVMe) |
| `acct` | registro de todos os comandos executados |
| `lm-sensors`, `rasdaemon` | sensores e erros de hardware (MCE/ECC) |

---

## 💻 Uso

### Primeiros passos

```bash
sudo logcentralizer verificar          # 1. confere fontes e ferramentas
sudo logcentralizer analisar           # 2. executa a análise
sudo logcentralizer relatorio | less   # 3. lê o relatório completo
```

**Primeira análise.** Os logs antigos que já estão na máquina (até 14 dias) são usados só para
**aprender o habitual**. Em seguida são analisadas as últimas 24 horas.

**Análises seguintes.** Cobrem o período **desde a última análise até agora**.

**Pouco histórico.** Enquanto houver menos de 24 horas de atividade registradas, a ferramenta
trabalha em **modo aprendizado**: as regras de limite e assinatura continuam ativas, mas as regras de
novidade e de volume ficam desligadas.

### Exemplo de saída

```text
$ sudo logcentralizer analisar
logcentralizer 1.0.0 — análise em desktop
• Analisando 16 fontes de 04/11/2026 23:12 até 05/11/2026 03:12...
• Pesquisando soluções na internet para as anomalias mais graves...
• Criando backup de todos os logs...

======================================================================
 VEREDITO: ANORMAL
 Eventos analisados: 1172 | Anomalias: CRÍTICA 2, ALTA 18, MÉDIA 13, BAIXA 4
======================================================================
  [CRITICA] LOGIN SSH BEM-SUCEDIDO após força bruta de 203.0.113.77
  [CRITICA] Conta criada com UID 0 (privilégio de root): backdoor
  [ALTA   ] Varredura de portas vinda de 45.33.32.156
  [ALTA   ] Serviço reiniciando em loop: minhaapi.service
  [ALTA   ] Pico de volume de eventos na fonte syslog
  [ALTA   ] Arquivo de log encolheu sem rotação: /var/log/mysql/error.log
  ... e mais 31 no relatório

 Relatório completo: /var/lib/logcentralizer/relatorios/relatorio_20261105_031207.txt
 Backup: /var/backups/logcentralizer/logcentralizer_desktop_20261105_031207.tar.gz (143.6 KB)
```

### Referência de comandos

| Comando | Descrição |
|---|---|
| `logcentralizer analisar` | Análise completa: coleta, compara com a linha de base, pesquisa soluções, gera o relatório e o backup |
| `logcentralizer verificar [--instalar] [-y]` | Verifica e instala as ferramentas de cada fonte |
| `logcentralizer fontes` | Lista as fontes e como cada uma é lida |
| `logcentralizer relatorio [arquivo] [--lista]` | Exibe o último relatório, um relatório específico ou a lista de relatórios |
| `logcentralizer historico [-n 20]` | Resumo das análises anteriores |
| `logcentralizer backup listar \| verificar \| criar` | Gerencia os backups |
| `logcentralizer backup descriptografar <arq.enc> [--saida x.tar.gz]` | Abre um backup criptografado |
| `logcentralizer linha-de-base status \| resetar` | Mostra ou reinicia o aprendizado |
| `logcentralizer config init \| mostrar \| validar` | Cria, exibe ou valida a configuração |
| `logcentralizer config sensibilidade baixa\|media\|alta` | Ajusta a sensibilidade |
| `logcentralizer config criptografia ativar\|desativar` | Liga ou desliga a criptografia dos backups |
| `logcentralizer config brave-chave <CHAVE>\|remover` | Configura a Brave Search API |
| `logcentralizer ajuda` / `versao` | Ajuda e versão |

**Opções de `analisar`:**

| Opção | Efeito |
|---|---|
| `--fontes auth,kernel,ufw` | analisa apenas as fontes indicadas |
| `--periodo 6h` | força o período analisado (`30m`, `6h`, `3d`) |
| `--sem-backup` | não gera backup nesta execução |
| `--sem-internet` | usa apenas a base local de soluções |
| `--imprimir` | exibe o relatório completo no terminal |
| `--codigo-saida` | retorna `2` (ATENÇÃO) ou `3` (ANORMAL), útil em scripts |
| `--verbose` | mostra detalhes da coleta e das pesquisas |
| `--config <arquivo>` | usa outro arquivo de configuração (opção global) |

### Estrutura do relatório

O relatório fica em `/var/lib/logcentralizer/relatorios/`, e `ultimo_relatorio.txt` sempre aponta para
o mais recente. Ele tem oito seções:

1. **Veredito:** `NORMAL`, `ATENÇÃO` ou `ANORMAL`, com a contagem de anomalias por severidade.
2. **Anomalias detectadas.** Para cada uma:
   - regra, fontes, período e número de ocorrências;
   - **por que é anomalia** (comparação com a linha de base);
   - evidências reais do log;
   - **solução sugerida** com comandos e referências;
   - **links da pesquisa na internet**.
3. **Fontes monitoradas:** eventos por hora no período atual comparados com o habitual, com a
   variação em %.
4. **Estado atual da máquina:** discos, memória, carga, temperatura, SMART e processos.
5. **Ferramentas de monitoramento:** o que falta instalar.
6. **Histórico das últimas N análises,** com tendência (piorou, melhorou ou estável) e anomalias
   recorrentes.
7. **Backup:** arquivo, tamanho, SHA-256 e criptografia.
8. **Linha de base:** o que já é considerado habitual.

Severidades, da mais grave para a menos grave: `CRITICA` › `ALTA` › `MEDIA` › `BAIXA` › `INFO`.

### Execução agendada (opcional)

A ferramenta foi feita para execução manual. Se quiser automatizar, use o cron do root:

```bash
sudo crontab -e
# Todos os dias às 12h30:
30 12 * * * /usr/local/bin/logcentralizer analisar >/dev/null 2>&1
```

---

## ⚙️ Configuração

Toda a personalização fica em **`/etc/logcentralizer/config.json`**. Chaves omitidas usam o valor
padrão, e há um exemplo completo em [`configs/config.exemplo.json`](configs/config.exemplo.json).
Depois de editar, valide com `sudo logcentralizer config validar`.

```jsonc
{
  "linha_de_base": { "dias": 14, "horas_minimas": 24 },
  "deteccao": {
    "sensibilidade": "media",                              // baixa | media | alta
    "horario_login": { "inicio": "07:00", "fim": "23:59", "dias_semana": [1,2,3,4,5] },
    "janela_manutencao_pacotes": { "inicio": "08:00", "fim": "22:00" },
    "forca_bruta_ssh": { "falhas": 5, "janela_minutos": 10 },
    "regras_desativadas": ["bateria_degradada"],
    "severidade_personalizada": { "sudo_novo_comando": "INFO" }
  },
  "lista_branca": {
    "ips": ["127.0.0.1", "::1", "192.168.1.0/24"],
    "usuarios": [], "processos": [], "portas": [], "pacotes": [],
    "padroes_regex": ["gnome-shell.*Window manager warning"]
  },
  "fontes": {
    "nginx": { "ativo": false },                           // desativar uma fonte
    "auth":  { "modo": "journal" }                         // auto | arquivo | journal
  },
  "fontes_personalizadas": [
    { "nome": "minha_api", "caminho": "/var/log/minha_api/*.log", "formato": "json",
      "campo_tempo": "time", "campo_mensagem": "msg", "campo_nivel": "level" }
  ],
  "busca_web": { "ativo": true, "provedores": ["brave", "duckduckgo"], "max_buscas_por_execucao": 8 },
  "backup": { "retencao_dias": 0, "criptografia": { "ativo": false } },
  "relatorio": { "analises_no_historico": 10 }
}
```

> O arquivo real é JSON puro. Os comentários `//` acima são só explicativos.

### Sensibilidade do detector de volume

Uma hora é considerada anômala quando duas condições valem ao mesmo tempo:

```
eventos > máx(média + Z × desvio-padrão, média × fator)
eventos ≥ mínimo
```

| Nível | Z | Mínimo de eventos | Fator |
|---|:---:|:---:|:---:|
| `baixa` | 4 | 50 | 5× |
| `media` *(padrão)* | 3 | 20 | 3× |
| `alta` | 2 | 10 | 2× |

Cada valor pode ser ajustado individualmente em `deteccao.sensibilidade_personalizada`.

### Logs de aplicações próprias

Os formatos suportados são `texto` (data ISO no início da linha), `json` (uma linha por evento),
`regex` (grupos nomeados `time`, `level` e `message`) e `syslog`. Essas fontes ganham automaticamente
detecção de pico de volume, de silêncio, de erros nunca vistos e de truncamento.

> A referência completa de **todas** as chaves está no [TUTORIAL.md](TUTORIAL.md#8-personalização-configjson).

---

## 🌐 Pesquisa de soluções na internet

Toda anomalia recebe uma **solução da base local**, que funciona sem internet. As anomalias a partir
da severidade `MEDIA` também recebem **links pesquisados na internet**, que ficam em cache por 7 dias.

| Provedor | Chave | Observação |
|---|---|---|
| DuckDuckGo | não precisa | Padrão. Pode limitar consultas automáticas |
| Brave Search API | precisa | Mais estável. É tentado primeiro quando há chave |

**Como obter a chave da Brave:**

1. Crie uma conta em **https://api-dashboard.search.brave.com** e confirme o e-mail.
2. Ative o plano **Search**. Ele inclui cerca de **US$ 5 de crédito grátis por mês (≈ 1.000 consultas)**
   e a Brave pede um cartão de crédito para ativar. As condições podem mudar, então confira no painel.
3. Em **API Keys**, crie uma chave e copie o valor.
4. Grave a chave: `sudo logcentralizer config brave-chave SUA_CHAVE`.

---

## 💾 Backups

Ao final de **cada análise** é criado um backup completo em `/var/backups/logcentralizer/`. O backup
inclui:

- todo o `/var/log`, inclusive arquivos rotacionados;
- `/var/lib/lastlog` e `/var/crash`;
- a exportação em texto do journal;
- a linha de base, o histórico e os relatórios;
- um `MANIFESTO.sha256` com o hash de cada arquivo.

```text
logcentralizer_<host>_20261105_031207.tar.gz          # ou .tar.gz.enc (criptografado)
logcentralizer_<host>_20261105_031207.tar.gz.sha256   # hash do backup (compatível com sha256sum)
```

```bash
sudo logcentralizer config criptografia ativar     # AES-256-GCM + PBKDF2; gera a senha em /etc/logcentralizer/backup.key
sudo logcentralizer backup verificar               # confere a integridade de todos os backups
sudo logcentralizer backup descriptografar ARQ.enc --saida /tmp/restauro.tar.gz
mkdir -p /tmp/restauro && sudo tar -xzf /tmp/restauro.tar.gz -C /tmp/restauro
cd /tmp/restauro && sudo sha256sum -c MANIFESTO.sha256
```

> ⚠️ **Retenção padrão: manter sempre.** Cada backup é uma recapitulação completa, então o espaço usado
> cresce a cada análise. O backup é cancelado automaticamente se o espaço livre ficar abaixo de 10%.
> Para limitar a retenção, use `backup.retencao_dias` ou `backup.max_backups`.
>
> 🔑 **Guarde uma cópia de `/etc/logcentralizer/backup.key` fora da máquina.** Sem ela, os backups
> criptografados não podem ser abertos.

Os logs originais **nunca são apagados nem alterados** pelo logcentralizer.

---

## 🚨 O que é detectado

| Categoria | Exemplos de regras |
|---|---|
| **Acesso e autenticação** | força bruta SSH, **login bem-sucedido após força bruta**, login root via SSH, IP ou usuário nunca visto, login fora do horário, falhas de sudo/su, sudo sem permissão |
| **Contas e privilégios** | conta criada, **conta extra com UID 0**, usuário incluído em `sudo`/`docker`/`lxd`, conta de sistema com login |
| **Integridade** | alteração de `passwd`/`shadow`/`sudoers`/`sshd_config`/PAM, `authorized_keys` modificado, **`/etc/ld.so.preload`** (rootkit), log truncado ou apagado |
| **Processos** | **malware fileless (memfd)**, executável em `/tmp`/`/dev/shm`, binário apagado, novo programa como root, nova porta em escuta, ferramentas suspeitas (`nc`, `nmap`, `xmrig`…) |
| **Persistência** | novo comando, arquivo ou timer do cron/systemd |
| **Rede e firewall** | varredura de portas, UFW inativo, banimentos do fail2ban, ataques web (SQLi, LFI, scanners), picos de 404/5xx |
| **Sistema e kernel** | serviço com falha ou em loop, OOM, segfault, core dump, BUG/Oops/lockup, negações do AppArmor, anomalias do auditd |
| **Hardware e recursos** | erro de disco/EXT4/NVMe, SMART reprovado, setores realocados, erros ECC, temperatura alta, disco/inodes/memória/swap no limite, bateria degradada, USB desconhecido |
| **Linha de base** | pico de volume por fonte, fonte de log silenciosa, mensagem de erro nunca vista |
| **Pacotes, containers e bancos** | instalação fora da janela de manutenção, ferramenta ofensiva instalada, remoção de ferramenta de segurança, container caindo ou em OOM, força bruta e falhas críticas em MySQL/PostgreSQL |

A lista completa de regras, com a severidade padrão de cada uma, está no
[TUTORIAL.md § 11](TUTORIAL.md#11-regras-de-detecção).

---

## 🧪 Teste com ataques simulados

Valide a instalação **sem tocar nos logs reais**:

```bash
sudo ./scripts/teste_simulado.sh
```

O script:

1. gera 14 dias de logs de um desktop comum e executa a primeira análise, que deve resultar em
   **NORMAL** (sem falsos positivos);
2. injeta mais de 15 incidentes: força bruta seguida de invasão, conta com UID 0, varredura de portas,
   OOM, erro de disco, ataque web, remoção do auditd, log adulterado e outros;
3. executa a segunda análise, que deve resultar em **ANORMAL**, com cerca de 37 anomalias.

O teste usa uma pasta e uma configuração próprias. Para limpar depois: `sudo rm -rf /tmp/logcentralizer-teste`.

---

## 🏗 Arquitetura

```
logcentralizer/
├── cmd/
│   ├── logcentralizer/      # CLI
│   └── simulador/           # gerador de logs sintéticos para testes
├── internal/
│   ├── collectors/          # 22 coletores (interface Collector) — 1 por fonte
│   ├── analyzer/            # linha de base × período atual, regras, volume, silêncio, truncamento
│   ├── store/               # persistência JSON (linha de base, histórico, cache)
│   ├── solutions/           # base local de soluções + Brave/DuckDuckGo
│   ├── backup/              # tar.gz + manifesto SHA-256 + AES-256-GCM
│   ├── report/              # relatório TXT
│   ├── toolcheck/           # verificação de ferramentas instaladas
│   └── config/              # configuração com padrões e validação
├── scripts/                 # install.sh, uninstall.sh, teste_simulado.sh
├── configs/                 # config.exemplo.json, regras do auditd
└── bin/                     # binários estáticos linux-amd64 e linux-arm64
```

- **Coleta em fluxo.** Os coletores rodam em paralelo e enviam eventos normalizados por um canal. Um
  único consumidor aplica as regras sem carregar os logs inteiros na memória.
- **Escalável.** Para adicionar uma fonte nova, basta implementar a interface `Collector` e registrá-la
  em `collectors.All()`. Volume, silêncio, erro novo e truncamento passam a funcionar automaticamente.
  O guia está no [TUTORIAL.md § 13](TUTORIAL.md#13-arquitetura-e-como-adicionar-novas-fontes).
- **Sem dependências externas.** Usa apenas a biblioteca padrão do Go, e os dados ficam em JSON.

### Arquivos no sistema

| Caminho | Conteúdo |
|---|---|
| `/usr/local/bin/logcentralizer` | executável |
| `/etc/logcentralizer/config.json` | configuração (0600) |
| `/etc/logcentralizer/backup.key` | senha dos backups criptografados (0600) |
| `/var/lib/logcentralizer/` | `linha_de_base.json`, `historico.json`, `cache_busca.json`, `relatorios/` |
| `/var/backups/logcentralizer/` | backups e arquivos `.sha256` |

---

## 🩺 Solução de problemas

| Sintoma | Solução |
|---|---|
| `precisa ser executado como root` | Use `sudo` |
| Muitas fontes "ausente" | Normal para serviços não usados. Para as fontes de sistema, rode `sudo logcentralizer verificar --instalar` |
| Sempre em "modo aprendizado" | Ative o journal persistente (`sudo mkdir -p /var/log/journal && sudo systemctl restart systemd-journald`) ou reduza `linha_de_base.horas_minimas` |
| Muitos `erro_novo` no início | Esperado: cada modelo de erro alerta só na primeira vez. Ruídos conhecidos podem ir para `lista_branca.padroes_regex` |
| Muitos picos de volume | `sudo logcentralizer config sensibilidade baixa` |
| Alertas de login fora do horário | Ajuste `deteccao.horario_login` |
| Pesquisa web "não realizada" | Sem internet ou DuckDuckGo limitando. Configure a Brave ou use `--sem-internet` |
| `backup cancelado: apenas X% livre` | Libere espaço, mude `backup.diretorio` ou defina `backup.retencao_dias` |
| `já existe uma análise em andamento` | Aguarde a outra execução terminar. Se ela travou, remova `/run/logcentralizer.lock` |

Para diagnosticar, use `sudo logcentralizer analisar --verbose --sem-backup`.

---

## 🗑 Desinstalação

```bash
sudo ./scripts/uninstall.sh
```

O script remove os binários e **pergunta antes** de apagar a configuração, os dados e relatórios, os
backups e as regras de auditoria. Os pacotes do sistema (ufw, auditd etc.) são mantidos.

---

## 🤝 Contribuindo

1. Faça um *fork* e crie uma *branch*: `git checkout -b minha-melhoria`
2. Rode `make vet test` e `sudo ./scripts/teste_simulado.sh`
3. Toda regra nova precisa de uma solução em `internal/solutions/kb.go`. O teste `TestKBCoversRules`
   confere isso.
4. Abra um *Pull Request* descrevendo a mudança.

---

## 📄 Licença

Distribuído sob a licença **MIT**. Veja o arquivo [`LICENSE`](LICENSE).

<div align="center">

📚 Documentação completa: **[TUTORIAL.md](TUTORIAL.md)**

</div>
