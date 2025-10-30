# Como Confiar no Certificado CA do Caddy

Este guia mostra como instalar o certificado root CA do Caddy para evitar avisos de segurança no navegador e não precisar usar `-k` no curl.

## 📋 O que foi feito

O certificado root CA do Caddy já foi extraído para este diretório:
- **Arquivo**: `caddy-root-ca.crt`
- **Emissor**: Caddy Local Authority - 2025 ECC Root
- **Validade**: até 08/09/2035

## 🐧 1. Instalar no WSL/Linux (para curl e ferramentas CLI)

### Ubuntu/Debian:
```bash
# Copiar o certificado para o diretório de CAs confiáveis
sudo cp caddy-root-ca.crt /usr/local/share/ca-certificates/caddy-local-ca.crt

# Atualizar o store de certificados
sudo update-ca-certificates

# Verificar se foi adicionado (deve mostrar "1 added")
```

### Testar no WSL:
```bash
# Agora deve funcionar SEM -k
curl -I https://backend.local/

# Se ainda der erro, pode ser necessário especificar o CA manualmente:
curl --cacert caddy-root-ca.crt -I https://backend.local/
```

## 🪟 2. Instalar no Windows (para navegadores Chrome, Edge, etc.)

### Método 1: Via Interface Gráfica (Recomendado)

1. **Abrir o gerenciador de certificados**:
   - Pressione `Win + R`
   - Digite: `certmgr.msc`
   - Pressione Enter

2. **Importar o certificado**:
   - No painel esquerdo, expanda **"Autoridades de Certificação Raiz Confiáveis"**
   - Clique com botão direito em **"Certificados"**
   - Selecione **"Todas as Tarefas" → "Importar..."**
   - Clique em **"Avançar"**
   - Clique em **"Procurar..."** e navegue até:
     ```
     C:\Users\Public\caddy-root-ca.crt
     ```
     (Você já copiou o arquivo para lá rodando: `cp caddy-root-ca.crt /mnt/c/Users/Public/caddy-root-ca.crt` no WSL)
   - Selecione o arquivo e clique em **"Abrir"**
   - Clique em **"Avançar"**
   - Certifique-se de que está selecionado **"Autoridades de Certificação Raiz Confiáveis"**
   - Clique em **"Avançar"** e depois em **"Concluir"**
   - Confirme o aviso de segurança clicando em **"Sim"**

### Método 2: Via PowerShell (Administrador) - **RECOMENDADO**

**Passo 1**: No WSL, copie o certificado para uma pasta acessível do Windows:
```bash
cp caddy-root-ca.crt /mnt/c/Users/Public/caddy-root-ca.crt
```

**Passo 2**: No PowerShell como Administrador:
```powershell
# Importar o certificado
Import-Certificate -FilePath "C:\Users\Public\caddy-root-ca.crt" -CertStoreLocation Cert:\LocalMachine\Root

# Verificar instalação
Get-ChildItem -Path Cert:\LocalMachine\Root | Where-Object {$_.Subject -like "*Caddy*"}
```

**Alternativa**: Se o caminho WSL funcionar no seu sistema:
```powershell
$certPath = "\\wsl.localhost\Ubuntu\home\victorcostac\Documentos\curso-si\SRI\reverse-proxy-caddy\caddy-root-ca.crt"
Import-Certificate -FilePath $certPath -CertStoreLocation Cert:\LocalMachine\Root
```

### Método 3: Via linha de comando (certutil)

**Passo 1**: Certifique-se que o certificado está em `C:\Users\Public\caddy-root-ca.crt` (rode o comando no WSL):
```bash
cp caddy-root-ca.crt /mnt/c/Users/Public/caddy-root-ca.crt
```

**Passo 2**: No CMD como Administrador:
```cmd
certutil -addstore -f "ROOT" "C:\Users\Public\caddy-root-ca.crt"
```

## 🦊 3. Firefox (requer importação separada)

O Firefox usa seu próprio store de certificados:

1. Abra o Firefox
2. Vá em **Menu (☰) → Configurações → Privacidade e Segurança**
3. Role até **Certificados** e clique em **"Ver certificados..."**
4. Vá na aba **"Autoridades"**
5. Clique em **"Importar..."**
6. Selecione o arquivo `caddy-root-ca.crt`
7. Marque **"Confiar nesta CA para identificar sites"**
8. Clique em **"OK"**

## ✅ 4. Testar a instalação

### No WSL (curl):
```bash
# Deve funcionar sem -k e sem erros
curl -I https://backend.local/
curl -L https://backend.local/app1
curl -L https://backend.local/balanced
```

### No Windows (navegador):
1. Abra o Chrome/Edge
2. Acesse: `https://backend.local/`
3. **Não deve aparecer aviso de segurança**
4. O cadeado deve aparecer como seguro 🔒

## 🔄 5. Quando renovar o certificado?

O certificado CA do Caddy é válido até **2035**, então não precisa renovar frequentemente. Mas se você:
- Recriar o volume Docker (`docker compose down -v`)
- Trocar de máquina/reinstalar
- O certificado expirar

Você precisará repetir este processo.

## 🗑️ 6. Remover o certificado (se necessário)

### No WSL/Linux:
```bash
sudo rm /usr/local/share/ca-certificates/caddy-local-ca.crt
sudo update-ca-certificates --fresh
```

### No Windows:
1. Abra `certmgr.msc`
2. Vá em **"Autoridades de Certificação Raiz Confiáveis" → "Certificados"**
3. Encontre **"Caddy Local Authority"**
4. Clique com botão direito e selecione **"Excluir"**

### No Firefox:
1. **Menu → Configurações → Privacidade e Segurança → Ver certificados**
2. Aba **"Autoridades"**
3. Procure **"Caddy Local Authority"**
4. Selecione e clique em **"Excluir ou Desconfiar"**

## 📝 Notas Importantes

- ⚠️ **Apenas para desenvolvimento**: Este certificado é para ambiente de desenvolvimento local
- 🔒 **Não compartilhe**: O arquivo `.crt` dá acesso a criar certificados confiáveis
- 🐳 **Docker**: O certificado é específico desta instalação do Caddy
- 🌐 **Rede**: Só funciona para sites acessados como `backend.local` (definido no `/etc/hosts`)

## 🆘 Troubleshooting

### PowerShell não encontra o arquivo no caminho WSL (`\\wsl$\...`)
**Solução**: Copie o certificado para uma pasta Windows acessível:
```bash
# No WSL
cp caddy-root-ca.crt /mnt/c/Users/Public/caddy-root-ca.crt
```
Depois use o caminho Windows no PowerShell:
```powershell
Import-Certificate -FilePath "C:\Users\Public\caddy-root-ca.crt" -CertStoreLocation Cert:\LocalMachine\Root
```

### Curl ainda reclama de certificado inválido no WSL
```bash
# Força recarregar os certificados
sudo update-ca-certificates --fresh

# Teste especificando o CA manualmente
curl --cacert caddy-root-ca.crt -I https://backend.local/
```

### Navegador ainda mostra aviso (Windows)
1. Feche TODOS os navegadores completamente
2. Reinicie o navegador
3. Limpe o cache SSL do Chrome: `chrome://net-internals/#sockets` → "Flush socket pools"
4. Tente novamente

### Certificado não aparece no certmgr.msc
- Certifique-se de que importou para **"LocalMachine\Root"** (não "CurrentUser")
- Tente usar o PowerShell como Administrador

### Firefox continua reclamando
- O Firefox tem store próprio, precisa importar manualmente (veja seção 3)
- Após importar, reinicie o Firefox

---

## 🎉 Pronto!

Agora você pode acessar `https://backend.local` sem avisos e usar curl sem `-k`!

**Comandos úteis para testar**:
```bash
# Curl simples
curl https://backend.local/

# Ver todas as rotas
curl https://backend.local/app1
curl https://backend.local/app2
curl https://backend.local/balanced

# Ver headers
curl -I https://backend.local/
```
