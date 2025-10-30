# 🔧 Problema: Páginas em Branco ao Navegar (/ator, /diretor, /classe)

## 🔍 O Problema

Quando você acessava `https://cine.locadora.local/`:
- ✅ Página inicial carregava com CSS perfeitamente
- ❌ Ao clicar em "Ator", "Diretor" ou "Classe" → **Página em branco**

## 🤔 Por que isso acontecia?

### Configuração problemática:

```caddyfile
handle_path / {
    reverse_proxy frontend1:3000 frontend2:3000
}
```

### O problema com `handle_path /`:

O `handle_path /` corresponde a **QUALQUER path** que comece com `/` (ou seja, TODOS os paths):

```
✅ /           → match!
✅ /ator       → match! (começa com /)
✅ /diretor    → match! (começa com /)
✅ /classe     → match! (começa com /)
✅ /qualquer   → match! (começa com /)
```

### O que acontecia passo a passo:

1. **Cliente acessa**: `https://cine.locadora.local/ator`
2. **Caddy vê**: `handle_path /` → match! ✅
3. **Caddy remove** o prefixo `/`: sobra `ator` (sem barra!)
4. **Caddy envia** para backend: `http://frontend1:3000/ator`... mas espera! Remove o `/` então envia apenas `ator`
5. **Next.js recebe**: path inválido `ator` (sem `/`)
6. **Next.js responde**: vazio/404
7. **Navegador mostra**: página em branco ❌

**Fluxo do problema:**
```
Cliente              Caddy                Backend
  |                    |                     |
  |-- GET /ator ------>|                     |
  |                    | handle_path / match |
  |                    | Remove "/" = "ator" |
  |                    |-- GET ator -------->| ❌ Path inválido!
  |                    |<-- (vazio) ---------|
  |<-- Página branca --|                     |
```

## ✅ A Solução

Mudamos `handle_path /` para `handle /*` **no final** da configuração:

```caddyfile
cine.locadora.local {
    # 1º: Recursos estáticos (específicos)
    handle /_next/* {
        reverse_proxy frontend1:3000 frontend2:3000
    }
    
    handle /favicon.ico {
        reverse_proxy frontend1:3000 frontend2:3000
    }
    
    # 2º: Rotas específicas para load balancing demo
    handle_path /app1* {
        reverse_proxy frontend1:3000
    }
    
    handle_path /app2* {
        reverse_proxy frontend2:3000
    }
    
    handle_path /balanced-random* {
        reverse_proxy frontend1:3000 frontend2:3000 {
            lb_policy random
        }
    }
    
    # ... outras rotas específicas ...
    
    # 3º: Rota catch-all (última!)
    handle /* {
        reverse_proxy frontend1:3000 frontend2:3000 {
            health_uri /
            health_interval 10s
            health_timeout 5s
            health_status 200
        }
    }
}
```

### Por que funciona agora:

**`handle /*` vs `handle_path /`:**

| Diretiva | O que faz | Exemplo |
|----------|-----------|---------|
| `handle_path /` | Remove o prefixo `/` antes de enviar | `/ator` → backend recebe `ator` ❌ |
| `handle /*` | **Mantém o path completo** | `/ator` → backend recebe `/ator` ✅ |

**Fluxo corrigido:**
```
Cliente              Caddy                Backend
  |                    |                     |
  |-- GET /ator ------>|                     |
  |                    | handle /* match     |
  |                    | Mantém path completo|
  |                    |-- GET /ator ------->| ✅ Path válido!
  |                    |<-- HTML + CSS ------|
  |<-- Página OK ------|                     |
```

## 📐 Ordem dos Handles é CRÍTICA!

O Caddy processa handles **na ordem que aparecem**. O primeiro que fizer match é usado:

### ✅ CORRETO (específico → genérico):
```caddyfile
handle /_next/* {          # 1º - Mais específico
    ...
}
handle /app1* {            # 2º - Específico
    ...
}
handle /* {                # 3º - Catch-all (último!)
    ...
}
```

### ❌ ERRADO (genérico primeiro):
```caddyfile
handle /* {                # 1º - Pega TUDO!
    ...
}
handle /_next/* {          # 2º - Nunca será alcançado!
    ...
}
handle /app1* {            # 3º - Nunca será alcançado!
    ...
}
```

## 🎯 Quando usar cada diretiva:

### `handle` (mantém path):
```caddyfile
# Use quando o backend PRECISA receber o path original
handle /* {
    reverse_proxy app:3000
}
# /ator → backend recebe /ator ✅
```

### `handle_path` (remove prefixo):
```caddyfile
# Use quando o backend NÃO espera o prefixo
handle_path /api* {
    reverse_proxy api-server:8000
}
# /api/users → backend recebe /users ✅
```

### Exemplo prático:

Imagine que você tem:
- Frontend Next.js em `frontend:3000` que espera `/ator`, `/diretor`
- API em `api:8000` que espera `/users`, `/posts` (sem prefixo `/api`)

Configuração ideal:
```caddyfile
app.com {
    # API: remove /api do path
    handle_path /api/* {
        reverse_proxy api:8000
    }
    
    # Frontend: mantém path original
    handle /* {
        reverse_proxy frontend:3000
    }
}
```

Resultado:
- `https://app.com/api/users` → `http://api:8000/users` ✅
- `https://app.com/ator` → `http://frontend:3000/ator` ✅

## 🧪 Como testar:

### 1. Via curl:
```bash
# Teste a página inicial
curl -I https://cine.locadora.local/

# Teste rotas internas
curl -I https://cine.locadora.local/ator
curl -I https://cine.locadora.local/diretor
curl -I https://cine.locadora.local/classe

# Todas devem retornar HTTP/2 200
```

### 2. No navegador:
1. Acesse `https://cine.locadora.local/`
2. Clique em "Ator" → deve carregar a página com CSS ✅
3. Clique em "Diretor" → deve carregar a página com CSS ✅
4. Clique em "Classe" → deve carregar a página com CSS ✅

### 3. DevTools (F12):
1. Abra DevTools → Network
2. Navegue para `/ator`
3. Veja que:
   - Request `/ator` → Status 200 ✅
   - Request `/_next/static/css/...` → Status 200 ✅
   - Todos os recursos carregam ✅

## 📋 Resumo dos 2 Problemas Resolvidos:

### Problema 1: CSS não carregava
**Causa**: Faltavam handles para `/_next/*` e `/favicon.ico`  
**Solução**: Adicionar handles específicos para recursos estáticos

### Problema 2: Páginas em branco ao navegar
**Causa**: `handle_path /` estava capturando todos os paths e removendo o `/`  
**Solução**: Trocar para `handle /*` que mantém o path completo

## 🎉 Resultado Final:

Agora sua configuração do Caddy:
- ✅ Serve a página inicial perfeitamente
- ✅ CSS e JavaScript carregam de todas as páginas
- ✅ Navegação entre páginas (/ator, /diretor, /classe) funciona
- ✅ Load balancing continua funcionando nas rotas específicas (/app1, /app2, etc.)
- ✅ HTTPS funciona sem avisos (com certificado instalado)

---

## 📚 Conceitos Importantes Aprendidos:

1. **Ordem importa**: Handles específicos antes de genéricos
2. **`handle` vs `handle_path`**: Um mantém o path, outro remove
3. **Recursos estáticos**: Sempre precisam de handles específicos
4. **Catch-all**: `handle /*` deve ser o ÚLTIMO

## 🔗 Documentação Oficial:

- [Caddy `handle` directive](https://caddyserver.com/docs/caddyfile/directives/handle)
- [Caddy `handle_path` directive](https://caddyserver.com/docs/caddyfile/directives/handle_path)
- [Caddy `reverse_proxy` directive](https://caddyserver.com/docs/caddyfile/directives/reverse_proxy)
