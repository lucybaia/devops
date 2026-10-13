---
title: Balanceamento de Carga
tags:
  - devops
  - networking
  - load-balancing
aliases:
  - Load Balancer
  - Balanceador de Carga
  - Health Checks
---

# Balanceamento de Carga

> [!info] Posição na trilha
> **Etapa 3 — Infraestrutura de Rede.** Pré-requisitos: [[05 - TCP e UDP]], [[15 - Proxy e Reverse Proxy]]. Próxima: [[17 - VLAN e Segmentação de Redes]].

## 1. Definição

Um **load balancer (balanceador de carga)** distribui conexões ou requisições entre um conjunto de servidores (**backends**, **targets**, **upstreams**, **pool**) para obter:

- **Escalabilidade horizontal:** mais instâncias = mais capacidade.
- **Alta disponibilidade:** instâncias com falha são retiradas automaticamente via **health checks**.
- **Manutenção sem downtime:** instâncias podem ser drenadas e substituídas (rolling deploys).

## 2. Por que isso importa

- É o ponto de entrada da maioria das aplicações em produção.
- Kubernetes Services, Ingress e LBs de cloud são balanceadores.
- Configurações erradas de health check, timeout ou afinidade causam indisponibilidade, erros 502/503 e comportamentos difíceis de reproduzir.

## 3. Como funciona

```mermaid
flowchart LR
    C1[Cliente] --> VIP["VIP / DNS do LB<br/>listener :443"]
    C2[Cliente] --> VIP
    VIP --> ALG{Algoritmo +<br/>estado de saúde}
    ALG --> B1[Backend 1 ✅]
    ALG --> B2[Backend 2 ✅]
    ALG -.x B3[Backend 3 ❌<br/>fora do pool]
    HC[Health checker] -. "GET /healthz a cada N s" .-> B1
    HC -.-> B2
    HC -.-> B3
```

Componentes:

| Componente | Descrição |
|---|---|
| **Listener / frontend** | IP (VIP) + porta + protocolo onde o LB recebe tráfego |
| **Regras** | (L7) Como escolher o grupo de backends: host, caminho, cabeçalho |
| **Target group / backend pool** | Conjunto de destinos e a porta deles |
| **Health check** | Teste periódico que define quem recebe tráfego |
| **Algoritmo** | Como escolher um backend saudável |

## 4. Conceitos centrais

### 4.1 L4 vs L7

| | **L4** | **L7** |
|---|---|---|
| Decide com base em | IP/porta (5‑tupla) | Conteúdo HTTP (host, caminho, cabeçalho, cookie) |
| Unidade balanceada | **Conexão** | **Requisição** |
| TLS | Repassa (passthrough) ou termina | Normalmente termina |
| Visibilidade do IP do cliente | Pode preservar o IP de origem (dependendo do modo) | Backend vê o IP do LB; IP original em `X-Forwarded-For` |
| Desempenho | Muito alto, baixa latência | Maior custo por requisição, mais recursos |
| Protocolos | Qualquer TCP/UDP | HTTP/HTTPS, gRPC, WebSocket |
| Exemplos | Network LB (AWS NLB, Azure LB), kube-proxy, IPVS, HAProxy tcp | Application LB (AWS ALB, Azure App Gateway), nginx, Envoy, Ingress |

> [!important] L4 e conexões longas
> Um LB L4 balanceia **conexões**, não requisições. Clientes com conexões persistentes (HTTP/2, gRPC, pools de banco) ficam "presos" a um backend. Com gRPC, isso leva a desbalanceamento severo: novas réplicas não recebem tráfego. Soluções: LB L7 que entenda HTTP/2, balanceamento no cliente, ou limitar a vida útil das conexões.

### 4.2 Algoritmos

| Algoritmo | Como escolhe | Bom para | Cuidado |
|---|---|---|---|
| **Round robin** | Em rodízio | Backends homogêneos, requisições parecidas | Ignora carga real |
| **Weighted round robin** | Rodízio proporcional a pesos | Backends de tamanhos diferentes, canary (ex.: 95/5) | — |
| **Least connections** | Backend com menos conexões ativas | Requisições de duração variável | Conexões ≠ carga em alguns casos |
| **Least response time / EWMA** | Menor latência observada | Heterogeneidade dinâmica | Implementação específica |
| **Random / power of two choices** | Sorteia dois e escolhe o menos carregado | Grande escala, LBs distribuídos | — |
| **Hash (IP de origem, cabeçalho, URL)** | Mesmo cliente/chave → mesmo backend | Afinidade, caches | Desbalanceia com poucos clientes (ex.: todos atrás de um NAT) |
| **Consistent hashing** | Hash com remapeamento mínimo quando o pool muda | Caches distribuídos | — |

### 4.3 Health checks

| Tipo | Verifica | Limitação |
|---|---|---|
| **TCP** | Handshake completa | Processo pode aceitar conexões e estar quebrado |
| **HTTP/HTTPS** | Status esperado (ex.: 200) em um caminho | Precisa de endpoint bem projetado |
| **gRPC** | Protocolo padrão `grpc.health.v1` | Requer suporte no serviço |

Parâmetros típicos: **intervalo**, **timeout**, **limiar de falhas** (unhealthy threshold) e **limiar de sucessos** (healthy threshold).

Boas práticas:

- O endpoint deve refletir a **capacidade de atender**, mas sem verificar dependências externas de forma pesada — senão uma falha no banco derruba **todas** as instâncias ao mesmo tempo (falha em cascata). Em Kubernetes, separe *liveness* (o processo está vivo?) de *readiness* (pode receber tráfego agora?).
- Deve ser **barato** e rápido.
- Health checks **ativos** (o LB testa) podem ser combinados com **passivos** (outlier detection: remover backends que estão retornando erros em tráfego real).

### 4.4 Sessões e afinidade (sticky sessions)

- **Sticky sessions** mandam o mesmo cliente sempre ao mesmo backend (via cookie do LB ou hash de IP).
- Necessárias quando a aplicação guarda estado de sessão **em memória local**.
- Desvantagens: distribuição desigual, perda de sessão quando a instância cai, dificulta escala.
- **Preferível:** aplicações **stateless**, com sessão em armazenamento compartilhado (Redis, banco) ou tokens.

### 4.5 Draining (connection draining / deregistration delay)

Ao remover um backend (deploy, scale‑in), o LB **para de enviar novas conexões** e aguarda as existentes terminarem até um tempo limite. Sem isso, deploys geram erros. Em Kubernetes, isso envolve o `terminationGracePeriodSeconds`, *preStop hooks* e a remoção do Pod dos endpoints antes do encerramento do processo.

### 4.6 Timeouts encadeados

```text
Cliente (30 s) → LB (idle 60 s) → Proxy/Ingress (read 60 s) → App (keep-alive 75 s) → Banco (query 25 s)
```

Regras práticas:

- O **idle timeout de keep‑alive do backend** deve ser **maior** que o do LB/proxy que o chama; caso contrário, o backend fecha conexões que o LB acha que estão vivas → **502 intermitentes**.
- Timeouts de requisição devem diminuir de fora para dentro, para que as camadas internas falhem primeiro e de forma controlada.

### 4.7 Alta disponibilidade do próprio LB

- **LBs de cloud** são gerenciados e distribuídos em várias zonas (você recebe um nome DNS, possivelmente com vários IPs).
- **On‑premises:** pares ativo/passivo com IP virtual (Keepalived/VRRP), ou ativo/ativo com ECMP/BGP anycast.
- **DNS round robin / GSLB:** vários registros A ou respostas por geolocalização/saúde. Simples, mas limitado pelos caches DNS (TTL) e sem visibilidade de saúde no cliente.

### 4.8 Modos de encaminhamento (L4)

| Modo | Como funciona | Consequência |
|---|---|---|
| **Proxy completo** | O LB termina a conexão e abre outra | Backend vê o IP do LB |
| **NAT (DNAT)** | Reescreve destino; resposta precisa voltar pelo LB | Backend pode ver o IP do cliente |
| **DSR (Direct Server Return)** | Backend responde diretamente ao cliente | Altíssimo desempenho de saída; configuração mais complexa |

## 5. Exemplos práticos

### Exemplo 1 — HAProxy L7 com health check

```text
frontend web
    bind *:80
    default_backend app

backend app
    balance roundrobin
    option httpchk GET /healthz
    http-check expect status 200
    server app1 10.0.2.11:8080 check inter 5s fall 3 rise 2
    server app2 10.0.2.12:8080 check inter 5s fall 3 rise 2
```

- `check inter 5s fall 3 rise 2` → testa a cada 5 s; 3 falhas seguidas tiram o servidor do pool; 2 sucessos o devolvem.

### Exemplo 2 — nginx com least connections e pesos

```nginx
upstream app {
    least_conn;
    server 10.0.2.11:8080 weight=3;
    server 10.0.2.12:8080 weight=1;
    server 10.0.2.13:8080 backup;      # só recebe tráfego se os outros estiverem indisponíveis
}
```

O nginx open source faz health checks **passivos** (`max_fails`, `fail_timeout`); checks ativos exigem módulos adicionais ou a versão comercial.

### Exemplo 3 — Canary por peso

Novo release recebe 5% do tráfego (peso 5 vs 95). Monitore taxa de erro e latência; aumente gradualmente ou reverta.

### Exemplo 4 — Kubernetes

- **Service ClusterIP**: balanceamento L4 interno (kube-proxy em iptables escolhe aleatoriamente; em IPVS suporta vários algoritmos; CNIs com eBPF têm implementação própria).
- **Service LoadBalancer**: provisiona um LB externo do provedor.
- **Ingress / Gateway API**: balanceamento L7.

Ver [[21 - Redes em Kubernetes]].

## 6. Comandos e demonstrações

```bash
# Ver qual backend respondeu (se a aplicação devolve um identificador, como hostname)
for i in $(seq 1 10); do curl -s http://lb.exemplo.com/whoami; echo; done

# Distribuição das respostas
for i in $(seq 1 100); do curl -s http://lb.exemplo.com/whoami; echo; done | sort | uniq -c

# Testar o health check como o LB faria, direto no backend
curl -s -o /dev/null -w '%{http_code} %{time_total}s\n' http://10.0.2.11:8080/healthz

# Ver os IPs por trás do nome de um LB de cloud (podem mudar ao longo do tempo)
dig +short meu-lb-123.elb.amazonaws.com

# HAProxy: estado dos servidores via socket de administração (se configurado)
echo "show servers state" | sudo socat stdio /run/haproxy/admin.sock

# Kubernetes: quais Pods estão atrás de um Service
kubectl get endpointslices -l kubernetes.io/service-name=meu-servico -o wide
```

## 7. Interpretação

- Distribuição muito desigual com round robin → como cada execução do `curl` no loop abre uma conexão nova, a reutilização de conexões não explica o desvio; verifique sticky sessions, hash por IP de origem ou pesos.
- Health check do backend demora perto do timeout configurado → instâncias vão "piscar" (entrar e sair do pool) sob carga.
- Endpoints vazios (`kubectl get endpointslices` sem endereços) → nenhum Pod pronto ou seletor errado → 503 no Ingress.

## 8. Problemas comuns

| Sintoma | Causa provável |
|---|---|
| 503 em todas as requisições | Todos os backends unhealthy: health check em caminho/porta errados, Security Group bloqueando o health checker, app retornando 3xx/401 no endpoint de saúde |
| 502 intermitentes | Keep‑alive do backend menor que o idle timeout do LB; backends sendo encerrados sem draining |
| Novas réplicas não recebem tráfego (gRPC/HTTP/2) | LB L4 com conexões longas |
| Usuários deslogados aleatoriamente | Sessão em memória sem sticky; ou sticky perdido após deploy |
| Um backend sobrecarregado | Hash por IP com muitos clientes atrás de um NAT; pesos errados |
| Erros durante deploys | Falta de draining, readiness probe ausente, encerramento abrupto |
| Backend vê sempre o IP do LB | Comportamento esperado em L7; use `X-Forwarded-For` |

## 9. Troubleshooting

1. **O LB está saudável?** Métricas/console: quantos targets *healthy*?
2. **Por que um target está unhealthy?** Execute o health check manualmente **a partir da rede do LB** (mesma porta, caminho, Host).
3. **O tráfego chega ao backend?** Logs de acesso do backend; `tcpdump` na porta.
4. **Quem gerou o erro?** Logs do LB distinguem erro do LB de erro retornado pelo backend.
5. **Timeouts** de cada camada coerentes?
6. **Durante deploys:** sequência de readiness → registro → draining → encerramento.

## 10. Aplicações em DevOps

- **Rolling, blue/green e canary deploys** dependem do LB.
- **Auto scaling** registra e remove instâncias do target group automaticamente.
- **Observabilidade:** métricas por target (latência, 5xx, conexões), logs de acesso do LB.
- **Infraestrutura como Código:** listeners, regras, target groups e health checks em Terraform; Services/Ingress em manifests.
- **Multi‑região:** DNS com roteamento por latência/saúde na frente de LBs regionais.

## 11. Considerações de segurança

- Exponha **apenas o LB**; backends em subnets privadas, aceitando tráfego **somente do LB** (SG de origem).
- Termine TLS com configuração moderna; considere re‑encryption até o backend em ambientes regulados.
- Use **WAF** e rate limiting na borda para mitigar abuso e DDoS de camada 7.
- Health check endpoints não devem expor informações sensíveis (versões, dependências, segredos).
- Cuidado com cabeçalhos de encaminhamento forjados ([[15 - Proxy e Reverse Proxy]]).

## 12. Resumo

- LB distribui tráfego, remove instâncias doentes e viabiliza deploys sem downtime.
- L4 balanceia conexões; L7 balanceia requisições e entende HTTP.
- Health checks bem projetados são essenciais; separe liveness de readiness.
- Prefira aplicações stateless a sticky sessions.
- Timeouts e keep‑alive devem ser coerentes entre camadas.

## 13. Verificação de conhecimento

1. Por que um serviço gRPC pode ficar desbalanceado atrás de um LB L4?
2. Um target group mostra todas as instâncias unhealthy, mas a aplicação responde `curl localhost:8080/healthz` com 200 nas instâncias. Cite três causas possíveis.
3. Qual a relação correta entre o keep‑alive timeout do backend e o idle timeout do LB?
4. Por que um health check que consulta o banco de dados pode causar uma indisponibilidade total?
5. Quando usar hashing por IP de origem é uma má ideia?

> [!success]- Respostas
> 1. gRPC usa conexões HTTP/2 longas e multiplexadas; o LB L4 só escolhe o backend no estabelecimento da conexão.
> 2. SG/firewall bloqueando o tráfego do health checker; app escutando só em 127.0.0.1; caminho/porta/protocolo configurados errados no health check; o endpoint retorna 301/401 para requisições sem o `Host` esperado.
> 3. O timeout do backend deve ser **maior** que o do LB, para que seja sempre o LB a fechar conexões ociosas.
> 4. Uma lentidão ou falha do banco faz todas as instâncias falharem no health check simultaneamente, e o LB remove todas do pool.
> 5. Quando muitos clientes compartilham o mesmo IP (NAT corporativo, CGNAT) ou há poucos clientes: a carga concentra-se em poucos backends.

## 14. Tópicos relacionados

- Anterior: [[15 - Proxy e Reverse Proxy]]
- Próxima: [[17 - VLAN e Segmentação de Redes]]
- [[21 - Redes em Kubernetes]]
- [[22 - Redes em Cloud e DevOps]]
- [[11 - HTTP e HTTPS]]
- [[Mapa de Estudos de Redes]]
