# Mevo Experts — Pesquisa + Painel Estratégico

Aplicação estática (Firebase Hosting + Cloud Firestore) para coletar respostas de prescritores (médicos e dentistas) sobre a Mevo Receita Digital e ler os resultados de forma estratégica.

| Rota | Página |
|---|---|
| `/mevoexpert` | Pesquisa — o prescritor seleciona a **sessão** (Mevo Expert é a padrão), responde e conclui com "Pronto — Obrigado pela colaboração". As respostas vão para o Firestore. |
| `/mevoexpert/resultado` | Painel estratégico — lê o Firestore, agrega por sessão, com botão **↻ Atualizar**, **+ Nova Sessão** e seções clicáveis com foco (fadeout do resto). |

## Dados (Firestore, projeto `mevoexpert`)

- **`sessoes`** — sessões de pesquisa. Seed automático na primeira carga: `Mevo Expert` (padrão) e `Mevo Teste` (para testes, não mistura com o resultado oficial). Novas sessões são criadas pelo botão do painel e aparecem no seletor da pesquisa.
- **`respostas`** — uma doc por resposta: `sessaoId`, `sessaoNome`, `respostas{...}`, `jornada{...}`, `criadaEm` (serverTimestamp). Regras permitem apenas criação (sem update/delete).

## Rodar local

```bash
npm install -g firebase-tools
firebase login
firebase serve          # http://localhost:5000/mevoexpert
```

## Deploy

1. No [console do Firebase](https://console.firebase.google.com/project/mevoexpert), crie o banco **Cloud Firestore** (modo produção, região `southamerica-east1`) — só na primeira vez.
2. Publique:

```bash
firebase deploy
```

URLs: `https://mevoexpert.web.app/mevoexpert` e `https://mevoexpert.web.app/mevoexpert/resultado`.

## Estrutura

```
├── firebase.json               # hosting + firestore
├── .firebaserc                 # projeto default: mevoexpert
├── firestore.rules             # regras de segurança
├── firestore.indexes.json
├── PLANO.md                    # plano detalhado da aplicação
└── public/mevoexpert/
    ├── index.html              # pesquisa
    └── resultado/index.html    # painel estratégico
```

> Nota: o painel tem leitura pública (sem autenticação). Restringir o `/resultado` com Firebase Auth fica registrado como evolução futura no PLANO.md.
