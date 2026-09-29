# Configuração detalhada — Rede, pfSense e Security Onion

Este README é o complemento do [`README.md`](./README.md) principal. Ali está o "o quê" e o "por quê" de cada peça; aqui está o **clique a clique** de três partes que costumam travar a turma: montar a rede no VirtualBox, configurar o pfSense e configurar o Security Onion. Kali e Metasploitable não estão aqui porque a turma já domina esses dois.

Sempre que aparecer **Settings** de uma VM: selecione a VM na lista à esquerda do VirtualBox e clique no ícone de engrenagem (ou botão direito → Settings).

---

## 0. O plano da rede, para não se perder

| VM | Adapter 1 | Adapter 2 |
|---|---|---|
| Kali | Internal Network `lab` | — |
| Metasploitable | Internal Network `lab` | — |
| pfSense | NAT (WAN) | Internal Network `lab` (LAN) |
| Security Onion | NAT (gestão) | Internal Network `lab`, sem IP, Promiscuous Mode **Allow All** |

O pfSense é o único que fala com o mundo (via NAT) e com a rede interna (via `lab`) ao mesmo tempo — por isso ele consegue distribuir IP por DHCP para Kali e Metasploitable. O Security Onion fica "escutando" tudo que passa pela `lab` através da segunda placa, sem participar da rede como um host normal.

---

## 1. Criando a rede "lab" no VirtualBox

**As 4 VMs não são todas iguais aqui.** Kali e Metasploitable têm uma placa só. pfSense e Security Onion têm duas, e nessas duas a **Adapter 1 é NAT, não `lab`** — é um erro comum copiar o mesmo padrão nas quatro máquinas, então preste atenção na tabela da seção 0 antes de clicar em qualquer coisa.

### 1.1 — Kali e Metasploitable (uma placa só)

1. Selecione a VM → **Settings** → aba **Network** → aba **Adapter 1**.
2. Marque **Enable Network Adapter**.
3. Em **Attached to**, escolha **Internal Network**.
4. No campo **Name**, digite `lab` (sem espaços, minúsculo — o nome tem que ser **idêntico** em todas as VMs, é isso que as coloca na mesma rede).
5. Clique **OK**.

### 1.2 — pfSense e Security Onion (duas placas, mesmo padrão nas duas)

1. Selecione a VM → **Settings** → aba **Network** → aba **Adapter 1**.
2. Marque **Enable Network Adapter**. Em **Attached to**, deixe **NAT** (é o padrão do VirtualBox, normalmente não precisa mexer em nada aqui).
3. Clique na aba **Adapter 2**.
4. Marque **Enable Network Adapter**. Em **Attached to**, escolha **Internal Network**, nome `lab`.
5. Clique **OK**.

Ou seja: nas duas, **Adapter 1 = NAT** e **Adapter 2 = Internal Network "lab"**. É o mesmo padrão nas duas VMs — a única diferença entre elas é o passo extra do Promiscuous Mode, que só o Security Onion precisa (próxima seção).

> ⚠️ Erro mais comum aqui: (1) digitar o nome da rede diferente em cada VM (`lab`, `Lab`, `lab ` com espaço no final) — se duas VMs não se enxergam, confira isso primeiro; (2) copiar "Internal Network lab" para o Adapter 1 do pfSense ou do Security Onion por engano — nessas duas, o Adapter 1 é NAT.

### 1.3 — Só no Security Onion: ligando o Promiscuous Mode

1. Ainda em **Settings → Network**, clique na aba **Adapter 2** (a que está em `lab`).
2. Clique em **Advanced** (uma seta ou link que expande mais opções, dependendo da versão do VirtualBox).
3. No campo **Promiscuous Mode**, troque de "Deny" para **"Allow All"**.
4. Clique **OK**.

Sem isso, o Security Onion só vê pacotes endereçados a ele mesmo — ou seja, nenhum. É o passo que mais gente esquece, e é o motivo número um de "Alerts" ficar vazio depois.

---

## 2. Configurando o pfSense

### 2.1 — Console (a telinha preta que aparece ao ligar a VM)

Depois de instalar o pfSense (instalador com opções padrão), ele reinicia e mostra um menu de texto. É ali que ele pergunta quem é WAN e quem é LAN.

1. Digite **1** (Assign Interfaces) e Enter.
2. Pergunta sobre VLANs → digite **n** e Enter (não usamos VLAN).
3. **Enter the WAN interface name**: digite o nome da primeira placa (geralmente `em0` ou `vtnet0` — o próprio menu lista os nomes disponíveis).
4. **Enter the LAN interface name**: digite o nome da segunda placa.
5. Se perguntar por mais interfaces opcionais, só dê Enter para pular.
6. Confirme as atribuições digitando **y** e Enter.

Depois disso, o próprio menu mostra as opções numeradas de novo. Confira o **IP da LAN** (normalmente já vem `192.168.1.1`):

7. Digite **2** (Set interface(s) IP address) e Enter.
8. Escolha a interface **LAN** (o menu lista o número correspondente).
9. Pode manter o IP sugerido ou digitar o seu, por exemplo `10.10.10.1`.
10. Quando perguntar a máscara (subnet bit count), digite **24**.
11. Pule (Enter) as perguntas de gateway e IPv6.
12. Quando perguntar se quer habilitar o servidor DHCP na LAN, digite **y** — é isso que vai dar IP automático para o Kali e o Metasploitable.
13. Defina o intervalo do DHCP, por exemplo de `10.10.10.100` até `10.10.10.200`.

Anote o IP da LAN que você configurou (ex.: `10.10.10.1`) — é o endereço que você vai digitar no navegador no próximo passo.

### 2.2 — Assistente de configuração (setup wizard) no navegador

Como o pfSense não está numa rede física com internet direto pra sala, o jeito mais simples de acessar a interface web é **de dentro do Kali** (que está na mesma rede `lab`), não do computador físico.

1. Ligue o Kali, abra o navegador e acesse `https://<IP da LAN do pfSense>` (ex.: `https://10.10.10.1`).
2. O navegador vai reclamar de certificado — clique em **Avançado** → **Continuar mesmo assim** (é um certificado autoassinado, esperado).
3. Login padrão: usuário `admin`, senha `pfsense`.
4. **Tela 1 — Welcome**: clique **Next**.
5. **Tela 2 — Support**: clique **Next**.
6. **Tela 3 — General Information**:
   - Hostname: `pfsense` (ou o que preferir).
   - Domain: `lab.local`.
   - DNS Servers: pode deixar em branco ou usar `1.1.1.1`.
   - Clique **Next**.
7. **Tela 4 — Time Server**: confira o fuso horário (São Paulo) e clique **Next**.
8. **Tela 5 — WAN Configuration**:
   - Deixe em **DHCP** (é assim que ele pega IP do NAT do VirtualBox).
   - **Desmarque** "Block RFC1918 Private Networks" e "Block bogon networks" — se deixar marcado, o pfSense bloqueia a própria rede interna por engano.
   - Clique **Next**.
9. **Tela 6 — LAN Configuration**: confira se o IP é o mesmo que você configurou no console. Não mexa. Clique **Next**.
10. **Tela 7 — Set Admin Password**: defina uma senha nova (fica mais fácil de lembrar: `SenacLab2026!`, por exemplo). Clique **Next**.
11. **Tela 8 — Reloading**: espera de 30 a 60 segundos.
12. **Tela 9 — Complete**: clique **Finish**. Você cai no Dashboard.

### 2.3 — Conferindo se está tudo funcionando

- Reinicie a VM do Kali e a do Metasploitable.
- No Kali, rode `ip a` (ou `ifconfig`) e confirme que ele recebeu um IP dentro da faixa configurada (ex.: `10.10.10.100`).
- No Metasploitable, rode `ifconfig` e confirme o mesmo.
- Se nenhum dos dois recebeu IP: volte no passo 2.1.12 e confirme que o DHCP da LAN está ligado.

---

## 3. Configurando o Security Onion

O instalador do Security Onion é todo por texto, tela após tela, dentro da própria VM (não é web ainda). Fique de olho na ordem — ele não deixa muito claro quando uma pergunta termina e a próxima começa.

> ⚠️ **Aviso de confiança**: a documentação oficial do Security Onion não publica o texto exato de cada tela do instalador (só descreve o fluxo geral), então a lista abaixo foi reconstruída cruzando relatos de quem já instalou a versão 2.4. O **conteúdo** de cada pergunta (rede, hostname, e-mail, interface de monitoramento etc.) está bem estabelecido — mas a **ordem exata de 2 ou 3 telas específicas** (por exemplo, se "Patch schedule" vem antes ou depois de "Docker IP range") pode variar um pouco conforme a build exata do ISO. Siga o **texto que aparecer na tela**, não o número do passo, se algum deles vier em ordem diferente — o importante é responder cada pergunta com o valor certo, não a ordem em si.

### 3.1 — Instalação do sistema operacional

1. Dê boot no ISO, escolha **Install**, e siga o instalador de disco com as opções padrão (idioma, teclado, disco inteiro).
2. Ele pede um **usuário e senha** — são as credenciais de login do sistema operacional (não confundir com o login do site depois). Ex.: usuário `analista`, senha à sua escolha.
3. Ao terminar, a VM reinicia e o **Security Onion Setup** inicia sozinho.

### 3.2 — Setup do Security Onion (tela por tela)

1. **Install type**: escolha **EVAL** (Evaluation Mode / Standalone) — é o modo de uma VM só, feito exatamente para laboratório. Digite `AGREE` quando pedir confirmação e selecione **OK**.
2. **Configure Network**: pergunta se você quer configurar a rede agora — escolha **Yes**.
3. **Management NIC**: selecione a placa que está em **NAT** como interface de gestão. Use a barra de espaço para marcar e Tab/Enter para confirmar.

   > 💡 Se aparecerem duas interfaces com nomes tipo `eth0`/`eth1` ou `enp0s3`/`enp0s8` e não estiver óbvio qual é qual: o instalador mostra o **endereço MAC** ao lado de cada uma. Compare com **Settings → Network → Adapter 1 → Advanced → MAC Address** (e o mesmo para o Adapter 2) no VirtualBox para ter certeza de qual placa é qual antes de confirmar — a ordem nem sempre bate 1-para-1 com "Adapter 1 = primeira da lista".
4. **Configure IP address**: escolha **DHCP** (mais simples — o NAT do VirtualBox já entrega IP sozinho) ou **STATIC** se quiser fixar um endereço. Para a sala, DHCP é suficiente.
5. Se escolheu STATIC, ele vai pedir: IP (ex.: `10.0.2.20/24`), gateway e DNS — pode manter os valores padrão sugeridos.
6. **Hostname**: digite algo como `securityonion`.
7. **DNS search domain**: pode deixar em branco ou usar `lab.local`.
8. **Docker IP range**: confirme com **Yes** (é a rede interna que o Security Onion usa entre seus próprios containers — não mexe na nossa rede `lab`).
9. **Internet connection**: escolha **DIRECT** (conexão direta, sem proxy).
10. **Patch schedule**: pode deixar a opção padrão (atualização automática) ou "Manage manually", tanto faz para o laboratório.
11. **Email e senha do painel web**: aqui é o login do **site** (diferente do login do sistema operacional do passo 3.1.2). Anote os dois — vai precisar depois.
12. **Access method**: escolha **IP** (acessar pelo endereço IP, mais simples que configurar um nome de domínio).
13. **Monitor interface**: agora ele pede para escolher a interface de **monitoramento** — selecione a placa que está na rede `lab` (sem IP, com Promiscuous Mode ligado) — de novo, confira pelo MAC Address se tiver dúvida.
14. **Monitoring network range**: informe a faixa de rede que o pfSense está distribuindo na `lab` (ex.: `10.10.10.0/24`). É esse endereço que o Security Onion vai considerar "tráfego interno".
15. **Confirmação final**: ele mostra um resumo de tudo o que você configurou. Use Tab para chegar em **Yes** e confirme.

A instalação demora — pode levar entre 20 minutos e mais de uma hora dependendo da máquina. Ela mostra uma barra de progresso; não desligue a VM no meio.

### 3.3 — Acessando o painel web (Alerts)

O painel do Security Onion é acessado pelo IP da interface de **gestão** (a NAT), mas como essa rede é privada do VirtualBox, o computador físico não enxerga esse IP direto — é preciso abrir uma porta:

1. Com a VM do Security Onion **desligada** (ou com a VM selecionada), vá em **Settings → Network → Adapter 1 (NAT) → Advanced → Port Forwarding**.
2. Clique no **+** para adicionar uma regra:
   - **Name**: `SO-Web`
   - **Protocol**: TCP
   - **Host IP**: (deixe em branco)
   - **Host Port**: `8443`
   - **Guest IP**: (deixe em branco, ou `10.0.2.15` se pedir)
   - **Guest Port**: `443`
3. Clique **OK** em tudo.
4. No navegador do computador da sala (o host), acesse `https://localhost:8443`.
5. Aceite o aviso de certificado autoassinado.
6. Faça login com o e-mail e senha definidos no passo 3.2.11.
7. Clique em **Alerts** no menu.

---

## 4. Checklist de rede — antes de rodar o teste

- [ ] Todas as 4 VMs com o nome de rede interna **exatamente** `lab` nas placas corretas
- [ ] pfSense: Adapter 1 = NAT, Adapter 2 = Internal "lab", DHCP ligado na LAN
- [ ] Security Onion: Adapter 1 = NAT, Adapter 2 = Internal "lab" **com Promiscuous Mode "Allow All"**
- [ ] Kali e Metasploitable reiniciados e com IP recebido via DHCP do pfSense (`ip a` / `ifconfig`)
- [ ] Consigo acessar `https://<IP da LAN do pfSense>` pelo navegador do Kali
- [ ] Consigo acessar `https://localhost:8443` pelo navegador do computador da sala e ver a tela de login do Security Onion

Se todos os itens acima estão marcados, o teste do `nmap -sV` seguido de checar **Alerts** deve funcionar. Se não funcionar mesmo assim, o problema é quase sempre o **Promiscuous Mode** do Security Onion — volte na seção 1, passos 8–11.

---

## 5. Problemas comuns de rede (específicos desta parte)

| Sintoma | Causa provável | Solução |
|---|---|---|
| Kali/Metasploitable não recebem IP | Nome da rede `lab` diferente entre VMs, ou DHCP desligado no pfSense | Revisar nome exato em todas as VMs; revisar passo 2.1.12 |
| Não consigo abrir a interface web do pfSense | Tentando acessar pelo computador físico em vez do Kali | Acesse pelo navegador **dentro do Kali**, que está na mesma rede `lab` |
| pfSense bloqueia tudo, "site não responde" | "Block RFC1918" ou "Block bogon networks" marcados na WAN | Voltar em Interfaces → WAN e desmarcar as duas opções |
| Security Onion "Alerts" sempre vazio | Promiscuous Mode não está "Allow All" na placa do monitor | Settings → Network → Adapter 2 → Advanced → Allow All, e reiniciar a VM |
| Não consigo abrir `https://localhost:8443` | Regra de Port Forwarding não criada, ou porta errada | Revisar seção 3.3; a VM do Security Onion precisa estar ligada |
| Security Onion não gera nenhum alerta mesmo com promiscuous certo | Faixa de rede informada no passo 3.2.14 está errada | Reabrir `sudo so-network` (linha de comando) ou reinstalar o setup e conferir a faixa (`10.10.10.0/24`, a mesma do pfSense) |

---

## Fontes usadas para as instruções desta parte

- [Console Menu Basics — pfSense Documentation](https://docs.netgate.com/pfsense/en/latest/config/console-menu.html)
- [Interface Configuration — pfSense Documentation](https://docs.netgate.com/pfsense/en/latest/config/interface-configuration.html)
- [The Ultimate pfSense Configuration Guide for Beginners — The CyberSec Guru](https://thecybersecguru.com/self-hosting/pfsense-configuration-guide-initial-setup/)
- [Getting Started — Security Onion 2.4 Documentation](https://docs.securityonion.net/en/2.4/getting-started.html)
- [Configuration — Security Onion 2.4 Documentation](https://docs.securityonion.net/en/2.4/configuration.html)
- [Security Onion: Introduction and Installation Steps — Brijesh Chauhan](https://medium.com/@cb346666/security-onion-introduction-and-installation-steps-54f8acc323dc)
- [Security Onion (Part 4) — How to monitor the networks — Danny Vargas](https://medium.com/@itdanny/security-onion-part-4-how-to-monitor-the-networks-96452a140149)
- [VirtualBox Network Settings: All You Need to Know — Nakivo](https://www.nakivo.com/blog/virtualbox-network-setting-guide/)
