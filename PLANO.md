# Plano — Aplicação Pesquisa Mevo Experts + Painel Estratégico

**Data:** 11/07/2026
**Base:** `pesquisa-mevo-experts-whatsapp_1.html` e `mevo-experts-painel-estrategico_1.html` (Downloads)
**Projeto Firebase:** `mevoexpert`

---

## 1. Visão geral

Duas páginas hospedadas no Firebase Hosting, com dados no Cloud Firestore:

| Rota | Página | Função |
|---|---|---|
| `/mevoexpert` | Pesquisa | Múltiplos prescritores (médicos e dentistas) respondem; dados salvos no Firestore por **sessão** |
| `/mevoexpert/resultado` | Painel Estratégico | Lê o Firestore, agrega e exibe os resultados, com filtro por sessão |

Sem backend próprio: apenas HTML estático + Firebase JS SDK (CDN, sem build) + Firestore.

```
Prescritor ──▶ /mevoexpert ──▶ Firestore (respostas)
                                    ▲
Organizador ─▶ /mevoexpert/resultado┘ (leitura + refresh + nova sessão)
```

---

## 2. Estrutura do projeto

```
Mevo Experts/
├── firebase.json              # config hosting (public, rewrites, cleanUrls)
├── .firebaserc                # projeto default: mevoexpert
├── firestore.rules            # regras de segurança
├── firestore.indexes.json     # índices (sessaoId + criadaEm)
└── public/
    └── mevoexpert/
        ├── index.html         # a pesquisa
        └── resultado/
            └── index.html     # o painel estratégico
```

URLs finais: `https://mevoexpert.web.app/mevoexpert` e `https://mevoexpert.web.app/mevoexpert/resultado`.

---

## 3. Modelo de dados (Firestore)

### Coleção `sessoes`
Uma doc por sessão de pesquisa:

```js
sessoes/{autoId} = {
  nome: "Mevo Expert",        // exibido no seletor
  slug: "mevo-expert",        // normalizado, evita duplicata
  criadaEm: serverTimestamp(),
  padrao: true                // true só na "Mevo Expert"
}
```

**Seed inicial (garantido pelo próprio app na primeira carga):**
- `Mevo Expert` — sessão padrão, pré-selecionada na pesquisa
- `Mevo Teste` — sessão para testes, para não misturar respostas no dash

### Coleção `respostas`
Uma doc por resposta enviada:

```js
respostas/{autoId} = {
  sessaoId: "<id da doc em sessoes>",
  sessaoNome: "Mevo Expert",           // denormalizado p/ leitura simples
  pesquisa: "Mevo Experts — Receita Digital",
  criadaEm: serverTimestamp(),
  respostas: { nome, especialidade, local_atendimento, nps, palavra,
               prioridade, renovar, renovar_obs, shop_dif, shop_ind, shop_obs,
               ia_familiar, ia_etapas[], ia_desejo, ia_receio,
               falta, livre },
  jornada: { acesso: {etapa, satisfacao, gosta, trava}, paciente: {...},
             medicamento: {...}, exames: {...}, documentos: {...},
             assinatura: {...}, envio: {...}, uso: {...} }
  // etapas da jornada: Acesso e login · Cadastro de paciente ·
  // Busca de medicamento e posologia · Prescrição de exames ·
  // Prescrição de atestado, encaminhamento e relatórios ·
  // Assinatura digital · Envio e entrega ao paciente ·
  // Uso da prescrição pelo paciente
}
```

### Regras de segurança (`firestore.rules`)
Pesquisa é pública (sem login), então:

- `sessoes`: **read** liberado; **create** liberado com validação (só campos `nome/slug/criadaEm/padrao`, nome 2–60 chars); **update/delete** negados.
- `respostas`: **create** liberado com validação de shape (campos esperados, tamanhos máximos de string); **read** liberado (o painel é client-side); **update/delete** negados.

> Risco assumido: leitura pública das respostas (painel sem auth). Mitigação futura: Firebase Auth (login Google restrito a @mevo.com.br) para a rota de resultado — fica registrado como fase 2, fora do escopo atual.

---

## 4. Página da Pesquisa (`/mevoexpert`)

Partir do `pesquisa-mevo-experts-whatsapp_1.html`, mantendo visual e stepper. Mudanças:

### 4.1 Remover
- Botão/link **"Enviar pelo WhatsApp"**, ícone, `WHATSAPP_DESTINO`, `montarTexto()`, `wa.me`
- Botão **"Copiar respostas (JSON)"** e o `<details>` de preview do JSON
- `DB_ENDPOINT` do Google Apps Script e `SECRET` (credencial hardcoded sai do código)
- Texto do hero: "Ao final, você envia num toque pelo WhatsApp" → substituir por "Ao final, é só concluir — suas respostas são registradas automaticamente"

### 4.2 Adicionar — seletor de sessão (tela inicial)
- Novo card no step 0 (acima de nome/especialidade): **"Sessão do encontro"**
- Carrega as sessões de `sessoes` (Firestore), ordenadas por `criadaEm`
- **"Mevo Expert" pré-selecionada** (a que tem `padrao: true`); "Mevo Teste" e sessões criadas no painel aparecem como opções
- Estilo: mesmos `.choice` já existentes no arquivo (radio pills)
- Fallback offline/erro: mostra "Mevo Expert" e "Mevo Teste" hardcoded

### 4.3 Alterar — fluxo de conclusão
- Último step vira tela de agradecimento:
  - Anel ✓ (já existe) + **"Pronto!"** + **"Obrigado pela colaboração."**
  - Sem nenhum botão de envio/cópia
- Ao clicar **"Finalizar"** no penúltimo step:
  1. `addDoc(respostas, payload)` com `serverTimestamp()`
  2. Enquanto grava: botão em estado "Enviando…" (desabilitado)
  3. Sucesso → avança para a tela "Pronto — Obrigado pela colaboração"
  4. Erro → mantém no step, exibe aviso discreto "Não foi possível salvar, verifique a conexão e tente de novo" (sem perder respostas)
- Guarda anti-duplicata: flag `_salvou` (já existe no código) impede dois envios

### 4.4 Integração Firebase
SDK modular v10 via CDN em `<script type="module">`:

```js
import { initializeApp } from "https://www.gstatic.com/firebasejs/10.12.0/firebase-app.js";
import { getFirestore, collection, addDoc, getDocs, query,
         orderBy, serverTimestamp } from ".../firebase-firestore.js";
```

Config do projeto `mevoexpert` (a fornecida). Analytics opcional (mantém `measurementId`).

---

## 5. Painel Estratégico (`/mevoexpert/resultado`)

Partir do `mevo-experts-painel-estrategico_1.html`, mantendo os gráficos SVG. Mudanças:

### 5.1 Fonte de dados
- Remover `DATA_ENDPOINT`/JSONP do Apps Script
- `getDocs(query(respostas, orderBy("criadaEm")))` no boot
- Adapter: converte o shape do Firestore (`jornada[etapa].satisfacao`) para o shape interno do painel (`jornada[etapa].sat`), normaliza `renovar/shop_dif/shop_ind` para as chaves usadas nos agregados
- `sample()` (dados de demonstração) permanece **apenas** como fallback quando a sessão tem 0 respostas, com aviso visível "exibindo dados de demonstração"

### 5.2 Filtro por sessão
- Nova linha de pills no header: **"Sessão: [Mevo Expert] [Mevo Teste] [ ... ]"**, carregada de `sessoes`
- Default: **Mevo Expert** — respostas da "Mevo Teste" nunca se misturam
- O filtro por especialidade (já existente) continua funcionando dentro da sessão

### 5.3 Botão Refresh
- Botão "↻ Atualizar" no header
- Refaz o `getDocs` do Firestore e re-renderiza tudo (`renderAll`)
- Atualiza o "atualizado em" com data + hora; estado girando enquanto busca

### 5.4 Botão "Nova Sessão"
- Botão "+ Nova Sessão" no header
- Abre modal simples: input do nome + [Salvar] [Cancelar]
- Validação: nome 2–60 chars, não duplicado (checa `slug`)
- Salva em `sessoes` → pill nova aparece imediatamente no painel **e** no seletor da pesquisa (próxima carga)

### 5.5 Seções clicáveis com foco (fadeout)
- Cada `<section>` (01 Mapa de oportunidade … 07 Backlog + KPIs) vira clicável
- Ao clicar: a seção ganha classe `.focada` e as demais `.apagada`
  - `.apagada { opacity:.18; filter:blur(1px); transition: opacity .4s }`
  - `.focada { opacity:1; transform:scale(1.01) }` + scroll suave até ela
- Clicar de novo na seção focada (ou tecla `Esc`, ou botão "ver tudo") volta ao estado normal
- Barra de navegação fixa com atalhos 01–07 para pular entre seções

---

## 6. Deploy

1. `npm install -g firebase-tools` (já indicado)
2. `firebase login`
3. **Console Firebase (ação manual, 1x):** criar o banco Cloud Firestore no projeto `mevoexpert` (modo produção, região `southamerica-east1`)
4. `firebase deploy` → publica hosting + rules + indexes

`firebase.json`:
```json
{
  "hosting": { "public": "public", "cleanUrls": true,
               "ignore": ["firebase.json", "**/.*"] },
  "firestore": { "rules": "firestore.rules", "indexes": "firestore.indexes.json" }
}
```

---

## 7. Ordem de execução

| # | Etapa | Entregável |
|---|---|---|
| 1 | Estrutura do projeto + configs Firebase | `firebase.json`, `.firebaserc`, `firestore.rules`, `firestore.indexes.json` |
| 2 | Pesquisa adaptada | `public/mevoexpert/index.html` (sessões + Firestore + tela "Obrigado") |
| 3 | Painel adaptado | `public/mevoexpert/resultado/index.html` (Firestore + refresh + nova sessão + foco por seção) |
| 4 | Seed das sessões | Auto-seed no primeiro load ("Mevo Expert" + "Mevo Teste") |
| 5 | Teste local | `firebase emulators:start` ou `firebase serve` — resposta ponta a ponta nas duas sessões |
| 6 | Deploy | `firebase deploy` + smoke test nas duas URLs |

---

## 8. Fora de escopo (registrado para depois)

- Autenticação no painel de resultados (hoje leitura pública)
- Exportação CSV/planilha a partir do painel
- Edição/arquivamento de sessões
- Análise de texto livre com IA (temas de "o que falta", desejos/receios de IA)
