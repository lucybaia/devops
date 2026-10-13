---
title: CIDR e Subnetting
tags:
  - devops
  - networking
  - subnetting
aliases:
  - CIDR
  - Subnetting
  - Cálculo de Sub-redes
---

# CIDR e Subnetting

> [!info] Posição na trilha
> **Etapa 1 — Fundamentos.** Pré-requisito: [[03 - Endereçamento IP e Sub-redes]]. Próxima: [[05 - TCP e UDP]].

## 1. Definição

- **CIDR (Classless Inter‑Domain Routing)**, definido na RFC 4632, é o esquema de endereçamento e roteamento em que o tamanho do prefixo de rede é **arbitrário** e escrito após uma barra: `10.0.0.0/16`.
- **Subnetting** é dividir um bloco de endereços em blocos menores (sub-redes).
- **Supernetting / sumarização** é o inverso: agrupar vários blocos contíguos em um prefixo maior para simplificar rotas.

O número após a barra é a quantidade de **bits de rede** (bits em 1 na máscara).

## 2. Por que isso importa

Em DevOps você vai, com frequência:

- Definir o CIDR de uma VPC e dividi-lo em subnets públicas e privadas por zona de disponibilidade.
- Escolher faixas para Pods e Services de um cluster Kubernetes sem colidir com a VPC ou a VPN.
- Escrever regras de firewall e Security Groups com blocos CIDR (`10.0.0.0/8`, `203.0.113.10/32`).
- Ler um `/20` e saber de imediato quantos IPs ele tem.

## 3. Como funciona

### 3.1 Fórmulas fundamentais (IPv4)

Seja **p** o tamanho do prefixo e **h = 32 − p** os bits de host:

| Grandeza | Fórmula |
|---|---|
| Total de endereços no bloco | 2^h |
| Hosts utilizáveis (rede tradicional) | 2^h − 2 (exceto /31 e /32) |
| Número de sub-redes ao "pegar emprestado" n bits | 2^n |
| Tamanho do bloco (incremento) no octeto "interessante" | 256 − valor da máscara naquele octeto |

### 3.2 Tabela de referência

| Prefixo | Máscara | Endereços | Hosts úteis | Uso típico |
|---|---|---|---|---|
| /8 | 255.0.0.0 | 16.777.216 | 16.777.214 | `10.0.0.0/8` inteiro |
| /12 | 255.240.0.0 | 1.048.576 | — | `172.16.0.0/12` |
| /16 | 255.255.0.0 | 65.536 | 65.534 | VPC grande |
| /20 | 255.255.240.0 | 4.096 | 4.094 | Subnet grande (ex.: nós/Pods) |
| /22 | 255.255.252.0 | 1.024 | 1.022 | |
| /23 | 255.255.254.0 | 512 | 510 | |
| /24 | 255.255.255.0 | 256 | 254 | LAN clássica, subnet comum |
| /25 | 255.255.255.128 | 128 | 126 | |
| /26 | 255.255.255.192 | 64 | 62 | |
| /27 | 255.255.255.224 | 32 | 30 | |
| /28 | 255.255.255.240 | 16 | 14 | Menor subnet permitida em vários provedores (ex.: AWS, Azure; verifique os limites atuais) |
| /29 | 255.255.255.248 | 8 | 6 | |
| /30 | 255.255.255.252 | 4 | 2 | Enlace ponto a ponto (tradicional) |
| /31 | 255.255.255.254 | 2 | 2* | Ponto a ponto (RFC 3021) |
| /32 | 255.255.255.255 | 1 | 1* | Host único (rotas, regras de firewall) |

\* Casos especiais sem endereço de rede/broadcast.

> [!tip] Atalho mental
> Cada bit a menos no prefixo **dobra** o bloco. /24 = 256 → /23 = 512 → /22 = 1024. Cada bit a mais **divide pela metade**: /25 = 128, /26 = 64.

### 3.3 Valores possíveis de um octeto de máscara

Os bits de máscara são contínuos a partir da esquerda, então um octeto só pode ser:

| Bits em 1 | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 |
|---|---|---|---|---|---|---|---|---|---|
| Valor | 0 | 128 | 192 | 224 | 240 | 248 | 252 | 254 | 255 |

## 4. Conceitos centrais

- **Prefixo mais longo = rede menor e mais específica.** `/28` é mais específico que `/24`.
- **Alinhamento:** um bloco `/n` sempre começa em um múltiplo do seu tamanho. `10.0.0.64/26` é válido; `10.0.0.70/26` não é endereço de rede (o host `.70` pertence a `10.0.0.64/26`).
- **Sobreposição:** dois blocos se sobrepõem se um contém (parte do) outro. Em CIDR, blocos alinhados ou são disjuntos ou um contém o outro.
- **Longest prefix match:** o roteador escolhe a rota com o prefixo mais longo que contém o destino. Ver [[09 - Roteamento e Gateway]].

## 5. Exemplos práticos (passo a passo)

### Exemplo 1 — Dado um IP com prefixo, encontrar a rede

**Problema:** `192.168.37.200/27`. Encontre rede, broadcast, primeiro e último host.

1. `/27` → máscara `255.255.255.224`. O octeto "interessante" é o 4º (valor 224).
2. Tamanho do bloco: 256 − 224 = **32**.
3. Blocos no 4º octeto: 0, 32, 64, 96, 128, 160, **192**, 224.
4. 200 está entre 192 e 223 → **rede `192.168.37.192`**.
5. Broadcast = próximo bloco − 1 = **`192.168.37.223`**.
6. Hosts: **`192.168.37.193` – `192.168.37.222`** (30 hosts).

### Exemplo 2 — Octeto interessante não é o último

**Problema:** `10.20.77.5/20`.

1. `/20` → `255.255.240.0`. Octeto interessante: o 3º (valor 240).
2. Bloco: 256 − 240 = **16** no 3º octeto.
3. Múltiplos de 16: 0, 16, 32, 48, **64**, 80...
4. 77 está em [64, 79] → rede **`10.20.64.0/20`**.
5. Broadcast: **`10.20.79.255`**.
6. Hosts: `10.20.64.1` – `10.20.79.254` (4.094 hosts).

### Exemplo 3 — Dividir uma rede em N sub-redes iguais

**Problema:** dividir `10.0.0.0/24` em 4 sub-redes.

1. 4 sub-redes = 2² → pegar **2 bits** emprestados: /24 + 2 = **/26**.
2. Tamanho de cada: 2^(32−26) = 64.
3. Resultado:

| Sub-rede | Rede | Hosts | Broadcast |
|---|---|---|---|
| 1 | 10.0.0.0/26 | .1 – .62 | .63 |
| 2 | 10.0.0.64/26 | .65 – .126 | .127 |
| 3 | 10.0.0.128/26 | .129 – .190 | .191 |
| 4 | 10.0.0.192/26 | .193 – .254 | .255 |

### Exemplo 4 — Dimensionar pela quantidade de hosts

**Problema:** preciso de uma sub-rede para 500 hosts.

1. Menor potência de 2 com 2^h − 2 ≥ 500 → 2⁹ = 512 → h = 9.
2. Prefixo: 32 − 9 = **/23** (510 hosts úteis).
3. Em cloud, desconte os endereços reservados pelo provedor e deixe margem de crescimento — provavelmente escolheria um `/22`.

### Exemplo 5 — VLSM (sub-redes de tamanhos diferentes)

**Problema:** a partir de `172.16.0.0/24`, criar: A com 100 hosts, B com 50, C com 20, e 2 enlaces ponto a ponto.

Regra: **aloque do maior para o menor** para manter o alinhamento.

| Rede | Necessidade | Prefixo | Bloco alocado |
|---|---|---|---|
| A | 100 hosts | /25 (126) | 172.16.0.0/25 (.0 – .127) |
| B | 50 hosts | /26 (62) | 172.16.0.128/26 (.128 – .191) |
| C | 20 hosts | /27 (30) | 172.16.0.192/27 (.192 – .223) |
| Enlace 1 | 2 hosts | /30 | 172.16.0.224/30 (.224 – .227) |
| Enlace 2 | 2 hosts | /30 | 172.16.0.228/30 (.228 – .231) |
| Livre | — | — | 172.16.0.232 – .255 |

### Exemplo 6 — Plano de VPC realista

**Problema:** VPC `10.50.0.0/16` em 3 zonas de disponibilidade (AZ), com subnets públicas pequenas e privadas grandes.

```text
10.50.0.0/16  (65.536 endereços)
├── Públicas (/24 cada — load balancers, NAT gateways)
│   ├── 10.50.0.0/24   AZ-a
│   ├── 10.50.1.0/24   AZ-b
│   └── 10.50.2.0/24   AZ-c
├── Privadas de aplicação (/20 cada — nós e, se o CNI usar IPs da VPC, Pods)
│   ├── 10.50.16.0/20  AZ-a   (10.50.16.0 – 10.50.31.255)
│   ├── 10.50.32.0/20  AZ-b   (10.50.32.0 – 10.50.47.255)
│   └── 10.50.48.0/20  AZ-c   (10.50.48.0 – 10.50.63.255)
├── Privadas de dados (/24 cada)
│   ├── 10.50.64.0/24  AZ-a
│   ├── 10.50.65.0/24  AZ-b
│   └── 10.50.66.0/24  AZ-c
└── Reservado para crescimento: 10.50.128.0/17
```

Note que os `/20` começam em múltiplos de 16 no terceiro octeto (16, 32, 48) — isso é o **alinhamento**.

### Exemplo 7 — Sumarização de rotas

`10.1.0.0/24`, `10.1.1.0/24`, `10.1.2.0/24`, `10.1.3.0/24` → podem ser anunciadas como **`10.1.0.0/22`** (os primeiros 22 bits são iguais em todas). Isso reduz o tamanho das tabelas de rotas.

### Exemplo 8 — IPv6

Um provedor entrega `2001:db8:abcd::/48`. Quantas sub-redes `/64` cabem? 2^(64−48) = **65.536**. Cada uma com 2⁶⁴ endereços. Em IPv6, o planejamento é sobre **quantidade de sub-redes**, não de hosts.

## 6. Comandos e demonstrações

```bash
# ipcalc (Debian/Ubuntu: pacote ipcalc; RHEL/Fedora têm uma implementação diferente com outras opções)
ipcalc 192.168.37.200/27

# Python como calculadora confiável (módulo padrão ipaddress)
python3 -c "import ipaddress as i; n=i.ip_network('192.168.37.200/27', strict=False); print(n, n.netmask, n.broadcast_address, n.num_addresses)"

# Listar as sub-redes /26 de um /24
python3 -c "import ipaddress as i; [print(s) for s in i.ip_network('10.0.0.0/24').subnets(new_prefix=26)]"

# Verificar se um IP pertence a um bloco
python3 -c "import ipaddress as i; print(i.ip_address('10.20.77.5') in i.ip_network('10.20.64.0/20'))"

# Verificar sobreposição entre dois blocos
python3 -c "import ipaddress as i; print(i.ip_network('10.0.0.0/16').overlaps(i.ip_network('10.0.128.0/20')))"
```

## 7. Interpretação

Saída do primeiro comando Python:

```text
192.168.37.192/27 255.255.255.224 192.168.37.223 32
```

→ rede, máscara, broadcast e total de endereços (32; hosts úteis = 30).

O argumento `strict=False` permite passar um IP de host (`.200`) e obter a rede que o contém; com `strict=True` (padrão), o Python rejeita bits de host ligados, o que é útil para validar CIDRs de configuração.

## 8. Problemas comuns

| Erro | Consequência |
|---|---|
| Usar CIDR não alinhado em configuração (`10.0.0.70/26` como rede) | Ferramentas recusam ou corrigem silenciosamente para `10.0.0.64/26` |
| VPC pequena demais (`/24`) com Kubernetes cujo CNI usa IPs da VPC | Esgotamento de IPs; Pods não sobem |
| Faixas sobrepostas entre VPCs | Peering e roteamento impossíveis |
| Regra de firewall `0.0.0.0/0` em vez de `/32` | Exposição de serviço para toda a Internet |
| Confundir `/32` com `/24` em regra de liberação | Liberar 256 IPs em vez de 1 |

## 9. Troubleshooting

- Ao receber um erro de "CIDR inválido" em Terraform/cloud, verifique **alinhamento** e **sobreposição** com Python `ipaddress`.
- Antes de criar VPN ou peering, liste todas as faixas envolvidas e teste `overlaps` entre cada par.
- Se um host "fora da faixa" não for alcançado, confira a máscara configurada nele e no gateway.

## 10. Aplicações em DevOps

- **Terraform** tem as funções `cidrsubnet()`, `cidrhost()` e `cidrnetmask()` para calcular subnets automaticamente. Ex.: `cidrsubnet("10.50.0.0/16", 8, 2)` → `10.50.2.0/24` (adiciona 8 bits ao prefixo e escolhe a sub-rede de número 2).
- **Kubernetes:** `--pod-network-cidr` (kubeadm) e `--service-cluster-ip-range` (kube‑apiserver) definem faixas que **não devem colidir** entre si nem com a rede dos nós.
- **Docker:** `docker network create --subnet 10.10.0.0/24 minha-rede`.

## 11. Considerações de segurança

- Prefira os blocos **mais estreitos** possíveis em regras de firewall (menor privilégio).
- Documente o plano de endereçamento: regras com CIDRs "mágicos" sem explicação viram dívida de segurança.
- Lembre que `0.0.0.0/0` (IPv4) e `::/0` (IPv6) significam "qualquer endereço". Liberar só o IPv4 e esquecer o IPv6 (ou vice-versa) é um erro comum.

## 12. Resumo

- `/p` = p bits de rede; endereços = 2^(32−p).
- Bloco = 256 − valor da máscara no octeto interessante; redes começam em múltiplos do bloco.
- Subnetting: pegar n bits → 2^n sub-redes. VLSM: alocar do maior para o menor.
- Em cloud, planeje com folga, sem sobreposição, alinhado por AZ.

## 13. Verificação de conhecimento

Exercícios (resolva no papel, depois confira com Python):

1. Rede, broadcast e faixa de hosts de `10.1.130.77/18`.
2. Quantos hosts úteis em um `/21`?
3. Divida `192.168.100.0/24` em 8 sub-redes. Qual o prefixo e qual a 6ª sub-rede?
4. Qual o menor prefixo que acomoda 1.000 hosts?
5. `10.0.0.0/16` e `10.0.255.0/24` se sobrepõem?
6. Sumarize `172.20.8.0/24` a `172.20.15.0/24`.
7. Calcule `cidrsubnet("10.0.0.0/16", 4, 3)`.

> [!success]- Respostas
> 1. `/18` → máscara `255.255.192.0`, bloco 64 no 3º octeto. 130 ∈ [128, 191] → rede `10.1.128.0`, broadcast `10.1.191.255`, hosts `10.1.128.1` – `10.1.191.254`.
> 2. 2^11 − 2 = 2.046.
> 3. 8 = 2³ → `/27` (blocos de 32). Sub-redes: .0, .32, .64, .96, .128, **.160** → 6ª é `192.168.100.160/27`.
> 4. 2^10 − 2 = 1.022 ≥ 1.000 → `/22`.
> 5. Sim: `10.0.255.0/24` está contido em `10.0.0.0/16` (10.0.0.0 – 10.0.255.255).
> 6. 8 a 15 no 3º octeto = 8 redes /24 alinhadas em múltiplo de 8 → `172.20.8.0/21`.
> 7. /16 + 4 bits = /20; blocos de 16 no 3º octeto; índice 3 → `10.0.48.0/20`.

## 14. Tópicos relacionados

- Anterior: [[03 - Endereçamento IP e Sub-redes]]
- Próxima: [[05 - TCP e UDP]]
- [[09 - Roteamento e Gateway]]
- [[22 - Redes em Cloud e DevOps]]
- [[Mapa de Estudos de Redes]]
