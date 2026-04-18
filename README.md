# 🔐 Simulação de Ataques de Força Bruta com Medusa & Kali Linux

> **Projeto prático do curso de Cibersegurança — DIO**  
> Simulação controlada de ataques de força bruta em ambiente isolado, com documentação honesta dos resultados — incluindo limitações encontradas com as ferramentas.

## ⚠️ Aviso Legal

> Este projeto foi desenvolvido **exclusivamente para fins educacionais**, em ambiente totalmente isolado e controlado (laboratório virtual).  
> **Jamais execute essas técnicas em sistemas sem autorização explícita por escrito.**  
> O uso não autorizado de ferramentas de força bruta é **ilegal** em território brasileiro (Lei nº 12.737/2012 — Lei Carolina Dieckmann) e em praticamente todos os países.  
> O autor não se responsabiliza pelo uso indevido das informações aqui descritas.

---

## 📋 Índice

- [Sobre o Projeto](#-sobre-o-projeto)
- [Ambiente e Ferramentas](#-ambiente-e-ferramentas)
- [Configuração do Ambiente](#-configuração-do-ambiente)
- [Reconhecimento com Nmap](#-reconhecimento-com-nmap)
- [Cenário 1 — Força Bruta em FTP](#-cenário-1--força-bruta-em-ftp--sucesso)
- [Cenário 2 — Formulário Web DVWA](#-cenário-2--formulário-web-dvwa--limitação-do-medusa)
- [Cenário 3 — Password Spraying em SMB](#-cenário-3--password-spraying-em-smb--sucesso)
- [Wordlists Utilizadas](#-wordlists-utilizadas)
- [Resultados Consolidados](#-resultados-consolidados)
- [Aprendizados Reais](#-aprendizados-reais)
- [Medidas de Mitigação](#-medidas-de-mitigação)
- [Estrutura do Repositório](#-estrutura-do-repositório)
- [Referências](#-referências)

---

## 🎯 Sobre o Projeto

Este é meu **primeiro projeto prático** de Cibersegurança, desenvolvido como parte do desafio da **Formação Cibersegurança da DIO**. O objetivo é simular ataques de força bruta em um ambiente controlado e documentar os resultados de forma honesta — incluindo os momentos em que as ferramentas **não funcionaram como esperado**.

A proposta vai além de executar comandos: é entender o **porquê** de cada técnica, reconhecer as limitações reais das ferramentas e, principalmente, aprender a se **defender** desses ataques.

### O que foi praticado

- Reconhecimento de rede e serviços com `nmap`
- Força bruta em FTP com `Medusa`
- Tentativa de força bruta em formulário web (DVWA) — com análise da limitação encontrada
- Password Spraying em SMB com enumeração de usuários

> 💡 **Nota sobre honestidade técnica:** Nem tudo funcionou perfeitamente. No cenário do DVWA, o Medusa apresentou falsos positivos. Isso está documentado e analisado neste README porque **entender as limitações das ferramentas é tão importante quanto saber usá-las.**

---

## 🚀 Projeto em Destaque

Este projeto demonstra na prática:

- ✅ Execução de ataques de força bruta em FTP e SMB
- ⚠️ Identificação de limitações em ferramentas (Medusa no DVWA)
- 🔍 Validação manual de resultados suspeitos
- 📝 Documentação técnica estruturada e honesta

---

## 🛠️ Ambiente e Ferramentas

### Infraestrutura

| Máquina | Sistema Operacional | Função | IP Utilizado |
|---|---|---|---|
| VM 1 | Kali Linux 2024.x | Atacante | `192.168.56.100` |
| VM 2 | Metasploitable 2 | Alvo vulnerável | `192.168.56.101` |

> Ambas as VMs foram configuradas em **rede Host-Only** no VirtualBox — sem acesso à internet e sem risco de afetar outras máquinas.

### Ferramentas

| Ferramenta | Finalidade |
|---|---|
| **Nmap** | Escaneamento de rede e serviços |
| **Medusa** | Força bruta em múltiplos protocolos |
| **DVWA** | Aplicação web vulnerável para testes |
| **smbclient** | Validação de acesso SMB |
| **VirtualBox** | Virtualização das máquinas |

---

## ⚙️ Configuração do Ambiente

### Passo 1 — Configurar a Rede Host-Only no VirtualBox

A rede Host-Only cria uma rede privada entre as VMs e o host, sem acesso à internet.

1. Abra o VirtualBox → **Arquivo → Gerenciador de Rede Host**
2. Clique em **Criar** → uma rede `vboxnet0` será gerada
3. Certifique-se que o **Servidor DHCP está habilitado**
4. Em cada VM → **Configurações → Rede → Adaptador 1**
5. Selecione **"Placa de rede exclusiva de hospedeiro"** → escolha `vboxnet0`

### Passo 2 — Descobrir os IPs das Máquinas

**No Kali Linux:**
```bash
ip a
```

**No Metasploitable 2** (credenciais padrão: `msfadmin` / `msfadmin`):
```bash
ifconfig
```

### Passo 3 — Testar a Comunicação

Do terminal do Kali, confirme que as máquinas se comunicam:

```bash
ping -c 4 192.168.56.101
```

Se receber respostas, o ambiente está pronto. ✅

---

## 📡 Reconhecimento com Nmap

Antes de qualquer ataque, o passo obrigatório é o **reconhecimento** — entender quais serviços estão rodando no alvo. Sem isso, atacaríamos "às cegas".

### Comando utilizado

```bash
nmap -sV 192.168.56.101
```

> **Flags explicadas:**
> - `-sV` → detecta a **versão** dos serviços rodando em cada porta aberta

### Resultado obtido

```
PORT     STATE SERVICE     VERSION
21/tcp   open  ftp         vsftpd 2.3.4
22/tcp   open  ssh         OpenSSH 4.7p1
80/tcp   open  http        Apache httpd 2.2.8
139/tcp  open  netbios-ssn Samba smbd 3.X
445/tcp  open  microsoft-ds Samba smbd 3.X
3306/tcp open  mysql       MySQL 5.0.51a
...
```

![Reconhecimento com Nmap](images/recon/nmap_scan.png)

### O que esses resultados significam

O Metasploitable 2 é propositalmente vulnerável e expõe vários serviços. Para este projeto, focamos em três deles:

- **Porta 21 (FTP)** → transferência de arquivos, alvo do Cenário 1
- **Porta 80 (HTTP)** → onde o DVWA está rodando, alvo do Cenário 2
- **Porta 445 (SMB)** → compartilhamento de arquivos, alvo do Cenário 3

---

## 🔐 Cenário 1 — Força Bruta em FTP ✅ Sucesso

### O que é FTP e por que é vulnerável?

O **FTP (File Transfer Protocol)** é um protocolo antigo de transferência de arquivos que transmite dados — incluindo usuário e senha — **sem criptografia**. Quando um serviço FTP usa senhas fracas ou credenciais padrão, um ataque de força bruta pode descobri-las automaticamente.

### Objetivo

Descobrir credenciais válidas no serviço FTP do Metasploitable usando o Medusa com uma wordlist simples.

### Wordlists utilizadas

**`users.txt`**
```
root
admin
msfadmin
user
ftp
test
```

**`passwords.txt`**
```
123456
password
admin
msfadmin
root
toor
```

### Comando executado

```bash
medusa -h 192.168.56.101 -U users.txt -P passwords.txt -M ftp
```

> **Flags explicadas:**
> - `-h` → IP do host alvo
> - `-U` → arquivo com lista de usuários
> - `-P` → arquivo com lista de senhas
> - `-M ftp` → módulo do protocolo a testar

### Resultado

```
ACCOUNT FOUND: [ftp] Host: 192.168.56.101 User: msfadmin Password: msfadmin [SUCCESS]
```

✅ **Credencial encontrada:** `msfadmin` / `msfadmin`

![Medusa FTP - Credencial encontrada](images/ftp/medusa_ftp_result.png)

### Validação do acesso

```bash
ftp 192.168.56.101
# → Connected to 192.168.56.101
# → Name: msfadmin
# → Password: msfadmin
# → 230 Login successful.
```

✅ Acesso confirmado ao servidor FTP.

### Por que funcionou?

O Metasploitable 2 usa **credenciais padrão** (`msfadmin:msfadmin`) — exatamente o tipo de senha que está nas primeiras linhas de qualquer wordlist de ataques reais. Em ambientes de produção, isso seria uma vulnerabilidade crítica.

---

## 🌐 Cenário 2 — Formulário Web DVWA ⚠️ Limitação do Medusa

### O que é DVWA?

O **DVWA (Damn Vulnerable Web Application)** é uma aplicação web propositalmente vulnerável, usada em treinamentos de segurança. Ele já vem instalado no Metasploitable 2 e é acessado pelo navegador.

**Acesso:** `http://192.168.56.101/dvwa`  
**Credenciais padrão:** `admin` / `password`

### Objetivo

Testar força bruta no formulário de login do DVWA usando o Medusa, e avaliar se a ferramenta consegue distinguir corretamente sucesso de falha.

### Configuração necessária no DVWA

Antes de testar, é preciso definir o nível de segurança como **Low**:

1. Acesse o DVWA no navegador do Kali
2. Faça login com `admin` / `password`
3. Vá em **DVWA Security → Security Level → Low → Submit**

> Isso desativa proteções como CAPTCHA e bloqueio de tentativas.

### Problema encontrado: Falsos Positivos

O Medusa opera identificando uma **"string de falha"** — um texto que aparece na resposta HTTP quando a autenticação falha. Quando esse texto **não aparece**, ele considera a tentativa como sucesso.

O problema com o DVWA é que a aplicação pode retornar respostas HTTP similares para login correto e incorreto, dificultando que o Medusa identifique o resultado com precisão.

**O que foi observado:** o Medusa indicou sucesso com a senha `123456` para múltiplos usuários — o que não condiz com o comportamento real. Ao validar manualmente, esses logins falharam. Isso caracteriza **falso positivo**.

```
# Exemplo do falso positivo reportado:
ACCOUNT FOUND: [web-form] Host: 192.168.56.101 User: admin Password: 123456 [SUCCESS]
# ❌ Falso positivo — esse login não funciona na prática
```

![Medusa DVWA - Falso positivo](images/dvwa/medusa_false_positive.png)

![Validação manual DVWA](images/dvwa/manual_validation.png)

### Por que o Medusa falha aqui?

O Medusa é projetado para protocolos com respostas binárias claras (FTP, SSH, SMB). Para formulários web, ele depende de uma string de falha muito específica — e o comportamento do DVWA não é suficientemente consistente para que essa detecção funcione com precisão.

**Ferramentas mais adequadas para esse cenário:**
- `Hydra` com módulo `http-post-form` e configuração de cookie de sessão
- `Burp Suite Intruder` (Community ou Pro)

### Validação manual realizada

Mesmo sem o brute force funcionar, foi possível confirmar o comportamento do sistema:

| Credencial testada | Resultado |
|---|---|
| `admin` / `password` | ✅ Login realizado com sucesso |
| `admin` / `123456` | ❌ Login falhou |
| `admin` / `msfadmin` | ❌ Login falhou |

> ⚠️ **Importante:** A senha correta (`admin` / `password`) **não foi descoberta por força bruta**. É a credencial padrão conhecida do DVWA, usada aqui apenas para validar o comportamento da aplicação.

### Aprendizado desta etapa

Esse foi o cenário mais valioso do projeto — não pelo ataque em si, mas pelo que ele ensinou. **Ferramentas têm escopo ideal e falham fora dele.** Um analista de segurança precisa saber identificar quando um resultado é falso, e documentar isso com clareza. Em um relatório de pentest real, reportar um falso positivo como sucesso seria um erro grave.

---

## 🖥️ Cenário 3 — Password Spraying em SMB ✅ Sucesso

### O que é SMB e o que é Password Spraying?

**SMB (Server Message Block)** é o protocolo usado para compartilhamento de arquivos e impressoras em redes Windows e Linux (via Samba).

**Password Spraying** é uma variação da força bruta tradicional. Em vez de testar muitas senhas para um usuário (o que dispara bloqueios de conta), testamos **uma ou poucas senhas contra muitos usuários**. É uma técnica mais furtiva, comum em ataques a ambientes corporativos.

### Passo 1 — Enumeração de usuários com Nmap

Antes de atacar, identificamos os usuários válidos no sistema:

```bash
nmap -p 445 --script smb-enum-users 192.168.56.101
```

O resultado lista os usuários existentes no Samba, que foram usados para montar a `users.txt`.

![Enumeração SMB com Nmap](images/smb/smb_enum_users.png)

### Passo 2 — Ataque com Medusa

```bash
medusa -h 192.168.56.101 -U users.txt -P passwords.txt -M smbnt
```

> O módulo `smbnt` é específico para autenticação SMB/NTLM — o protocolo de autenticação usado pelo Samba.

### Resultado

```
ACCOUNT FOUND: [smbnt] Host: 192.168.56.101 User: msfadmin Password: msfadmin [SUCCESS]
```

✅ **Credencial encontrada:** `msfadmin` / `msfadmin`

### Passo 3 — Validação do acesso

```bash
smbclient //192.168.56.101/tmp -U msfadmin
# Password: msfadmin
# → smb: \>

# Listar o conteúdo do compartilhamento:
smb: \> ls
```

✅ Acesso confirmado ao sistema de arquivos remoto via SMB.

![Medusa SMB - Credencial encontrada](images/smb/medusa_smb_result.png)

![Acesso SMB validado via smbclient](images/smb/smbclient_access.png)

---

## 📄 Wordlists Utilizadas

As wordlists utilizadas neste projeto são simples e didáticas. No mundo real, existem listas muito maiores — a **rockyou.txt**, por exemplo, já está disponível no Kali em `/usr/share/wordlists/` e contém mais de 14 milhões de senhas reais vazadas.

### `users.txt`
```
root
admin
msfadmin
user
ftp
test
service
```

### `passwords.txt`
```
123456
password
admin
msfadmin
root
toor
admin123
```

> 💡 **Por que senhas tão simples funcionam?**  
> Porque credenciais padrão e senhas fracas continuam sendo um dos vetores de ataque mais explorados. Relatórios como o Verizon DBIR mostram que senhas padrão figuram entre as principais causas de incidentes de segurança ano após ano.

---

## 📊 Resultados Consolidados

| Cenário | Serviço | Ferramenta | Resultado | Credencial |
|---|---|---|---|---|
| 1 | FTP (porta 21) | Medusa | ✅ Sucesso | `msfadmin:msfadmin` |
| 2 | Web Form — DVWA (porta 80) | Medusa | ⚠️ Falso positivo | *(validação manual: `admin:password`)* |
| 3 | SMB (porta 445) | Medusa | ✅ Sucesso | `msfadmin:msfadmin` |

---

## 🧠 Aprendizados Reais

**1. Credenciais padrão são o vetor mais simples — e ainda muito comum**  
Nos três cenários testados, a senha era `msfadmin:msfadmin` ou `admin:password`. Isso continua sendo uma das vulnerabilidades mais exploradas em sistemas reais.

**2. Ferramentas têm escopo ideal — e limitações fora dele**  
O Medusa funciona muito bem em protocolos com autenticação binária clara (FTP, SSH, SMB). Em formulários web com respostas HTTP ambíguas, ele falha. Documentar isso foi tão valioso quanto os ataques bem-sucedidos.

**3. Reconhecimento define a qualidade do ataque**  
O Nmap foi essencial para saber onde atacar. Sem essa etapa, gastaríamos tempo tentando serviços que nem estão ativos.

**4. Validação manual é insubstituível**  
Quando uma ferramenta retorna resultados suspeitos, é o profissional que precisa validar. A ferramenta é um auxiliar — o raciocínio crítico é do analista.

**5. Documentar honestamente é uma habilidade profissional**  
Em Segurança da Informação, relatórios técnicos precisam ser precisos. Reportar um falso positivo como sucesso em um pentest real levaria a decisões erradas. Honestidade técnica não é fraqueza — é profissionalismo.

---

## 🛡️ Medidas de Mitigação

### Geral — Contra Força Bruta

| Medida | Descrição |
|---|---|
| **Política de senha forte** | Mínimo 12 caracteres, com letras maiúsculas, minúsculas, números e símbolos |
| **MFA / 2FA** | Mesmo descobrindo a senha, o atacante não acessa sem o segundo fator |
| **Account Lockout** | Bloquear conta após 5–10 tentativas falhas consecutivas |
| **Rate Limiting** | Limitar tentativas por IP por unidade de tempo |
| **Monitoramento e alertas** | Detectar padrões anômalos de autenticação em tempo real |

### FTP

- Substituir FTP por **SFTP** (usa criptografia via SSH)
- Bloquear login de `root` via FTP
- Implementar whitelist de IPs autorizados

### Aplicações Web

- **CAPTCHA** em formulários de login
- Bloqueio temporário por IP após tentativas excessivas
- Logs de autenticação com alertas automáticos
- **Web Application Firewall (WAF)**
- Nunca usar credenciais padrão — alterar imediatamente após instalação

### SMB

- **Desativar SMBv1** (versão antiga e crítica)
- Bloquear portas 139 e 445 no firewall para acesso externo
- **Segmentação de rede**: SMB deve existir apenas em redes internas confiáveis
- Monitorar tentativas de autenticação distribuídas entre contas (padrão de spraying)

---

## 📁 Estrutura do Repositório

```
brute-force-lab/
│
├── README.md                        # Documentação principal
├── users.txt                        # Lista de usuários testados
├── passwords.txt                    # Lista de senhas testadas
│
└── images/
    ├── recon/                       # Prints do escaneamento com Nmap
    ├── ftp/                         # Prints do ataque FTP e validação
    ├── dvwa/                        # Prints dos falsos positivos e validação manual
    └── smb/                         # Prints da enumeração SMB e acesso validado
```

---

## 📚 Referências

- [Kali Linux — Site Oficial](https://www.kali.org)
- [Metasploitable 2 — SourceForge](https://sourceforge.net/projects/metasploitable/)
- [DVWA — Damn Vulnerable Web Application](https://dvwa.co.uk)
- [Medusa — Documentação Oficial](http://foofus.net/goons/jmk/medusa/medusa.html)
- [Nmap — Manual Oficial](https://nmap.org/book/man.html)
- [OWASP — Brute Force Attack](https://owasp.org/www-community/attacks/Brute_force_attack)
- [Verizon DBIR — Data Breach Investigations Report](https://www.verizon.com/business/resources/reports/dbir/)
- [DIO — Formação Cibersegurança](https://dio.me)
- [Lei nº 12.737/2012 — Crimes Informáticos (BR)](https://www.planalto.gov.br/ccivil_03/_ato2011-2014/2012/lei/l12737.htm)

---

## 👤 Autor

**kskios**  
Estudante de Cibersegurança — Formação DIO  
[![LinkedIn](https://img.shields.io/badge/LinkedIn-blue?style=flat&logo=linkedin)](https://linkedin.com/in/giovannibf)
[![GitHub](https://img.shields.io/badge/GitHub-black?style=flat&logo=github)](https://github.com/kskios)

---

> *"Entender como os ataques funcionam — inclusive quando falham — é o que separa um bom profissional de segurança de alguém que apenas executa comandos."*
