---
title: Laboratórios Práticos de Redes
tags:
  - devops
  - networking
  - labs
aliases:
  - Laboratórios de Redes
  - Labs de Redes
---

# Laboratórios Práticos de Redes

> [!info] Como usar esta nota
> Sequência **progressiva** de laboratórios, da inspeção básica de IP até troubleshooting em Kubernetes. Cada laboratório indica **pré-requisitos**, **objetivo**, **passos**, **perguntas** e **solução/dicas separadas** (em blocos recolhíveis). Faça os exercícios antes de abrir as soluções.
> Referências: [[19 - Ferramentas de Diagnóstico]] (comandos) e [[18 - Diagnóstico e Troubleshooting]] (método).

## 0. Preparação do ambiente

### 0.1 Ambiente recomendado

Os laboratórios assumem **Linux**. Opções, da mais simples para a mais completa:

| Opção | Bom para | Observações |
|---|---|---|
| **WSL 2** (Windows) com Ubuntu | Labs 1–8 | Rede do WSL é virtualizada (NAT por padrão); alguns comandos mostram a rede da VM do WSL, não a do Windows |
| **VM Linux** (VirtualBox, Hyper‑V, VMware, Multipass) | Todos | Isolamento total; pode-se quebrar a rede sem afetar o computador |
| **Docker** (Docker Desktop ou Docker Engine no Linux) | Labs 9–12 | Containers servem como "hosts" de laboratório baratos |
| **kind / minikube / k3d** | Labs 13–15 | Kubernetes local em containers/VM; requer ~4 GB de RAM livres |

> [!warning] Segurança dos laboratórios
> - Execute comandos que alteram rede/firewall **somente em VMs, containers ou ambientes descartáveis**, nunca em servidores de produção ou na máquina de trabalho da empresa.
> - Não faça varreduras nem capturas em redes de terceiros (Wi‑Fi público, rede corporativa) sem autorização.
> - Use senhas de laboratório apenas em laboratório.

### 0.2 Pacotes (Debian/Ubuntu)

```bash
sudo apt update
sudo apt install -y iproute2 iputils-ping iputils-tracepath traceroute mtr-tiny \
  dnsutils netcat-openbsd curl openssl tcpdump nmap iperf3 conntrack nftables python3
```

### 0.3 Imagem de diagnóstico

```bash
docker pull nicolaka/netshoot
```

### 0.4 Sem ambiente disponível?

- Os laboratórios 1–6 podem ser estudados lendo as saídas de exemplo das notas e respondendo às perguntas.
- Para Docker/Kubernetes, ambientes online de prática (playgrounds) existem, mas mudam com frequência; verifique a disponibilidade atual.
- Laboratório 6 (subnetting) só precisa de papel e, opcionalmente, Python.

---

## Etapa 1 — Fundamentos

### Lab 1 — Inspecionar a própria configuração de rede

**Pré-requisitos:** [[00 - Fundamentos de Redes]], [[03 - Endereçamento IP e Sub-redes]]
**Objetivo:** identificar interfaces, IPs, prefixos, gateway e DNS.

**Passos:**

```bash
ip -br link
ip -br addr
ip route
cat /etc/resolv.conf
resolvectl status 2>/dev/null | head -30
curl -s https://ifconfig.me ; echo
```

**Perguntas:**

1. Quais interfaces existem e quais estão `UP`?
2. Qual seu IPv4, prefixo, endereço de rede e broadcast?
3. Qual o gateway padrão? Ele está dentro da sua sub-rede?
4. Seu IP local é público ou privado? Compare com o IP retornado por `ifconfig.me`. O que isso indica?
5. Qual servidor DNS está configurado?

> [!tip]- Dicas e solução
> - Use o prefixo (`/24`, por exemplo) para calcular rede e broadcast ([[04 - CIDR e Subnetting]]).
> - Se o IP local é privado (RFC 1918) e o `ifconfig.me` mostra outro IP, há **NAT** no caminho ([[10 - NAT e PAT]]). Se o IP público começar com `100.64.`–`100.127.` em algum ponto do caminho (visto em traceroute), há CGNAT.
> - No WSL 2, o gateway é a interface virtual do Windows; o DNS pode ser um IP interno do WSL.
> - Se `resolv.conf` mostrar `127.0.0.53`, o sistema usa systemd‑resolved; `resolvectl status` mostra os servidores reais.

### Lab 2 — Latência, perda e caminho

**Pré-requisitos:** [[02 - Modelo TCP-IP]], [[09 - Roteamento e Gateway]]

```bash
ping -c 10 <seu_gateway>
ping -c 10 1.1.1.1
tracepath -n 1.1.1.1
mtr -n -r -c 50 1.1.1.1
```

**Perguntas:**

1. Compare o RTT até o gateway com o RTT até `1.1.1.1`. De onde vem a diferença?
2. Quantos saltos até o destino? Algum salto não responde? Isso é um problema?
3. Qual a PMTU informada pelo `tracepath`?
4. Há perda em algum salto? Ela persiste até o destino?

> [!tip]- Dicas e solução
> - RTT até o gateway (LAN/Wi‑Fi) costuma ser de 1 a poucos ms; o restante é a rede do provedor e a distância.
> - Saltos com `no reply`/`???` são comuns e não indicam problema se os seguintes respondem.
> - PMTU 1500 é típica de Ethernet; 1492 indica PPPoE; valores menores indicam túneis.
> - Perda só em saltos intermediários = limitação de ICMP. Perda que persiste até o destino = perda real.

### Lab 3 — Encapsulação ao vivo com tcpdump

**Pré-requisitos:** [[01 - Modelo OSI]], [[05 - TCP e UDP]]
**Objetivo:** observar ARP, DNS, handshake TCP e HTTP em uma única requisição.

Terminal 1:

```bash
sudo tcpdump -n -e -i any '(arp or udp port 53 or tcp port 80)' -c 40
```

Terminal 2:

```bash
sudo ip neigh flush dev <sua_interface>     # força novo ARP (somente em laboratório)
curl -s -o /dev/null http://example.com/
```

**Perguntas:**

1. Para qual IP foi feito o ARP request? Por que não para o IP de `example.com`?
2. Identifique a consulta DNS (`A?`) e a resposta.
3. Identifique `[S]`, `[S.]` e `[.]` do three‑way handshake e a porta de origem efêmera.
4. Qual o MAC de destino dos frames enviados ao servidor? A quem pertence?

> [!tip]- Dicas e solução
> 1. Para o **gateway** — o servidor está em outra rede, então o frame vai ao gateway ([[08 - DHCP e ARP]]).
> 2. Linhas com `A? example.com.` e a resposta com o IP.
> 3. A porta efêmera aparece como porta de origem no SYN (ex.: `.51544 > ....80`).
> 4. O MAC do gateway: o IP de destino é do servidor, mas o MAC é do próximo salto.
> - Se o sistema usa systemd‑resolved, a consulta DNS pode aparecer em `127.0.0.53` (loopback) e depois saindo para o resolver real; com `-i any`, você verá ambas.

---

## Etapa 2 — Protocolos

### Lab 4 — Portas, sockets e erros TCP

**Pré-requisitos:** [[05 - TCP e UDP]], [[06 - Portas e Sockets]]

```bash
# Terminal 1: servidor HTTP simples apenas no loopback
python3 -m http.server 8000 --bind 127.0.0.1

# Terminal 2
ss -tlnp | grep 8000
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8000/
curl -v http://$(hostname -I | awk '{print $1}'):8000/      # pelo IP da interface
nc -vz 127.0.0.1 8001                                       # porta sem serviço
ss -tan | awk 'NR>1{print $1}' | sort | uniq -c
```

Repita com `--bind 0.0.0.0`.

**Perguntas:**

1. Por que o acesso pelo IP da interface falhou com `--bind 127.0.0.1`? Qual a mensagem?
2. O que aconteceu na porta 8001 e por quê?
3. Após várias requisições, aparecem sockets em `TIME-WAIT`? De quem?

> [!tip]- Dicas e solução
> 1. `Connection refused`: nada escuta naquele endereço; o kernel responde RST.
> 2. Também `refused`: nenhum socket na porta.
> 3. Sim, normalmente do lado que fechou primeiro (com `http.server` e `curl`, geralmente o servidor fecha após responder em HTTP/1.0). Use `ss -tan state time-wait`.

### Lab 5 — DNS na prática

**Pré-requisitos:** [[07 - DNS]]

```bash
dig example.com
dig +short example.com AAAA
dig example.com NS
dig +trace example.com
dig @1.1.1.1 example.com
dig nao-existe-$(date +%s).example.com     # NXDOMAIN
dig -x 1.1.1.1

# Hosts local vs DNS (em laboratório!)
echo "127.0.0.1 meu-teste.local" | sudo tee -a /etc/hosts
getent hosts meu-teste.local
dig +short meu-teste.local
# Desfaça depois: remova a linha adicionada de /etc/hosts
```

**Perguntas:**

1. Qual o TTL da resposta? Rode duas vezes: o que muda?
2. Na saída do `+trace`, identifique raiz, TLD e autoritativo.
3. Por que `getent` resolve `meu-teste.local` e `dig` não?
4. Qual o `status` para o nome inexistente? E o que aparece na seção AUTHORITY?

> [!tip]- Dicas e solução
> 1. O TTL diminui entre consultas: a resposta vem do cache do resolver.
> 2. Primeiro bloco: servidores `.` (raiz); depois `com.`; depois os NS de `example.com`.
> 3. `getent` usa NSS (`/etc/hosts` incluso); `dig` consulta apenas o DNS.
> 4. `NXDOMAIN`; o SOA da zona, cujo TTL/mínimo define o cache negativo.

### Lab 6 — Subnetting (papel e Python)

**Pré-requisitos:** [[04 - CIDR e Subnetting]]

Resolva manualmente e confira com Python:

1. Rede, broadcast, primeiro e último host de `10.12.200.130/21`.
2. Divida `172.20.0.0/22` em sub-redes `/24` e liste-as.
3. Planeje uma VPC `10.80.0.0/16` com 3 zonas, cada uma com uma subnet pública `/24`, uma privada `/20` e uma de dados `/24`, sem sobreposição.
4. Verifique se `10.80.16.0/20` e `10.80.20.0/24` se sobrepõem.

```bash
python3 - <<'EOF'
import ipaddress as ip
n = ip.ip_network('10.12.200.130/21', strict=False)
print(n, n.broadcast_address, list(n.hosts())[0], list(n.hosts())[-1])
print(list(ip.ip_network('172.20.0.0/22').subnets(new_prefix=24)))
print(ip.ip_network('10.80.16.0/20').overlaps(ip.ip_network('10.80.20.0/24')))
EOF
```

> [!tip]- Solução
> 1. `/21` → bloco de 8 no 3º octeto; 200 ∈ [200, 207] → rede `10.12.200.0`, broadcast `10.12.207.255`, hosts `10.12.200.1`–`10.12.207.254`.
> 2. `172.20.0.0/24`, `172.20.1.0/24`, `172.20.2.0/24`, `172.20.3.0/24`.
> 3. Uma possibilidade: públicas `10.80.0.0/24`, `10.80.1.0/24`, `10.80.2.0/24`; privadas `10.80.16.0/20`, `10.80.32.0/20`, `10.80.48.0/20`; dados `10.80.64.0/24`, `10.80.65.0/24`, `10.80.66.0/24`.
> 4. Sim: `10.80.16.0/20` vai de `10.80.16.0` a `10.80.31.255` e contém `10.80.20.0/24`.

### Lab 7 — HTTP e TLS

**Pré-requisitos:** [[11 - HTTP e HTTPS]], [[12 - TLS e Certificados]]

```bash
curl -v https://example.com/ -o /dev/null
curl -s -o /dev/null -w 'dns=%{time_namelookup} tcp=%{time_connect} tls=%{time_appconnect} ttfb=%{time_starttransfer} total=%{time_total}\n' https://example.com/
openssl s_client -connect example.com:443 -servername example.com </dev/null 2>/dev/null | openssl x509 -noout -subject -issuer -dates -ext subjectAltName

# Gerar e servir um certificado autoassinado localmente
openssl req -x509 -newkey rsa:2048 -nodes -keyout lab.key -out lab.crt -days 7 \
  -subj "/CN=localhost" -addext "subjectAltName=DNS:localhost"
openssl s_server -accept 8443 -cert lab.crt -key lab.key -www &
curl -v https://localhost:8443/                    # deve falhar: CA desconhecida
curl -v --cacert lab.crt https://localhost:8443/   # deve funcionar
curl -v --cacert lab.crt https://127.0.0.1:8443/   # deve falhar: IP não está no SAN
kill %1
```

**Perguntas:**

1. Qual versão de TLS e qual protocolo HTTP (ALPN) foram negociados com `example.com`?
2. Quanto tempo levou cada fase?
3. Por que o terceiro `curl` falhou mesmo com `--cacert`?

> [!tip]- Dicas e solução
> 1. Veja as linhas `SSL connection using TLSv1.3 / ...` e `ALPN: server accepted ...`.
> 2. Subtraia os tempos acumulados (tcp−dns, tls−tcp, ttfb−tls).
> 3. O SAN do certificado contém apenas `DNS:localhost`; o nome usado (`127.0.0.1`) não está nele.

### Lab 8 — Túnel SSH (local)

**Pré-requisitos:** [[13 - SSH]]. Requer um servidor SSH — pode ser o próprio host (`sudo apt install openssh-server`) ou um container/VM.

```bash
# Serviço "privado" escutando só em loopback
python3 -m http.server 9000 --bind 127.0.0.1 &

# Túnel: porta local 9100 → 127.0.0.1:9000 do servidor SSH (aqui, o próprio host)
ssh -N -L 9100:127.0.0.1:9000 $USER@localhost &
curl -s http://127.0.0.1:9100/ | head -5
kill %2 %1
```

**Pergunta:** quem resolve/conecta o destino `127.0.0.1:9000` do `-L` — o cliente ou o servidor SSH?

> [!tip]- Solução
> O **servidor SSH**. Num cenário real com bastion, `127.0.0.1` no destino seria o próprio bastion; para alcançar um banco interno, use o IP/nome do banco como visto pelo bastion.

---

## Etapa 3 — Infraestrutura (com namespaces e containers)

### Lab 9 — Construir uma "rede" com network namespaces

**Pré-requisitos:** [[09 - Roteamento e Gateway]], [[10 - NAT e PAT]], [[14 - Firewall e Filtragem de Pacotes]]. Requer **root em uma VM de laboratório**.
**Objetivo:** entender o que Docker e Kubernetes fazem por baixo dos panos.

```bash
# Dois "hosts" isolados ligados por uma bridge (switch virtual)
sudo ip netns add h1
sudo ip netns add h2
sudo ip link add br-lab type bridge
sudo ip link set br-lab up
sudo ip addr add 10.99.0.1/24 dev br-lab            # IP do "gateway" no host

for n in 1 2; do
  sudo ip link add veth$n type veth peer name veth$n-br
  sudo ip link set veth$n netns h$n
  sudo ip link set veth$n-br master br-lab
  sudo ip link set veth$n-br up
  sudo ip netns exec h$n ip addr add 10.99.0.1$n/24 dev veth$n
  sudo ip netns exec h$n ip link set veth$n up
  sudo ip netns exec h$n ip link set lo up
  sudo ip netns exec h$n ip route add default via 10.99.0.1
done

# Teste L2/L3 entre "hosts"
sudo ip netns exec h1 ping -c 2 10.99.0.12
sudo ip netns exec h1 ip neigh

# Saída para a Internet: encaminhamento + NAT (substitua eth0 pela sua interface de saída)
sudo sysctl -w net.ipv4.ip_forward=1
sudo iptables -t nat -A POSTROUTING -s 10.99.0.0/24 ! -o br-lab -j MASQUERADE
sudo ip netns exec h1 ping -c 2 1.1.1.1
```

> [!note] Se o ping para a Internet falhar
> Em hosts com Docker ou firewall com política `DROP` na chain `FORWARD`, adicione regras de laboratório:
> `sudo iptables -I FORWARD -i br-lab -j ACCEPT` e `sudo iptables -I FORWARD -o br-lab -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT`.

**Experimentos:**

1. Em `h2`, inicie `python3 -m http.server 8080` (com `sudo ip netns exec h2 ...`) e acesse de `h1` com `curl`.
2. Bloqueie `h1 → h2:8080` com uma regra e observe se o erro é *refused* ou *timeout*. (Dica: tráfego bridged pode exigir `br_netfilter` para passar pelo iptables; alternativamente, aplique a regra **dentro** de `h2`: `sudo ip netns exec h2 iptables -A INPUT -p tcp --dport 8080 -j DROP`.)
3. Troque `DROP` por `REJECT` e compare.
4. Capture com `tcpdump -i br-lab -n` durante os testes.

**Limpeza:**

```bash
sudo ip netns del h1; sudo ip netns del h2
sudo ip link del br-lab
sudo iptables -t nat -D POSTROUTING -s 10.99.0.0/24 ! -o br-lab -j MASQUERADE
# Remova também as regras FORWARD que você adicionou, se for o caso (iptables -D ...).
```

> [!tip]- Discussão
> - Você acabou de criar o equivalente à rede bridge do Docker: namespaces (containers), veth (cabos), bridge (switch), gateway, NAT (MASQUERADE).
> - DROP → `curl` espera até o timeout; REJECT → erro imediato (`Connection refused` com `--reject-with tcp-reset`, ou `No route to host`/`Connection refused` conforme o ICMP enviado por padrão).

---

## Etapa 4 — Docker

### Lab 10 — Redes Docker: padrão vs customizada

**Pré-requisitos:** [[20 - Redes em Docker]]

```bash
# Bridge padrão: sem DNS por nome
docker run -d --name a1 nginx:1.27
docker run --rm --name t1 nicolaka/netshoot sh -c 'curl -s -m 3 http://a1 || echo FALHOU'

# Rede customizada: DNS embutido
docker network create labnet
docker run -d --name a2 --network labnet nginx:1.27
docker run --rm --network labnet nicolaka/netshoot sh -c 'cat /etc/resolv.conf; dig +short a2; curl -sI http://a2 | head -1'

# Inspeção
docker network inspect labnet
ip -br link show type veth
sudo iptables -t nat -S | grep -i docker   # ou: sudo nft list ruleset | grep -i docker
```

**Perguntas:**

1. Por que `curl http://a1` falhou?
2. Qual o nameserver dentro do container na `labnet`?
3. Quantas interfaces `veth` apareceram no host?

> [!tip]- Solução
> 1. A bridge padrão não tem resolução de nomes de containers.
> 2. `127.0.0.11` (DNS embutido).
> 3. Uma por container em execução conectado a uma bridge (pode haver mais, de outros containers).

**Limpeza:** `docker rm -f a1 a2; docker network rm labnet`

### Lab 11 — Publicação de portas e o erro do bind em 127.0.0.1

```bash
# App que escuta só em 127.0.0.1 DENTRO do container
docker run -d --name ruim -p 8081:8000 python:3.12-slim python -m http.server 8000 --bind 127.0.0.1
# App correta
docker run -d --name bom  -p 8082:8000 python:3.12-slim python -m http.server 8000 --bind 0.0.0.0

curl -sv http://127.0.0.1:8081/ -o /dev/null
curl -sv http://127.0.0.1:8082/ -o /dev/null
docker run --rm --network container:ruim nicolaka/netshoot ss -tlnp
docker port bom
```

**Perguntas:**

1. Qual erro aparece na porta 8081? Por que a publicação não ajuda?
2. Como publicar a 8082 apenas para o próprio host?

> [!tip]- Solução
> 1. Tipicamente `Connection reset by peer` ou `Empty reply from server` (o texto exato depende da versão do Docker e do mecanismo de encaminhamento). O tráfego chega pela `eth0` do container, mas a aplicação só aceita no loopback do container.
> 2. `-p 127.0.0.1:8082:8000`.

**Limpeza:** `docker rm -f ruim bom`

### Lab 12 — Compose com redes segmentadas

Crie `compose.yaml`:

```yaml
services:
  web:
    image: nginx:1.27
    ports: ["127.0.0.1:8080:80"]
    networks: [front]
  api:
    image: nicolaka/netshoot
    command: ["sleep", "infinity"]
    networks: [front, back]
  db:
    image: redis:7
    networks: [back]
networks:
  front:
  back:
    internal: true
```

```bash
docker compose up -d
docker compose exec api sh -c 'nc -vz db 6379; nc -vz web 80; curl -sI -m 5 https://example.com | head -1'
docker compose exec web sh -c 'getent hosts db || echo "web não enxerga db"'
docker compose down
```

**Perguntas:**

1. Por que `web` não resolve `db`?
2. O `api` consegue acessar a Internet? Por quê?

> [!tip]- Solução
> 1. Estão em redes diferentes; o DNS embutido só resolve containers de redes compartilhadas.
> 2. Sim, porque também está na rede `front` (não interna), que tem saída via NAT. Se `api` estivesse **apenas** em `back` (`internal: true`), não teria saída.

---

## Etapa 5 — Kubernetes

> [!info] Pré-requisitos para os labs 13–15
> - Docker funcionando e **kind** (ou minikube/k3d) e **kubectl** instalados. Consulte a documentação oficial de cada ferramenta para instalação.
> - Notas: [[21 - Redes em Kubernetes]].
> - **Sem cluster?** Leia os manifests e as saídas descritas, desenhe o fluxo de rede de cada exercício e responda às perguntas; a lógica de diagnóstico é a mesma.
> - Atenção: o CNI padrão do **kind** (kindnet) **não** implementa NetworkPolicy; para o Lab 15, crie o cluster com outro CNI (ex.: Calico ou Cilium, conforme a documentação deles) ou use minikube com `--cni=calico`.

### Lab 13 — Pods, Services e DNS

```bash
kind create cluster --name redes
kubectl create namespace loja
kubectl -n loja create deployment api --image=nginx:1.27 --replicas=2
kubectl -n loja expose deployment api --port=80 --target-port=80
kubectl -n loja get pods -o wide
kubectl -n loja get svc,endpointslices

kubectl -n loja run netshoot --rm -it --image=nicolaka/netshoot -- bash
# Dentro do Pod:
#   cat /etc/resolv.conf
#   dig +short api.loja.svc.cluster.local
#   curl -sI http://api | head -1
#   curl -sI http://<IP_de_um_pod> | head -1
#   ping -c 2 <ClusterIP>        # observe o resultado
```

**Perguntas:**

1. Quais são os domínios de busca e o `ndots`?
2. O DNS do Service retorna o IP de um Pod ou o ClusterIP?
3. O `ping` ao ClusterIP funciona? Por quê?

> [!tip]- Solução
> 1. `loja.svc.cluster.local svc.cluster.local cluster.local` e `ndots:5`.
> 2. O ClusterIP (para um Service não headless).
> 3. Normalmente não: o ClusterIP é virtual e só as portas do Service são encaminhadas.

### Lab 14 — Troubleshooting: quebre e conserte

Execute cada "defeito", observe o sintoma e diagnostique **antes** de ler a solução. **Corrija cada defeito antes de passar ao próximo**, para que os sintomas não se misturem.

**Defeito A — selector errado**

```bash
kubectl -n loja patch svc api -p '{"spec":{"selector":{"app":"api-errado"}}}'
kubectl -n loja run t --rm -it --image=nicolaka/netshoot -- curl -s -m 5 http://api || true
```

> [!tip]- Diagnóstico A
> `kubectl -n loja get endpointslices -l kubernetes.io/service-name=api` mostra nenhum endpoint. Compare `kubectl get svc api -o jsonpath='{.spec.selector}'` com `kubectl get pods --show-labels`. Correção: `kubectl -n loja patch svc api -p '{"spec":{"selector":{"app":"api"}}}'`.

**Defeito B — targetPort errado**

```bash
kubectl -n loja patch svc api --type=json -p '[{"op":"replace","path":"/spec/ports/0/targetPort","value":8080}]'
kubectl -n loja run t --rm -it --image=nicolaka/netshoot -- curl -sv -m 5 http://api 2>&1 | tail -3
```

> [!tip]- Diagnóstico B
> Os endpoints existem, mas apontam para a porta 8080, onde o nginx não escuta → `Connection refused`. Teste `curl http://<IP_do_pod>:80` (funciona) vs `:8080` (falha). Correção: `targetPort` 80.

**Defeito C — readiness falhando**

```bash
kubectl -n loja patch deployment api --type=json -p '[{"op":"add","path":"/spec/template/spec/containers/0/readinessProbe","value":{"httpGet":{"path":"/nao-existe","port":80},"periodSeconds":5}}]'
kubectl -n loja get pods -w     # Ctrl+C após alguns segundos
```

> [!tip]- Diagnóstico C
> Os novos Pods ficam `0/1 READY`; o rollout não avança (os Pods antigos continuam atendendo até o limite de indisponibilidade). `kubectl describe pod` mostra `Readiness probe failed: HTTP probe failed with statuscode: 404`. Correção: ajustar o caminho (`/`) ou desfazer a mudança com `kubectl -n loja rollout undo deployment api`.

### Lab 15 — NetworkPolicy (requer CNI com suporte)

Os Pods criados por `kubectl create deployment api` já recebem o label `app=api`, usado pela política abaixo. Garanta que os defeitos do Lab 14 foram corrigidos.

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: loja
spec:
  podSelector: {}
  policyTypes: [Ingress]
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-allow-frontend
  namespace: loja
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes: [Ingress]
  ingress:
    - from:
        - podSelector:
            matchLabels:
              role: frontend
      ports:
        - protocol: TCP
          port: 80
EOF

kubectl -n loja run sem-label --rm -it --image=nicolaka/netshoot -- curl -s -m 5 http://api || echo "BLOQUEADO"
kubectl -n loja run com-label --rm -it --labels=role=frontend --image=nicolaka/netshoot -- curl -sI -m 5 http://api | head -1
```

**Perguntas:**

1. O que acontece com o Pod sem label? O erro é imediato ou após o timeout?
2. Escreva uma política de **egress** default deny para o namespace que ainda permita DNS.

> [!tip]- Solução
> 1. Bloqueado; normalmente **timeout** (a maioria das implementações descarta os pacotes). Se ambos funcionarem, seu CNI provavelmente não implementa NetworkPolicy.
> 2. Exemplo:
> ```yaml
> apiVersion: networking.k8s.io/v1
> kind: NetworkPolicy
> metadata:
>   name: default-deny-egress-allow-dns
>   namespace: loja
> spec:
>   podSelector: {}
>   policyTypes: [Egress]
>   egress:
>     - to:
>         - namespaceSelector:
>             matchLabels:
>               kubernetes.io/metadata.name: kube-system
>           podSelector:
>             matchLabels:
>               k8s-app: kube-dns
>       ports:
>         - protocol: UDP
>           port: 53
>         - protocol: TCP
>           port: 53
> ```

**Limpeza:** `kind delete cluster --name redes`

---

## Desafios integradores

Desafios que combinam vários conceitos. Escreva o diagnóstico/plano antes de consultar as dicas.

### Desafio 1 — "O site caiu às 3h"

Usuários recebem `NET::ERR_CERT_DATE_INVALID` no navegador. O app roda em Kubernetes atrás de um Ingress com cert-manager. Descreva a investigação passo a passo e a prevenção.

> [!tip]- Dicas
> `openssl s_client ... | openssl x509 -noout -dates` para confirmar a expiração; `kubectl get certificate,certificaterequest,order,challenge -A`; eventos do cert-manager (falha no desafio HTTP‑01 por regra de Ingress/NetworkPolicy/DNS? limite de taxa da CA?). Prevenção: alertas de expiração (30/7 dias), monitorar o estado `Ready` dos Certificates. Ver [[12 - TLS e Certificados]].

### Desafio 2 — "Funciona na VPN, não funciona no escritório"

Um serviço interno `10.30.5.20` é acessível de casa via VPN, mas não a partir da rede do escritório, onde a sub-rede local é `10.30.0.0/16`.

> [!tip]- Dicas
> Sobreposição de faixas: no escritório, `10.30.5.20` é considerado **local** (mesma sub-rede `/16`), então o host faz ARP em vez de enviar ao gateway/VPN. Verifique com `ip route get 10.30.5.20`. Solução definitiva: replanejar endereçamento; paliativos: rotas mais específicas ou NAT. Ver [[04 - CIDR e Subnetting]], [[09 - Roteamento e Gateway]].

### Desafio 3 — "Deploy gera erros 502 por 30 segundos"

A cada rolling update, o Ingress devolve 502 por alguns segundos.

> [!tip]- Dicas
> Pods recebem SIGTERM e encerram antes de serem removidos dos endpoints/do controlador. Adicione `preStop` (ex.: `sleep` de alguns segundos), graceful shutdown na aplicação, readiness probe correta, `terminationGracePeriodSeconds` adequado; verifique keep‑alive entre o controlador e os Pods. Ver [[16 - Balanceamento de Carga]].

### Desafio 4 — "Pipeline não baixa dependências"

Um job de CI self‑hosted em subnet privada falha em `npm ci` com `ETIMEDOUT`, mas `curl https://registry.npmjs.org` funciona no host do runner.

> [!tip]- Dicas
> O job roda num container: verifique DNS e proxy **dentro** do container (`HTTP(S)_PROXY`, `NO_PROXY`, configuração do npm), MTU da rede Docker vs MTU da VPC/VPN, e se a rede do container tem saída (NAT, `ip_forward`). Compare `curl` no host vs `docker run --rm nicolaka/netshoot curl -v https://registry.npmjs.org`. Ver [[15 - Proxy e Reverse Proxy]], [[20 - Redes em Docker]].

### Desafio 5 — Projeto de rede

Projete a rede de uma aplicação com: frontend público, API, worker que consome uma fila, PostgreSQL gerenciado e chamadas a uma API de parceiro que exige allowlist de IP. Entregue: plano de CIDRs (3 zonas), tabelas de rotas, Security Groups, onde termina o TLS, como o parceiro recebe o IP de saída, e como o time acessa o banco para manutenção sem expor portas.

> [!tip]- Elementos esperados
> VPC `/16` com subnets públicas (LB, NAT por zona), privadas de app e isoladas de dados; SGs encadeados (LB → API; API/worker → DB); TLS no LB (e opcionalmente re‑encryption); IPs fixos dos NAT Gateways informados ao parceiro; endpoints privados para serviços do provedor; acesso ao banco via gerenciador de sessão/bastion com túnel SSH ou proxy de identidade; flow logs habilitados. Ver [[22 - Redes em Cloud e DevOps]], [[23 - Segurança de Redes]].

---

## Registro de progresso

| Lab | Concluído | Observações |
|---|---|---|
| 1 — Inspeção de IP | ☐ | |
| 2 — Latência e caminho | ☐ | |
| 3 — tcpdump e encapsulação | ☐ | |
| 4 — Portas e erros TCP | ☐ | |
| 5 — DNS | ☐ | |
| 6 — Subnetting | ☐ | |
| 7 — HTTP e TLS | ☐ | |
| 8 — Túnel SSH | ☐ | |
| 9 — Network namespaces | ☐ | |
| 10 — Redes Docker | ☐ | |
| 11 — Publicação de portas | ☐ | |
| 12 — Compose segmentado | ☐ | |
| 13 — Pods, Services, DNS | ☐ | |
| 14 — Quebre e conserte | ☐ | |
| 15 — NetworkPolicy | ☐ | |
| Desafios 1–5 | ☐ | |

## Tópicos relacionados

- [[Mapa de Estudos de Redes]]
- [[18 - Diagnóstico e Troubleshooting]]
- [[19 - Ferramentas de Diagnóstico]]
- [[20 - Redes em Docker]]
- [[21 - Redes em Kubernetes]]
- [[22 - Redes em Cloud e DevOps]]
