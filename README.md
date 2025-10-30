# Load Balancing com Caddy

## 🎯 Estratégias de Load Balancing Implementadas

### 1. **Round-Robin (Padrão)** - `/balanced`
- Distribui requisições alternadamente entre os servidores
- Uso: Quando todos os servidores têm capacidade similar
```
Req 1 → Servidor 1
Req 2 → Servidor 2
Req 3 → Servidor 1
Req 4 → Servidor 2
```

### 2. **Random** - `/balanced-random`
- Escolhe um servidor aleatoriamente
- Uso: Distribuição simples sem necessidade de ordem

### 3. **Least Connections** - `/balanced-least`
- Encaminha para o servidor com menos conexões ativas
- Uso: Quando requisições têm duração variável
- Ideal para: WebSockets, long-polling, streaming

### 4. **IP Hash (Sticky Sessions)** - `/balanced-ip`
- Mesmo IP sempre vai para o mesmo servidor
- Uso: Quando precisa manter sessão do usuário
- Ideal para: Aplicações com sessão local (sem Redis/Memcached)

## 🏥 Health Checks

Configurado para verificar a saúde dos backends:

```caddyfile
health_uri /          # Endpoint para verificar
health_interval 10s   # Verifica a cada 10 segundos
health_timeout 5s     # Timeout de 5 segundos
health_status 200     # Espera status 200 (opcional)
```

**Comportamento:**
- ✅ Backend saudável → Recebe requisições
- ❌ Backend com falha → Removido automaticamente da rotação
- ♻️ Backend recuperado → Reintegrado automaticamente

## 🧪 Como Testar

### Testar Round-Robin
```bash
for i in {1..6}; do 
  curl -s http://localhost/balanced | grep -o "Servidor Python [12]"
done
```

### Testar IP Hash (deve sempre ir pro mesmo)
```bash
for i in {1..4}; do 
  curl -s http://localhost/balanced-ip | grep -o "Servidor Python [12]"
done
```

### Testar Health Check
```bash
# Parar um servidor
docker compose stop python-server-1

# Aguardar 10s para o health check detectar
sleep 11

# Testar (deve usar apenas servidor 2)
curl http://localhost/balanced

# Reativar servidor
docker compose start python-server-1
```

## 📊 Configurações Avançadas

### Pesos (Weight) - Quando servidores têm capacidades diferentes
```caddyfile
reverse_proxy {
    to python-server-1:8000 2  # Recebe 2x mais requisições
    to python-server-2:8000 1  # Recebe 1x
    lb_policy weighted_round_robin
}
```

### Retries - Tentar novamente em caso de falha
```caddyfile
reverse_proxy python-server-1:8000 python-server-2:8000 {
    lb_try_duration 2s
    lb_try_interval 250ms
}
```

### Max Fails - Remover após N falhas consecutivas
```caddyfile
reverse_proxy python-server-1:8000 python-server-2:8000 {
    fail_duration 30s
    max_fails 3
    unhealthy_status 5xx
}
```

## 🔍 Monitoramento

Acessar admin API do Caddy:
```bash
curl http://localhost:2019/config/ | jq
```

Ver upstreams:
```bash
curl http://localhost:2019/reverse_proxy/upstreams | jq
```

## 🎓 Quando Usar Cada Estratégia

| Estratégia | Caso de Uso | Vantagem |
|------------|-------------|----------|
| Round-Robin | Servidores idênticos, requisições rápidas | Simples e eficiente |
| Least Conn | Requisições de duração variável | Melhor distribuição de carga |
| IP Hash | Aplicações com sessão local | Mantém usuário no mesmo servidor |
| Random | Quando não há requisitos específicos | Implementação leve |
| Weighted | Servidores com capacidades diferentes | Aproveita melhor os recursos |

## 🚀 Próximos Passos

1. **Adicionar mais backends** → Apenas adicionar mais containers no docker-compose
2. **Usar aplicações reais** → Substituir os servidores Python
3. **Monitorar com métricas** → Integrar Prometheus/Grafana
4. **HTTPS automático** → Configurar domínio real
