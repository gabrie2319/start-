# SADMED Prospecção + CRM — Plano (Etapa 1)

> Status: **aguardando aprovação**. Nenhum código de aplicação foi escrito ainda.

## 1. Fontes de dados pesquisadas

### a) Dados de CNPJ (Receita Federal) — filtro por CNAE, município e data de abertura

| Opção | Filtra CNAE + município + data de abertura? | Telefone/e-mail | Custo aproximado | Observações |
|---|---|---|---|---|
| **Casa dos Dados – API v5** (`POST /v5/cnpj/pesquisa`) ✅ *recomendada* | Sim (CNAE, UF/município, `data_abertura` com intervalo ou `ultimos_dias`, situação cadastral, `mais_filtros.com_telefone` / `com_email`, só matriz) | Sim (dados declarados à Receita) | R$ 0,01 por CNPJ consultado; plano mensal de 5.000 consultas por **R$ 29,90** | Melhor custo/benefício para o nosso volume. Chave de API no header. |
| **CNPJá** (API comercial) | Sim (pesquisa de estabelecimentos com filtros) | Sim | Planos a partir de ~R$ 20–40/mês em créditos; 50 créditos grátis para testar | Boa alternativa/backup; créditos compartilhados entre recursos. |
| **CNPJ.ws** (API comercial) | Sim (endpoint "Pesquisa de empresas") | Sim | Plano mensal (preço sob consulta no site) | API pública gratuita só consulta CNPJ individual (3/min), sem busca. |
| **Dados abertos da Receita** (download mensal) | Sim, depois de importar | Sim | **Grátis** | Arquivos grandes (vários GB), atualização mensal ⇒ empresas abertas há < 30–45 dias podem não aparecer ainda. Exige rotina de importação. Fica como provider opcional numa fase posterior. |
| BrasilAPI / ReceitaWS públicas | Não (só consulta por CNPJ) | Sim | Grátis | Úteis apenas para enriquecer/validar um CNPJ já conhecido. |

**Estimativa de custo com Casa dos Dados:** para entregar 20 leads válidos/dia, a busca deve trazer ~60–100 CNPJs (muitos são descartados por duplicidade, sem telefone, opt-out etc.). ≈ 100 × 22 dias úteis = 2.200 consultas/mês ⇒ **cabe no plano de R$ 29,90/mês**.

⚠️ Observação importante: o telefone/e-mail da Receita é o informado na abertura; com frequência é o **do contador**. O app vai sinalizar e-mails de domínios típicos de contabilidade e permitir marcar "contato do contador".

### b) Google Places API (New) — complemento de telefone, site e endereço
- Endpoint: `places:searchText` com field mask enxuta (`displayName, formattedAddress, nationalPhoneNumber, websiteUri`).
- Preço: Text Search Pro US$ 32 / 1.000 com **5.000 grátis/mês**; campos de telefone/site caem no nível Enterprise: US$ 35 / 1.000 com **1.000 grátis/mês** (cotas grátis por SKU, mensais).
- Uso previsto: só para leads sem celular/site ⇒ ~20–30 chamadas/dia ⇒ **≈ R$ 0/mês** dentro da cota grátis. Requer conta de faturamento no Google Cloud (cartão), com limite diário de chamadas configurado no app e no console para evitar surpresa.

### c) PNCP – Portal Nacional de Contratações Públicas
- API pública de consulta, **gratuita e sem cadastro**: `https://pncp.gov.br/api/consulta`.
- Endpoints: `/v1/contratacoes/proposta` (licitações com propostas **abertas**; só data final obrigatória) e `/v1/contratacoes/publicacao` (por data de publicação; exige data inicial, final e modalidade).
- Filtro por UF/município do órgão; o filtro de palavras-chave será feito **localmente** no campo "objeto" (a API ignora alguns filtros silenciosamente), com dicionário editável: "medicamento", "material médico", "odontológico", "hospitalar", "luva", "seringa", "EPI" etc.
- Aba própria "Compradores públicos": órgão, objeto, valor estimado, data de encerramento de propostas e link para o edital. Botão "Levar para o funil".

### d) Provider "mock"
- Gera empresas fictícias (CNPJ com dígitos válidos, telefones celulares/fixos, datas de abertura variadas) para testar tudo sem custo.

## 2. Arquitetura

```
Next.js 15 (App Router) + TypeScript
├─ UI: Tailwind CSS + componentes próprios (visual Microsoft To Do/Planner), dnd-kit (Kanban, touch + mouse), Recharts (dashboard)
├─ Server Actions / Route Handlers (API interna)
├─ Domínio (src/lib)
│   ├─ providers/      ← interface LeadProvider { id, buscar(criterios) } : casadosdados | cnpja | googlePlaces (enriquecedor) | pncp | mock
│   ├─ busca/          ← orquestrador do lote diário (60% novas / 40% estabelecidas, ampliação automática)
│   ├─ telefone.ts     ← normalização +55 DDD número, celular x fixo (libphonenumber-js)
│   ├─ dedup.ts        ← CNPJ, telefone e opt-out
│   ├─ scoring.ts      ← pontuação do lead
│   └─ links.ts        ← wa.me e mailto com modelos e variáveis
├─ Prisma ORM + SQLite (arquivo local; trocar para PostgreSQL = mudar 1 linha + migrate)
├─ Auth: login por senha (bcrypt) + sessão em cookie assinado; tabela User pronta p/ múltiplos usuários
├─ PWA: manifest + service worker (instalável no celular)
└─ Testes: Vitest (telefone, dedup, links, scoring, ampliação)
```

Comando único: `npm run setup` (instala, cria banco, semeia colunas/modelos/CNAEs) e `npm run dev`. Chaves de API só no `.env` (`.env.example` versionado, `.env` no `.gitignore`).

### Algoritmo do "Buscar leads de hoje"
1. Monta critérios a partir dos Ajustes (CNAEs ativos, região, janela de abertura).
2. **Lote A (≈60% = 12 leads)**: abertas nos últimos 12 meses, com telefone ou e-mail.
3. **Lote B (≈40% = 8 leads)**: estabelecidas (> 12 meses), ativas, com telefone/e-mail, preferindo matriz e porte compatível.
4. Normaliza telefones, descarta inválidos, deduplica (CNPJ, telefone, opt-out), enriquece pelo Google Places quando falta celular/site, calcula pontuação.
5. Se < 20 válidos, amplia em degraus até atingir o mínimo: (1) mais páginas → (2) cidades da Grande SP → (3) demais CNAEs secundários configurados → (4) janela de abertura 18 → 24 meses → (5) todo o estado. Cada degrau fica registrado no relatório da busca.
6. Os leads entram na coluna "Novo lead" com a mensagem já montada.

### Pontuação (0–100, pesos editáveis)
Empresa nova (≤ 12 meses: +30; ≤ 3 meses: +10 extra) · celular/provável WhatsApp +20 · e-mail próprio (não genérico/contador) +10 · site +5 · CNAE prioritário (ex.: odontologia, hospital) +15 · na capital/Grande SP +10 · matriz +5 · capital social/porte +5.

## 3. Modelo de dados (principais tabelas)
`User` · `Lead` (razão social, fantasia, CNPJ, CNAE, segmento, abertura, endereço, telefones, e-mail, site, fonte, data da coleta, base legal = "legítimo interesse", pontuação, etapa, valor estimado, produtos de interesse, motivo de perda, `naoContatar`) · `Phone` (E.164, tipo celular/fixo) · `Stage` (nome, ordem, tipo: aberto/ganho/perdido) · `Activity` (linha do tempo: contato WhatsApp/e-mail, nota, mudança de etapa) · `Task` (título, data, lembrete, concluída) · `Tag`/`LeadTag` · `MessageTemplate` (canal, segmento, "recém-aberta", assunto, corpo) · `OptOut` (CNPJ/telefone/e-mail, data, motivo) · `SearchRun` (data, critérios, degraus usados, quantos por fonte) · `PublicTender` (PNCP) · `Setting` (CNAEs, região, metas, dias para alerta, seu nome, WhatsApp de resposta).

## 4. Mensagens e links
- `https://wa.me/55DDDNUMERO?text=<mensagem codificada>` e `mailto:email?subject=...&body=...`.
- Variáveis: `{nome_fantasia}`, `{cidade}`, `{segmento}`, `{meu_nome}` (+ `{whatsapp_sadmed}`).
- Modelos por segmento + modelo "Empresa recém-aberta" (parabeniza pela inauguração). O app escolhe o modelo mais específico automaticamente; você pode trocar no card.
- Rodapé padrão: "Responda SAIR para não receber mais contatos." — e assinatura com o WhatsApp **+55 11 2365-3693** para respostas.
- Clique no botão ⇒ registra "Contato via WhatsApp/E-mail" no histórico e, se o card estiver em "Novo lead", oferece movê-lo para "Contatado". Nada é enviado automaticamente.

## 5. Telas
1. **Login**
2. **Hoje** (início): botão "Buscar leads de hoje", meta diária (x/20 com barra de progresso), lista de leads novos com botões WhatsApp/E-mail, "Minhas tarefas de hoje", alertas de leads parados.
3. **Funil (Kanban)**: colunas editáveis (criar/renomear/reordenar), arrastar e soltar (desktop e celular), cards com nome, cidade, segmento, selo "Empresa nova", pontuação, próxima tarefa e valor; modal de motivo ao mover para "Perdido"; filtros no topo.
4. **Detalhe do lead** (painel lateral, estilo To Do): dados cadastrais, telefones, LGPD (fonte, coleta, base legal), valor, produtos, tags, tarefas, linha do tempo, notas, "Não contatar", "Excluir definitivamente".
5. **Leads (lista/tabela)**: busca e filtros (nome, CNPJ, cidade, segmento, etapa, tag, data), cadastro manual, importar/exportar CSV/Excel.
6. **Compradores públicos (PNCP)**.
7. **Tarefas**: hoje / atrasadas / próximas.
8. **Painel**: leads por etapa, conversão entre etapas, vendas do mês, valor em negociação, motivos de perda, desempenho por segmento e por fonte.
9. **Ajustes**: CNAEs e segmentos, região, providers (ligar/desligar + teste de conexão), mix novas/estabelecidas, meta diária, dias para alerta, pesos da pontuação, modelos de mensagem, etapas do funil, opt-out, backup/restauração, usuários.

## 6. LGPD
Somente CNPJ/contatos institucionais; cada lead guarda fonte, data de coleta e base legal; opt-out bloqueia CNPJ, telefones e e-mail em todas as buscas futuras; exclusão definitiva remove o lead e o histórico, mantendo apenas o hash do CNPJ/telefone na lista de opt-out (para garantir que não volte).

## 7. Entrega em etapas
1. ✅ Este plano.
2. Base de dados + login + Kanban + cadastro manual + importação/exportação CSV/Excel + modelos e links + provider mock + testes.
3. Providers reais: Casa dos Dados (e/ou CNPJá), Google Places, PNCP.
4. Painel, tarefas/lembretes, alertas de lead parado, PWA, backup/restauração, refinamentos.

Ao final de cada etapa: instruções de como testar.

## Fontes consultadas
- Casa dos Dados – preços/planos: https://portal.casadosdados.com.br/precos , https://portal.casadosdados.com.br/planos
- Casa dos Dados – documentação da pesquisa avançada: https://docs.casadosdados.com.br/pesquisa-avan%C3%A7ada-de-empresas-16579062e0
- CNPJá – planos: https://cnpja.com/plans ; termos: https://cnpja.com/en/subscription/tos
- CNPJ.ws – API comercial: https://docs.cnpj.ws/referencia-de-api/api-comercial
- Google Places pricing (2026): https://www.woosmap.com/blog/google-places-api-pricing , https://openplacesapi.com/blog/google-places-api-pricing
- PNCP – manual e API: https://pncp.gov.br/manual/pt-br/latest/index.html , https://www.gov.br/pncp/pt-br/pncp/perguntas-e-respostas , https://apify.com/jgoia/pncp-lookup , https://www.tabnews.com.br/dmanoel/a-api-do-pncp-ignora-filtros-em-silencio-o-que-aprendi-montando-um-servidor-mcp-de-licitacoes
- Dados abertos CNPJ: https://www.socialhub.pro/blog/dados-abertos-receita-federal-cnpj/
