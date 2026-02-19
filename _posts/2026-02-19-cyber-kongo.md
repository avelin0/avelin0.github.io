# Resumo dos Desafios CTF - Exercício Cyber Kongo

Resumo dos achados técnicos extraídos durante dois dias de operação do CTF, divididos entre caça a ameaças (*Threat Hunting*) e resposta a incidentes (*Incident Response*). 

## Dia 1: Threat Hunting (Rastreamento de Ameaças)

A primeira fase do exercício concentrou-se na análise de logs, tráfego de rede e artefatos de e-mail para reconstruir a cadeia de ataque inicial e a movimentação do adversário.

### 1. Exploração Inicial e Mineração de Criptomoedas
O ataque inicial caracterizou-se pela exploração da vulnerabilidade **React2Shell (CVE-2025-55182)**. Os logs do serviço `nodejs-debug.log` revelaram um *payload* que executou comandos remotos para baixar um minerador de criptomoedas (`xmrig`) hospedado no `github.com`. O adversário utilizou ferramentas como o **Chisel** para estabelecer túneis, o que foi identificado através de análise de conexões TCP e processos na máquina comprometida.

### 2. Vetor de Phishing e Comprometimento de E-mail
Os analistas investigaram caixas de correio e cabeçalhos MIME de e-mails, identificando uma campanha de *phishing* cujo assunto era *"Re: Network diagram you requested"*. O ataque utilizou:
*   Um servidor de e-mail local comprometido (`mail.jpufsp1.as.jp`) para distribuir o malware.
*   Um anexo HTML malicioso (`network-diagram.html`) que atuava como um *stager* para baixar um arquivo malicioso hospedado na URL `https://waichi-shop.jp/uploads/order.zip`.
*   Ataques bem-sucedidos a portais de *webmail*, além do roubo de arquivos de configuração de VPN (`h.tanaka-openvpn.config`) que continham a senha em texto claro (`enrage-upscale-budding`).

### 3. Comprometimento de Infraestrutura CI/CD e Automação
O atacante realizou milhares de tentativas de login (*brute force*) no GitLab, culminando em um acesso bem-sucedido seguido de navegação autenticada em projetos. O acesso indevido permitiu:
*   O roubo de *tokens* do GitLab (`glpat-9fA3XK2LmN7QwR8P`) extraídos através de injeções em URLs.
*   A exploração da plataforma de automação **n8n**, especificamente através da vulnerabilidade **CVE-2026-21858**.
*   A extração de senhas em *Base64* e chaves privadas SSH criptografadas (usuário `jh-private`) no banco de dados SQLite do n8n, permitindo que o atacante executasse comandos remotos nos servidores de deploy.

No fim desta fase, os logs revelaram que o atacante obteve acesso via chaves públicas SSH e configurou interfaces VPN do **WireGuard** para garantir persistência na rede.

---

## Dia 2: Incident Response (Resposta a Incidentes)

A segunda fase exigiu o uso intensivo de plataformas de agregação de logs (como o Graylog) e auditoria do Windows para rastrear o movimento lateral, a elevação de privilégios e as táticas de evasão de defesa dentro de um ambiente *Active Directory*.

### 1. Reconhecimento de Rede e Coleta de Credenciais
Os adversários usaram varreduras de rede clássicas para mapear o ambiente interno:
*   Execução de *scans* **TCP Connect via Nmap** realizados por contas não privilegiadas.
*   Consultas maliciosas aos Controladores de Domínio (identificados via LDAP no EventID 1644) e o uso da ferramenta **BloodHound** para mapear os caminhos de escalonamento de privilégios no domínio.
*   Procura ativa por arquivos críticos do sistema operacional Linux, como o `/etc/shadow` e o `.bash_history`.

### 2. Escalonamento de Privilégios e Movimento Lateral no Active Directory
Os logs de auditoria do Windows evidenciaram táticas avançadas de ataque focadas na infraestrutura Kerberos e certificados corporativos:
*   **Ataque ESC1:** O atacante forjou um certificado através do *Active Directory Certificate Services*, definindo manualmente o *User Principal Name* (UPN) no campo SAN (Subject Alternative Name) para se passar pela conta `Administrator@yachiyoni.local`.
*   **Golden Ticket:** Evidências apontaram para o comprometimento da conta `krbtgt`, permitindo a geração de *tickets* Kerberos falsificados.
*   Modificações em chaves de registro sensíveis, como `LocalAccountTokenFilterPolicy` (para habilitar a execução remota via SMB/WMI/PsExec) e `DisableRestrictedAdmin`.

### 3. Evasão de Defesas e Persistência
Para esconder seus rastros e manter o acesso, os invasores realizaram múltiplas ações destrutivas e alterações de segurança:
*   Desabilitação de recursos essenciais do **Microsoft Defender**, especificamente o `DisableRealtimeMonitoring` e `DisableBehaviorMonitoring`.
*   Uso de ferramentas para extrair as credenciais em memória do Windows (detectadas como `VirTool:Win32/DumpLsassProc.C` acessando o processo LSASS).
*   Criação de tarefas agendadas (como o processo "Updater" rodando no logon do usuário como "SYSTEM") e acesso via RDP com contas de administrador.
*   Limpeza de rastros em sistemas Linux, utilizando o comando `shred` para destruir arquivos de auditoria e de histórico do terminal (ex: `HISTFILE`, `.bash_history`, `wtmp`).
