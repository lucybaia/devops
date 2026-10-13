---
title: Ferramentas de Diagnóstico
tags:
  - devops
  - networking
  - troubleshooting
  - linux
aliases:
  - Ferramentas de Rede
  - Comandos de Rede Linux
  - tcpdump
  - dig
  - curl
---

# Ferramentas de Diagnóstico de Rede

> [!info] Posição na trilha
> **Etapa 4 — Diagnóstico e Segurança.** Pré-requisitos: Etapas 1–3. Complementa [[18 - Diagnóstico e Troubleshooting]] (metodologia) e [[24 - Laboratórios Práticos]] (prática).

## 1. Definição e visão geral

Esta nota é uma **referência prática** das ferramentas de linha de comando mais usadas para diagnosticar redes em Linux. Para cada uma: finalidade, sintaxe, opções importantes, exemplos e como interpretar a saída.

| Ferramenta | Camada | Pergunta que responde | Pacote comum (Debian/Ubuntu) |
|---|---|---|---|
| `ip addr` / `ip link` | 1–3 | Quais interfaces e IPs tenho? | `iproute2` |
| `ip route` | 3 | Por onde sai o tráfego? | `iproute2` |
| `ip neigh` | 2–3 | Conheço o MAC dos vizinhos? | `iproute2` |
| `ping` | 3 | O destino responde? Qual o RTT e a perda? | `iputils-ping` |
| `traceroute` / `tracepath` | 3 | Por quais saltos o tráfego passa? | `traceroute` / `iputils-tracepath` |
| `mtr` | 3 | Onde há perda/latência no caminho, ao longo do tempo? | `mtr-tiny` ou `mtr` |
| `ss` | 4 | Quem escuta? Quais conexões existem? | `iproute2` |
| `nc` | 4 | A porta aceita conexão? | `netcat-openbsd` |
| `dig` / `nslookup` | 7 (DNS) | O que o DNS responde? | `dnsutils` (`bind9-dnsutils`) |
| `getent` | 7 (NSS) | O que o sistema/aplicação resolve? | `libc-bin` (já instalado) |
| `curl` | 7 | O serviço HTTP(S) responde corretamente? | `curl` |
| `openssl s_client` | TLS | O handshake TLS e o certificado estão corretos? | `openssl` |
| `tcpdump` | 2–7 | O que realmente passa pela interface? | `tcpdump` |

Em RHEL/Fedora os nomes de pacote diferem (ex.: `bind-utils` para `dig`, `iputils`, `nmap-ncat` para `nc`).

> [!tip] Imagem "canivete suíço"
> A imagem `nicolaka/netshoot` contém praticamente todas essas ferramentas e é muito usada para depurar redes de containers e Pods.

## 2. Por que isso importa

Ferramentas certas transformam suposições em **evidências**. Saber ler a saída de cada uma permite localizar a camada e o ponto exato da falha.

---

## 3. `ip` (iproute2)

Substitui as ferramentas legadas `ifconfig`, `route` e `arp` (pacote `net-tools`, que não vem instalado em muitas distribuições e imagens atuais).

### `ip addr` / `ip link`

```bash
ip addr show                 # ip a
ip -br addr                  # resumo (brief)
ip -br link                  # estado das interfaces
ip -4 addr show dev eth0     # IPv4 de uma interface
ip -s link show eth0         # estatísticas: RX/TX, erros, drops
ip -d link show              # detalhes (tipo: veth, bridge, vlan, vxlan...)
```

Interpretação de `ip -s link`:

```text
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ...
    RX:  bytes  packets errors dropped  missed   mcast
    9876543210  8123456      0     12       0       0
    TX:  bytes  packets errors dropped carrier collsns
    1234567890  5123456      0      0       0       0
```

- `errors` crescendo → problema físico/driver.
- `dropped` crescendo → filas/buffers cheios ou pacotes descartados pelo kernel.
- `UP` sem `LOWER_UP` → sem link físico.

### `ip route`

```bash
ip route                         # tabela principal
ip route get 1.1.1.1             # rota efetiva para um destino
ip -6 route
ip route show table all          # todas as tabelas (policy routing)
ip rule                          # regras de seleção de tabela
```

Ver interpretação em [[09 - Roteamento e Gateway]].

### `ip neigh`

```bash
ip neigh                         # tabela ARP/NDP
ip neigh show dev eth0
```

Estados `REACHABLE`, `STALE`, `FAILED` — ver [[08 - DHCP e ARP]].

> [!warning] Mudanças com `ip` não são persistentes e podem derrubar acesso
> `ip addr del`, `ip link set ... down` e `ip route del default` têm efeito imediato. Em servidores remotos, isso pode cortar sua sessão SSH. A configuração permanente fica em netplan, NetworkManager, systemd-networkd ou `/etc/network/interfaces`, conforme a distribuição.

---

## 4. `ping`

**Finalidade:** testar alcance L3 e medir RTT, perda e jitter usando ICMP Echo.

```bash
ping -c 4 8.8.8.8                # -c: número de pacotes (no Linux, sem -c pinga indefinidamente)
ping -c 4 -i 0.2 host            # -i: intervalo em segundos (intervalos muito curtos exigem root; o limite varia com a versão do iputils)
ping -c 3 -W 2 host              # -W: tempo de espera por resposta (s)
ping -c 3 -s 1400 host           # -s: tamanho do payload
ping -M do -s 1472 -c 3 host     # -M do: proíbe fragmentação (teste de MTU)
ping -c 3 -I eth1 host           # -I: interface/endereço de origem
ping -6 -c 3 ipv6.google.com     # força IPv6 (ou: ping6)
```

Windows: `ping -n 4 host`, `-l` tamanho, `-f` não fragmentar.

**Interpretação:**

| Saída | Significado |
|---|---|
| `64 bytes from ...: icmp_seq=1 ttl=57 time=12.4 ms` | Resposta; RTT de 12,4 ms |
| `ttl=57` | TTL restante na resposta. Sistemas usam TTL inicial típico de 64 (Linux), 128 (Windows) ou 255 (equipamentos de rede); 57 sugere ~7 saltos a partir de 64. É apenas uma pista |
| `X% packet loss` | Perda |
| `rtt min/avg/max/mdev` | Estatísticas; `mdev` ≈ jitter |
| `Request timeout` / nenhuma linha | Sem resposta (filtro, host fora, rota de volta) |
| `Destination Host Unreachable` | ARP falhou (se vier do próprio IP) ou roteador remoto não entrega |
| `DUP!` | Respostas duplicadas: possível IP duplicado ou problema de L2 |

> [!note] Ausência de resposta ao ping não prova que o host está fora
> Clouds e firewalls frequentemente bloqueiam ICMP. Confirme com um teste de porta TCP (`nc -vz`).

---

## 5. `traceroute` e `tracepath`

**Finalidade:** revelar os roteadores do caminho, salto a salto, usando TTL crescente. Ver o funcionamento em [[09 - Roteamento e Gateway]].

```bash
traceroute -n 8.8.8.8                 # -n: sem DNS reverso
traceroute -n -I 8.8.8.8              # -I: sondas ICMP Echo (requer root)
sudo traceroute -n -T -p 443 exemplo.com   # -T: TCP SYN na porta 443 (atravessa firewalls que liberam só 443)
traceroute -n -q 1 -w 2 host          # -q: sondas por salto | -w: espera (s)

tracepath -n 8.8.8.8                  # sem root; mostra a PMTU descoberta
```

Windows: `tracert -d host` (ICMP).

**Interpretação de `tracepath`:**

```text
 1?: [LOCALHOST]                      pmtu 1500
 1:  192.168.1.1                       0.950ms
 2:  10.20.0.1                         7.812ms
 3:  no reply
 4:  187.x.x.x                        11.902ms pmtu 1492
 ...
     Resume: pmtu 1492 hops 9 back 9
```

- `pmtu 1492` → a MTU do caminho caiu para 1492 em algum ponto (típico de PPPoE).
- `no reply` ou `* * *` → salto não respondeu; só é relevante se **todos os saltos seguintes** também não responderem.

---

## 6. `mtr`

**Finalidade:** combina `ping` e `traceroute`, enviando sondas continuamente e mostrando estatísticas por salto. Excelente para perda intermitente.

```bash
mtr -n 8.8.8.8                        # modo interativo
mtr -n -r -c 100 8.8.8.8              # -r: relatório | -c: 100 ciclos
mtr -n -r -c 100 -T -P 443 exemplo.com   # TCP na porta 443 (requer root na maioria das instalações)
mtr -n -r -c 100 -u host              # sondas UDP
```

**Interpretação:**

```text
HOST: meu-host                Loss%   Snt   Last   Avg  Best  Wrst StDev
  1.|-- 192.168.1.1             0.0%   100    0.9   1.0   0.7   3.1   0.3
  2.|-- 10.20.0.1               0.0%   100    7.9   8.2   7.1  15.0   1.1
  3.|-- 187.x.x.1              35.0%   100   12.0  12.3  11.2  20.1   1.2
  4.|-- 187.x.x.2               0.0%   100   12.4  12.6  11.9  19.4   1.0
  5.|-- 8.8.8.8                 0.0%   100   13.1  13.0  12.5  18.2   0.7
```

- Perda de 35% **apenas no salto 3**, com 0% nos seguintes e no destino → o roteador 3 limita respostas ICMP (*rate limiting*). **Não é perda real.**
- Perda que **começa num salto e persiste até o destino** → perda real a partir daquele ponto.
- `StDev` alto → jitter.

---

## 7. `ss`

**Finalidade:** listar sockets (escuta e conexões), com estados e detalhes TCP. Substitui `netstat`.

| Opção | Significado |
|---|---|
| `-t` / `-u` / `-x` | TCP / UDP / sockets Unix |
| `-l` | Somente em escuta |
| `-a` | Todos (escuta + não escuta) |
| `-n` | Numérico (sem resolver nomes/serviços) |
| `-p` | Processo dono do socket (root para ver de outros usuários) |
| `-i` | Informações internas do TCP (rtt, cwnd, retransmissões) |
| `-s` | Resumo |
| `-o` | Timers (keepalive, retransmissão) |
| `-e` | Informações estendidas (uid, inode) |

```bash
sudo ss -tlnp                                  # quem escuta em TCP
sudo ss -ulnp                                  # quem escuta em UDP
ss -tn state established                       # conexões estabelecidas
ss -tan state time-wait | wc -l                # quantas em TIME-WAIT
ss -tn dst 10.0.3.15                           # conexões para um IP
ss -tn '( dport = :5432 )'                     # conexões para a porta 5432 remota
ss -ti dst 1.1.1.1                             # métricas TCP internas
ss -s                                          # resumo
```

Interpretação detalhada em [[05 - TCP e UDP]] e [[06 - Portas e Sockets]].

---

## 8. `nc` (netcat)

**Finalidade:** abrir conexões TCP/UDP arbitrárias — teste de porta, servidores e clientes simples.

> [!note] Existem várias implementações
> **OpenBSD netcat** (padrão no Debian/Ubuntu), **netcat tradicional**, **Ncat** (do Nmap, padrão no RHEL como `nc`) e o `nc` do BusyBox (Alpine) têm opções diferentes. Os exemplos abaixo usam o OpenBSD netcat. Confira com `nc -h`.

```bash
nc -vz -w 3 10.0.3.15 5432         # testar porta TCP: -v verbose, -z sem enviar dados, -w timeout
nc -vz 10.0.3.15 8000-8010         # varrer uma faixa (apenas em hosts próprios/autorizados)
nc -vzu 10.0.0.2 53                # UDP: "sucesso" é pouco confiável (ver abaixo)
nc -l 9000                         # servidor TCP na porta 9000 (tradicional: nc -l -p 9000)
nc 10.0.0.5 9000                   # cliente; digite texto para enviar
printf 'GET / HTTP/1.0\r\nHost: exemplo.com\r\n\r\n' | nc exemplo.com 80
```

**Interpretação:** ver tabela em [[06 - Portas e Sockets]] (`succeeded` / `refused` / `timed out`).

> [!warning] Testes UDP com `nc` enganam
> Como o UDP não tem handshake, `nc -zu` frequentemente reporta "sucesso" mesmo sem nenhum serviço (ausência de ICMP Port Unreachable é interpretada como porta aberta). Para validar serviços UDP, use um cliente do próprio protocolo (ex.: `dig @servidor` para DNS).

Alternativa sem `nc` (bash): `timeout 3 bash -c '</dev/tcp/10.0.3.15/5432' && echo aberta`.

---

## 9. `dig` e `nslookup`

**Finalidade:** consultar servidores DNS diretamente.

```bash
dig exemplo.com                     # registro A, resolver do sistema
dig +short exemplo.com AAAA
dig @1.1.1.1 exemplo.com MX         # resolver específico e tipo
dig +trace exemplo.com              # delegação completa a partir da raiz
dig +norecurse @ns1.exemplo.com exemplo.com   # pergunta ao autoritativo sem recursão
dig -x 203.0.113.10                 # reverso (PTR)
dig +noall +answer exemplo.com      # só a seção de resposta
dig +tcp exemplo.com                # força TCP
dig exemplo.com +dnssec             # inclui registros DNSSEC

nslookup exemplo.com
nslookup -type=TXT exemplo.com 8.8.8.8
```

Interpretação detalhada em [[07 - DNS]]. Lembretes: `status:` mostra o RCODE; `flags: aa` indica resposta autoritativa; o número após o nome é o TTL restante; `SERVER:` mostra quem respondeu.

`nslookup` é útil por existir também no Windows, mas `dig` dá saídas mais completas e previsíveis. Ambos **ignoram `/etc/hosts`**.

---

## 10. `getent`

**Finalidade:** consultar as bases do **NSS** (Name Service Switch) exatamente como a glibc faz para as aplicações, respeitando `/etc/nsswitch.conf`, `/etc/hosts`, systemd‑resolved, etc.

```bash
getent hosts exemplo.com            # endereço preferido
getent ahosts exemplo.com           # todos os endereços (IPv4 e IPv6), com tipos de socket
getent ahostsv4 exemplo.com         # apenas IPv4
getent services 443                 # /etc/services
getent passwd root                  # (também funciona para outras bases NSS)
```

**Quando usar:** sempre que `dig` e a aplicação "discordarem". Se `getent` falha e `dig` funciona, o problema está na configuração local (nsswitch, hosts, search domains, resolv.conf) e não no servidor DNS.

> [!note] Alpine/musl
> Imagens baseadas em musl não usam NSS da glibc da mesma forma; o `getent` do BusyBox/musl pode ter comportamento limitado. Em Alpine, teste também com a própria aplicação ou com `nslookup`.

---

## 11. `curl`

**Finalidade:** cliente para HTTP(S) e muitos outros protocolos. Ferramenta principal de teste L7.

| Opção | Função |
|---|---|
| `-v` | Detalhes de conexão, TLS, cabeçalhos |
| `-I` | Requisição HEAD (só cabeçalhos) |
| `-i` | Inclui cabeçalhos da resposta na saída |
| `-L` | Segue redirects |
| `-o arquivo` / `-O` | Salva corpo em arquivo |
| `-s` / `-S` | Silencioso / mostrar erros mesmo em modo silencioso |
| `-X MÉTODO` | Método HTTP |
| `-H 'Nome: valor'` | Cabeçalho |
| `-d dados` / `--json dados` | Corpo (o `--json` existe nas versões 7.82+) |
| `--resolve host:porta:ip` | Força o IP para um nome (mantém Host e SNI) |
| `--connect-to host:porta:host2:porta2` | Redireciona a conexão para outro destino |
| `--max-time N` / `--connect-timeout N` | Limites de tempo |
| `-w formato` | Imprime métricas (tempos, status) |
| `--fail` | Código de saída ≠ 0 para HTTP ≥ 400 (útil em scripts) |
| `-x proxy` / `--noproxy` | Usar/ignorar proxy |
| `--cacert`, `--cert`, `--key` | CA personalizada, certificado de cliente (mTLS) |
| `-4` / `-6` | Forçar IPv4/IPv6 |
| `-k` | **Ignora erros de certificado — apenas para diagnóstico pontual** |

```bash
curl -v https://api.exemplo.com/health
curl -sS -o /dev/null -w '%{http_code}\n' https://api.exemplo.com/health
curl -sS -o /dev/null -w 'dns=%{time_namelookup} conn=%{time_connect} tls=%{time_appconnect} ttfb=%{time_starttransfer} total=%{time_total} ip=%{remote_ip}\n' https://api.exemplo.com/
curl -v --resolve api.exemplo.com:443:10.0.1.20 https://api.exemplo.com/
curl -v -4 https://api.exemplo.com/
curl --cert cliente.crt --key cliente.key --cacert ca.crt https://mtls.interno/
```

Códigos de saída úteis em scripts: `6` (não resolveu host), `7` (falha ao conectar), `28` (timeout), `35` (erro de handshake TLS), `52` (resposta vazia), `56` (falha ao receber dados), `60` (certificado não verificado). Lista completa: `man curl`, seção *EXIT CODES*.

Interpretação de `curl -v` e `-w`: ver [[11 - HTTP e HTTPS]].

---

## 12. `openssl s_client`

**Finalidade:** testar handshakes TLS e inspecionar certificados.

```bash
openssl s_client -connect host:443 -servername host </dev/null
openssl s_client -connect host:443 -servername host -showcerts </dev/null
openssl s_client -connect host:443 -servername host -verify_hostname host </dev/null
openssl s_client -connect host:443 -servername host -alpn h2,http/1.1 </dev/null   # ver ALPN negociado
openssl s_client -connect host:443 -servername host -tls1_2 </dev/null
openssl s_client -connect smtp.exemplo.com:587 -starttls smtp </dev/null           # STARTTLS
openssl s_client -connect db.exemplo.com:5432 -starttls postgres </dev/null        # OpenSSL 1.1.1+

# Datas e SANs do certificado
openssl s_client -connect host:443 -servername host </dev/null 2>/dev/null | openssl x509 -noout -dates -subject -issuer -ext subjectAltName
```

Interpretação e códigos de verificação: ver [[12 - TLS e Certificados]].

---

## 13. `tcpdump`

**Finalidade:** capturar e exibir pacotes reais. É a "verdade de campo" quando outras ferramentas divergem.

| Opção | Função |
|---|---|
| `-i eth0` / `-i any` | Interface (any = todas, no Linux) |
| `-n` | Não resolver nomes (use sempre; evita gerar consultas DNS e atrasos) |
| `-nn` | Também não converter portas em nomes |
| `-e` | Mostrar cabeçalho de enlace (MACs, VLAN) |
| `-v`, `-vv` | Mais detalhes (TTL, opções, decodificação) |
| `-c N` | Parar após N pacotes |
| `-s 0` | Capturar o pacote inteiro (padrão em versões modernas) |
| `-w arquivo.pcap` | Gravar para análise posterior (ex.: Wireshark) |
| `-r arquivo.pcap` | Ler captura |
| `-A` / `-X` | Mostrar payload em ASCII / hex+ASCII |
| `-S` | Números de sequência TCP absolutos |
| `-tttt` | Timestamps com data legíveis |

**Filtros (sintaxe BPF/pcap-filter):**

```bash
sudo tcpdump -n -i any host 10.0.3.15
sudo tcpdump -n -i any port 443
sudo tcpdump -n -i eth0 'src host 10.0.1.10 and dst port 5432'
sudo tcpdump -n -i any 'tcp[tcpflags] & (tcp-syn|tcp-rst) != 0'     # só SYN e RST
sudo tcpdump -n -i any icmp
sudo tcpdump -n -i any 'udp port 53'
sudo tcpdump -n -i eth0 -e arp
sudo tcpdump -n -i any -c 200 -w /tmp/captura.pcap port 8080
sudo tcpdump -n -A -i any 'tcp port 80 and (tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x47455420)'   # requisições HTTP GET (texto claro)
```

**Interpretação de linhas comuns:**

```text
14:02:11.123456 IP 10.0.1.10.51544 > 10.0.3.15.5432: Flags [S], seq 3829104, win 64240, options [mss 1460,...], length 0
14:02:12.130001 IP 10.0.1.10.51544 > 10.0.3.15.5432: Flags [S], seq 3829104, win 64240, ..., length 0
14:02:14.146002 IP 10.0.1.10.51544 > 10.0.3.15.5432: Flags [S], seq 3829104, ...
```

→ O **mesmo SYN** retransmitido (1 s, depois 2 s) e nenhum SYN‑ACK: **o destino não responde** (filtro ou rota). Este é o padrão de "Connection timed out".

```text
IP 10.0.1.10.51544 > 10.0.3.15.5432: Flags [S], ...
IP 10.0.3.15.5432 > 10.0.1.10.51544: Flags [R.], seq 0, ack 3829105, win 0, length 0
```

→ RST imediato: **porta fechada** → "Connection refused".

```text
IP 10.0.1.10.40000 > 10.0.0.2.53: 41235+ A? api.exemplo.com. (33)
IP 10.0.0.2.53 > 10.0.1.10.40000: 41235 NXDomain 0/1/0 (108)
```

→ Consulta DNS (`A?`) e resposta NXDOMAIN.

Flags TCP: ver tabela em [[05 - TCP e UDP]].

> [!warning] Cuidados com `tcpdump`
> - Requer privilégios (root ou capabilities `CAP_NET_RAW`/`CAP_NET_ADMIN`).
> - Capturas podem conter **dados sensíveis**; não as compartilhe sem sanitizar.
> - Em servidores muito movimentados, capture com filtros específicos e `-c`, ou grave em arquivo, para não sobrecarregar CPU e disco.
> - Em hosts com containers, `-i any` mostra o mesmo pacote várias vezes (em `veth`, bridge e interface física), às vezes com endereços diferentes (antes/depois do NAT). Isso é útil para ver o NAT acontecendo.

---

## 14. Outras ferramentas úteis

| Ferramenta | Uso |
|---|---|
| `nmap` | Varredura de portas e serviços (somente em sistemas próprios/autorizados) |
| `iperf3` | Medir throughput entre dois hosts (servidor `iperf3 -s`, cliente `iperf3 -c host`) |
| `ethtool` | Velocidade, duplex, link, offloads e estatísticas da placa |
| `conntrack` | Inspecionar a tabela de conexões do netfilter |
| `nft` / `iptables` | Regras de firewall/NAT |
| `resolvectl` | Estado do systemd-resolved |
| `nstat` / `netstat -s` | Contadores de protocolo (retransmissões, erros) |
| `lsof -i` | Arquivos/sockets abertos por processo |
| `wget` | Alternativa ao curl em imagens mínimas |
| `nsenter` | Entrar no network namespace de um processo/container: `sudo nsenter -t <PID> -n ip addr` |
| `ip netns` | Gerenciar namespaces de rede nomeados |
| Wireshark / `tshark` | Análise gráfica/detalhada de capturas `.pcap` |
| `kubectl debug`, `kubectl port-forward` | Diagnóstico em Kubernetes |

## 15. Tabela-resumo: qual ferramenta para qual sintoma

| Sintoma | Comece com |
|---|---|
| Sem rede nenhuma | `ip -br link`, `ip -br addr`, `ip route` |
| Nome não resolve | `getent hosts`, `dig`, `cat /etc/resolv.conf` |
| Porta não conecta | `nc -vz`, `ss -tlnp` (no destino), `tcpdump` |
| HTTP com erro | `curl -v` |
| Erro TLS | `openssl s_client -servername` |
| Lentidão | `curl -w`, `mtr`, `ss -ti`, `iperf3` |
| Perda intermitente | `mtr -r -c 200`, `ip -s link` |
| MTU suspeita | `ping -M do -s`, `tracepath` |
| Comportamento inexplicável | `tcpdump` nas duas pontas |

## 16. Considerações de segurança

- Use ferramentas de varredura e captura apenas em ambientes próprios ou com autorização formal.
- Evite `curl -k` e `--insecure` fora de diagnósticos pontuais.
- Containers de diagnóstico privilegiados devem ser removidos após o uso.
- Não cole saídas com tokens, cookies ou cabeçalhos `Authorization` em tickets ou chats.

## 17. Resumo

- `ip` (endereços, rotas, vizinhos), `ss` (sockets), `nc` (portas), `dig`/`getent` (nomes), `curl` (HTTP), `openssl s_client` (TLS), `ping`/`tracepath`/`mtr` (caminho), `tcpdump` (verdade de campo).
- Prefira saídas numéricas (`-n`) durante diagnósticos.
- Cada ferramenta responde a uma pergunta de uma camada; combine-as seguindo [[18 - Diagnóstico e Troubleshooting]].

## 18. Verificação de conhecimento

1. Qual comando mostra qual interface e gateway serão usados para alcançar `10.8.0.5`?
2. O `mtr` mostra 50% de perda no salto 4 e 0% no destino. Há problema?
3. Qual a diferença entre `dig exemplo.com` e `getent hosts exemplo.com`?
4. Escreva um filtro `tcpdump` que mostre apenas SYNs e RSTs para a porta 443.
5. Como medir separadamente o tempo de DNS, conexão TCP, TLS e resposta do servidor?
6. Por que `nc -vzu host 53` "funcionando" não prova que existe um servidor DNS?

> [!success]- Respostas
> 1. `ip route get 10.8.0.5`.
> 2. Provavelmente não: é limitação de respostas ICMP do roteador 4. Perda real persistiria até o destino.
> 3. `dig` consulta o servidor DNS diretamente; `getent` usa o NSS (inclui `/etc/hosts`, ordem do nsswitch, systemd-resolved), refletindo o que as aplicações veem.
> 4. `sudo tcpdump -n -i any 'tcp port 443 and (tcp[tcpflags] & (tcp-syn|tcp-rst) != 0)'`.
> 5. `curl -s -o /dev/null -w 'dns=%{time_namelookup} conn=%{time_connect} tls=%{time_appconnect} ttfb=%{time_starttransfer}\n' URL`.
> 6. Porque UDP não tem handshake; a ausência de ICMP Port Unreachable (que pode estar filtrado) é tratada como "aberta". Use `dig @host`.

## 19. Tópicos relacionados

- [[18 - Diagnóstico e Troubleshooting]]
- [[24 - Laboratórios Práticos]]
- [[07 - DNS]]
- [[05 - TCP e UDP]]
- [[12 - TLS e Certificados]]
- [[Mapa de Estudos de Redes]]
