# Resumo dos Desafios CTF - Exercício Cyber Kongo

<img width="335" height="407" alt="image" src="https://github.com/user-attachments/assets/ff710168-280f-4dad-a77b-2212e0a8c8da" />

---

# Resumo do Exercício Cyber Kongo (Dias 1, 2 e 3)

Este relatório compila os achados técnicos de três dias de operação, cobrindo desde a detecção inicial (*Threat Hunting*), passando pela resposta a incidentes no Active Directory (*Incident Response*) até a análise forense de rede e exfiltração de dados (*Response Actions*).

## Dia 1: Threat Hunting (Rastreamento de Ameaças)

A primeira fase focou na reconstrução da cadeia de ataque inicial (Kill Chain) através de logs de aplicação e servidores de e-mail.

### 1. Vetor Inicial e Exploração Web
O ataque iniciou-se com a exploração da vulnerabilidade **React2Shell (CVE-2025-55182)**. A análise do `nodejs-debug.log` revelou a injeção de comandos para baixar o minerador `xmrig` do GitHub. O atacante estabeleceu persistência e tunelamento utilizando a ferramenta **Chisel** (detectada via análise de portas e argumentos de linha de comando `socksr`).

### 2. Engenharia Social e Phishing
Foi identificada uma campanha de *phishing* com o assunto *"Re: Network diagram you requested"*. O e-mail continha um anexo HTML malicioso (`network-diagram.html`) que atuava como *stager* para baixar um arquivo ZIP de `waichi-shop.jp`. O ataque resultou no comprometimento do servidor de e-mail `mail.jpufsp1.as.jp` e no roubo de configurações de VPN (`h.tanaka-openvpn.config`) com senhas em texto claro.

### 3. Comprometimento de DevOps (GitLab/n8n)
O atacante realizou *brute force* no GitLab e explorou a vulnerabilidade **CVE-2026-21858** na plataforma de automação **n8n**. Isso permitiu a extração de chaves SSH privadas (usuário `jh-private`) e senhas criptografadas do banco de dados SQLite do n8n, garantindo acesso aos servidores de *deploy* e movimentação lateral.

---

## Dia 2: Incident Response (Resposta a Incidentes)

A segunda fase aprofundou-se na análise de sistemas Windows e Linux para conter o movimento lateral e a escalada de privilégios.

### 1. Reconhecimento e Movimentação Lateral
O adversário utilizou varreduras **TCP Connect** (Nmap) e a ferramenta **BloodHound** para mapear caminhos de ataque no Active Directory. Identificou-se o uso de contas comprometidas, como `redmine`, para autenticação LDAP e tunelamento.

### 2. Comprometimento do Active Directory
Houve um comprometimento crítico da infraestrutura de identidade:
*   **Ataque ESC1:** O atacante forjou certificados digitais definindo manualmente o SAN (Subject Alternative Name) para se passar por `Administrator@yachiyoni.local`.
*   **Golden Ticket:** Indícios de criação de tickets Kerberos falsos via comprometimento da conta `krbtgt`.
*   **Persistência:** Alteração de chaves de registro para permitir execução remota (`LocalAccountTokenFilterPolicy`) e desativação de monitoramento do Microsoft Defender.

### 3. Ações Destrutivas e Anti-Forense
O atacante tentou apagar seus rastros em sistemas Linux utilizando o comando `shred` em arquivos de histórico (`.bash_history`, `wtmp`) e utilizou ferramentas para extração de credenciais da memória LSASS (`VirTool:Win32/DumpLsassProc.C`).

---

## Dia 3: Response Action (Forense de Rede e Exfiltração)

A terceira e última fase concentrou-se na análise de tráfego de rede (PCAP) e forense avançada para identificar canais de comando e controle (C2) e recuperar dados exfiltrados.

### 1. Análise de C2 e Tráfego Malicioso
A análise com **Wireshark** e **Tshark** revelou uma infraestrutura de comunicação robusta do atacante:
*   **IPs Maliciosos:** Foram identificados os endereços de C2 `13.37.13.37`, `13.13.13.13` e `13.13.13.30`.
*   **Beaconing:** Detectou-se um padrão de comunicação (beacon) com intervalo médio de **300 segundos** para o IP `13.13.13.30` na porta 443.
*   **TLS Suspeito:** O tráfego criptografado revelou conexões para `g.api.mega.co.nz` durante o handshake TLS, sugerindo uso de armazenamento em nuvem para download de ferramentas ou exfiltração.

### 2. Recuperação de Dados Exfiltrados (DNS Tunneling)
O atacante utilizou **DNS Tunneling** (consultas do tipo TXT) para exfiltrar dados sensíveis de forma furtiva para o domínio `git-update-srv.net`.
*   **Técnica:** Os analistas extraíram *payloads* em Base64 das respostas DNS (flags `dns.flags.response == 0`).
*   **Desofuscação:** Os dados recuperados estavam protegidos por uma cifra XOR. Foi necessário desenvolver um script em Python para realizar *brute-force* na chave XOR (de 0 a 255) e decifrar o conteúdo.
*   **Achados:** A exfiltração continha o arquivo `backup_2024_12.zip` e a senha crítica **`Y@mam0t0CEO2024!`**.

### 3. Forense de Host e Atribuição
A investigação final nos artefatos de disco e logs de autenticação permitiu mapear as ferramentas e a identidade do atacante:
*   **Ferramentas:** Identificação do script malicioso `agent.sh`, arquivos de configuração `john_13.37.13.37.cfg` e uso do usuário `joker`.
*   **Exfiltração via FTP:** Além do DNS, houve exfiltração via FTP (comando `STOR`) utilizando as credenciais `ftp:3xf1ltr4t0rP@ssw0rXD`.
*   **Identidade SSH:** A análise do `auth.log` e do histórico do bash revelou uma conexão SSH para `root@156.206.278.135`. Foi possível recuperar a impressão digital (fingerprint) da chave SSH do atacante: **SHA256:Unb5aUxMXi+xgT0oUzVb8Lm04OX01tPgqTIHaYkaqGA**, associada ao identificador `satoshi.nakamoto@home-pc`.

---

### Observação sobre IA e Simulador de CIF
Reitero que, mesmo com a inclusão dos documentos do Dia 3, **não foram encontradas menções** ao uso de "Simulador de CIF" ou "Otimização via Inteligência Artificial". O exercício seguiu um fluxo estritamente técnico de análise manual e uso de scripts (Python/Bash) para automação de tarefas forenses específicas, como a decodificação XOR.
