# Análise de Problemas no WUZAPI (histórico e timestamps)

Contexto: histórico recente não aparece; `/chat/history` retorna apenas mensagens antigas (ex.: 12/09/2025). Abaixo estão os achados, locais exatos e como corrigir.

---

## 1) Timestamp salvo errado (usa `time.Now()` em vez do timestamp real da mensagem)
- **Local:** `_wuzapi-source/db.go`, função `saveMessageToHistory` (linha ~124).
- **Código atual:** `VALUES (..., time.Now(), ...)`
- **Impacto:** mensagens ficam registradas com a data de inserção, não a data real (`Info.Timestamp` / `RawMessage.messageTimestamp`), bagunçando a ordenação e o trim.
- **Correção sugerida:**
  - Extrair o timestamp real do payload da mensagem (quando vindo de webhook ou history sync) e salvar no DB.
  - Ajustar a assinatura de `saveMessageToHistory` para receber `messageTimestamp time.Time` em vez de usar `time.Now()`.
  - Propagar a mudança para quem chama `saveMessageToHistory` (webhooks e outgoing).
  - Ordenar e fazer trim com base no timestamp real.

## 2) `trimMessageHistory` remove registros com base no timestamp errado
- **Local:** `_wuzapi-source/db.go`, função `trimMessageHistory` (linha ~137).
- **Código atual:** deleta com `ORDER BY timestamp DESC OFFSET $limit`.
- **Impacto:** como o timestamp é o de inserção (item 1), o trim pode apagar mensagens “novas” reais e manter antigas, corrompendo o histórico.
- **Correção sugerida:**
  - Após consertar o timestamp real, o trim passa a funcionar. Antes disso, evite rodar trim ou aumente `history` para valor alto para não perder dados.

## 3) `/chat/history` lê apenas o banco local; não consulta WhatsApp em tempo real
- **Local:** `_wuzapi-source/handlers.go`, handler `GetHistory` (linha ~5737).
- **Comportamento:** faz `SELECT ... FROM message_history WHERE user_id=? AND chat_jid=? ORDER BY timestamp DESC LIMIT ?`.
- **Impacto:** se a tabela não tiver as mensagens recentes (não recebidas ou não persistidas), a API nunca vai devolvê-las.
- **Correção sugerida:**
  - Garantir que webhooks salvem tudo (entrada/saída) com timestamp real.
  - Quando precisar ressincronizar, chamar `syncHistoryForChat` (ver item 4) para popular `message_history`.

## 4) `syncHistoryForChat` existe, mas não é acionado pelo `GetHistory`
- **Local:** `_wuzapi-source/handlers.go`, função `syncHistoryForChat` (linha ~5888).
- **Comportamento:** monta `BuildHistorySyncRequest` e envia para WhatsApp, mas só é chamada por fluxos específicos (não pelo `GetHistory`).
- **Correção sugerida:**
  - Oferecer endpoint ou job que invoque `syncHistoryForChat` para chats que faltam histórico.
  - Após a resposta do WhatsApp, persistir usando o timestamp real (item 1).

## 5) Configuração `history` pode desativar ou truncar histórico
- **Local:** `_wuzapi-source/handlers.go`, `GetHistory` (linha ~5737) e `Set history` (linha ~5291).
- **Comportamento:** se `history=0`, retorna `501 message history is disabled`. Se `history` for baixo (ex.: 1000), o trim apaga mensagens antigas.
- **Correção sugerida:**
  - Para testes, usar valor alto (ex.: 100000) na criação do usuário.
  - Revisar trim (item 2) para não apagar incorretamente.

## 6) Campos retornados pelo WhatsApp são ignorados na persistência
- **Local:** `_wuzapi-source/db.go` (persistência), e quem chama `saveMessageToHistory`.
- **Comportamento:** apenas `text_content`/`media_link` são salvos; `data_json` guarda o payload bruto, mas não usa `Info.Timestamp`/`RawMessage.messageTimestamp`.
- **Correção sugerida:**
  - Extrair `Info.Timestamp` (ISO ou epoch) e `RawMessage.messageTimestamp` ao salvar.
  - Usar esse timestamp como coluna `timestamp`.

## 7) Erro de chave duplicada no HistorySync (`message_history_user_id_message_id_key`)
- **Log observado (11/12/2025):**
  - `failed to save message to history: pq: duplicate key value violates unique constraint "message_history_user_id_message_id_key"`
- **Causa provável:**
  - HistorySync traz mensagens já existentes; `saveMessageToHistory` faz `INSERT` simples, e a tabela tem UNIQUE (user_id, message_id) → viola ao reprocessar.
  - Sync pode repetir chunks ou ser reexecutada; sem upsert, falha.
- **Correção sugerida:**
  - Em `_wuzapi-source/db.go`, na `saveMessageToHistory`, trocar `INSERT` por upsert:
    - Postgres: `INSERT ... ON CONFLICT (user_id, message_id) DO NOTHING`.
    - SQLite: `INSERT OR IGNORE ...`.
  - Manter a mudança de timestamp real (item 1), para que ordenação/trim use o tempo correto.

---

## Plano de correção mínimo (passo a passo)
1. Alterar `saveMessageToHistory` para receber `messageTimestamp time.Time` e usar esse valor em vez de `time.Now()`.
2. Em todos os pontos de entrada (webhooks, outgoing, history sync), extrair timestamp real de `Info.Timestamp` ou `RawMessage.messageTimestamp` e passar para `saveMessageToHistory`.
3. Reprocessar/sincronizar chats faltantes chamando `syncHistoryForChat`, agora que o timestamp real é salvo.
4. Aumentar `history` para valor alto durante testes e, depois, reativar trim apenas com timestamps corretos.
5. (Opcional) Expôr um endpoint para forçar resync por chat usando `syncHistoryForChat`.
6. Ajustar `saveMessageToHistory` para upsert (ON CONFLICT/INSERT OR IGNORE) e evitar erro de chave duplicada no HistorySync.

---

## Efeito esperado após correção
- `/chat/history` passa a devolver mensagens em ordem cronológica real.
- Últimas mensagens recentes aparecem (não ficam presas em setembro/2025).
- Trim passa a apagar apenas mensagens realmente antigas.
