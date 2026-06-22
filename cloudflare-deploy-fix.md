# Cloudflare — Resolução do problema de publicação

> Documento técnico. Registra o diagnóstico e a correção do deploy da landing page
> `bariatric-seed-wl` na Cloudflare. Não é publicado (`*.md` está no `.assetsignore`).

---

## 1. Contexto do ambiente de deploy

- **Produto:** landing page estática (`index.html` + assets) servida como **static-assets Worker**.
- **Worker:** `bariatric-seed-wl` · zona `icvsgo.com` · domínio `https://bariatric-seed.icvsgo.com`.
- **CI:** **Cloudflare Workers Build** (build nativo da Cloudflare). **Não há GitHub Actions.**
  Todo push em `main` dispara automaticamente um build/deploy na Cloudflare.
- **Fluxo de publicação:**
  ```bash
  git push origin claude/brave-goodall-m0myau            # desenvolvimento
  git push origin claude/brave-goodall-m0myau:main       # publica (dispara o Workers Build)
  ```

---

## 2. Sintomas observados

Houve **dois bloqueios distintos** que impediam a página de ir ao ar:

### Sintoma A — falha de autenticação no deploy direto (sandbox)
Ao tentar publicar manualmente do ambiente de trabalho:
```bash
npx wrangler deploy
# → Authentication error [code: 10000]
```

### Sintoma B — build da Cloudflare falhando no painel
O Workers Build (acionado pelo push em `main`) parava com:
```
Your build token is no longer valid.
```
Ou seja, mesmo com o código correto no repositório, o pipeline de build não conseguia
autenticar para publicar a nova versão.

---

## 3. Investigação e causa raiz

### 3.1 Causa do Sintoma A — token do sandbox sem escopo de Workers
O `CLOUDFLARE_API_TOKEN` disponível no ambiente de trabalho (sandbox) possui apenas
permissões de **Zone Read** e **Settings Edit**. Ele **não** tem o escopo
**Account → Workers Scripts → Edit**, que é o exigido por `wrangler deploy`.

- **Diagnóstico:** o erro `10000` é o código padrão da API da Cloudflare para
  "autenticação/autorização insuficiente". Como o token tinha escopo de zona mas não de
  Workers, qualquer tentativa de `wrangler deploy`/`versions upload` era rejeitada.
- **Conclusão:** **não é corrigível pelo sandbox** — o deploy direto por aqui é
  estruturalmente impossível com o token disponível. A publicação **tem** que passar pelo
  Workers Build da Cloudflare (que usa o próprio build token da conta, não este token).

### 3.2 Pré-requisito de configuração — `wrangler.jsonc` (entry point)
Para o Workers Build conseguir publicar um site **sem código de servidor**, o wrangler
precisa de um arquivo de configuração que aponte para o diretório de assets. Sem ele, o
build falha com:
```
Missing entry-point to Worker script or to assets directory
```
Foi garantido que o repositório contém o `wrangler.jsonc` correto (Worker apenas de
assets estáticos):
```jsonc
{
  "name": "bariatric-seed-wl",
  "compatibility_date": "2026-06-16",
  "assets": { "directory": "." }
}
```
E o `.assetsignore` para não publicar arquivos internos como assets públicos:
```
wrangler.jsonc
.gitignore
*.md
```

### 3.3 Causa do Sintoma B — build token expirado/revogado (lado da conta)
O Workers Build autentica com um **build token** próprio, gerenciado no painel da
Cloudflare (independente do token do sandbox). Esse token **expirou/foi invalidado**, daí
a mensagem "Your build token is no longer valid". Isso é um estado do **lado da conta**,
não algo que o código ou o sandbox possam corrigir.

---

## 4. Ações tomadas

| # | Ação | Onde | Resultado |
|---|------|------|-----------|
| 1 | Tentativa de `npx wrangler deploy` para publicar direto | sandbox | Falhou (`error 10000`) — confirmou que o token do sandbox não tem escopo de Workers. |
| 2 | Confirmado/validado `wrangler.jsonc` (assets dir `.`) e `.assetsignore` no repo | repo | Pré-requisito de build atendido (entry point presente). |
| 3 | Diagnóstico de que a publicação **deve** ocorrer via Workers Build (push→`main`), nunca via token do sandbox | — | Estratégia de deploy definida. |
| 4 | **Orientação ao cliente** para renovar o build token no painel: **Workers & Pages → `bariatric-seed-wl` → Settings → Builds → API token** → renovar → **Retry** | painel Cloudflare (lado do cliente) | Token de build renovado. |
| 5 | **Re-disparo do build** via commit vazio em `main` para forçar novo Workers Build com o token renovado | repo | Commit `09dfb37` "redeploy: re-trigger Workers Build after build-token renewal". |

### Comando usado para re-disparar o build
```bash
# commit vazio só para acionar o Workers Build com o token já renovado
git commit --allow-empty -m "redeploy: re-trigger Workers Build after build-token renewal"
git push origin claude/brave-goodall-m0myau:main
```

---

## 5. Causa raiz (resumo)

A publicação dependia de **duas autenticações diferentes**, e ambas precisavam ser tratadas:

1. **Deploy direto pelo sandbox era inviável** — o `CLOUDFLARE_API_TOKEN` do ambiente só
   tem escopo de zona (Zone Read / Settings Edit), sem **Workers Scripts → Edit**. Por
   isso `wrangler deploy` retorna `error 10000`. **Solução:** não publicar pelo sandbox;
   usar o pipeline nativo (push em `main` → Workers Build).
2. **O pipeline nativo estava parado** porque o **build token da conta expirou**
   ("build token no longer valid"). **Solução (lado do cliente):** renovar o build token
   no painel e dar Retry; em seguida re-disparei o build com um push em `main`.

Em ambos os casos a **causa raiz é de credenciais/escopo de autenticação**, não de código.
O conteúdo do repositório (HTML/assets + `wrangler.jsonc` + `.assetsignore`) já estava correto.

---

## 6. Como publicar daqui pra frente

```bash
# 1. desenvolver e validar na branch de trabalho
git push origin claude/brave-goodall-m0myau

# 2. publicar: fast-forward para main (dispara o Workers Build)
git push origin claude/brave-goodall-m0myau:main
```

- **Não** tente `wrangler deploy` pelo sandbox — o token não tem escopo de Workers.
- Se o build voltar a falhar com **"build token no longer valid"**: renovar em
  **Workers & Pages → bariatric-seed-wl → Settings → Builds → API token** e dar **Retry**
  (ou um novo push/commit vazio em `main`).
- Se o build falhar com **"Missing entry-point..."**: confirmar que `wrangler.jsonc`
  (com `assets.directory`) está presente na raiz do repo.

---

## 7. Verificação (pendente, do lado do cliente)

O sandbox **não consegue acessar `bariatric-seed.icvsgo.com`** (egress bloqueado —
`host_not_allowed`), então a confirmação final é visual/do lado do cliente:

- [ ] Workers Build mais recente com status **Success** no painel.
- [ ] `https://bariatric-seed.icvsgo.com` servindo a versão atual (conferir headline/VSL).
- [ ] Cache do HTML honrando `_headers` (`max-age=0, must-revalidate`) — alterações
      aparecem sem precisar limpar cache.
