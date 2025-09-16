# Lógica de Agendamento e Match — Web App de Voluntariado para Igreja

> Documento técnico focado exclusivamente na **lógica de agendamento/matching** e nas **intervenções manuais guiadas** que não quebrem o sistema. Não trata de login, cadastro, pagamentos ou infra — só regras, invariantes, fluxos, pseudo‑código e exemplos práticos.

---

## Objetivo

Garantir que a igreja tenha **escalas completas**, justas e flexíveis: o sistema deve gerar escala periódica (semana/mês), suportar trocas e reagendamentos em tempo real, sugerir substitutos automaticamente, permitir intervenções manuais seguras e preservar invariantes (p.ex. funções cobertas, limites por voluntário, regras de treinamento).

## Definições principais

* **Turno (Shift)**: item atômico a ser preenchido — definido por `data_hora_inicio`, `data_hora_fim`, `local` (igreja/templo/centro), `min_vagas`/`max_vagas`, `role` (função: som, recepção, criança, louvor, cozinha, etc.), e `nivel_treinamento` (se necessário).
* **Voluntário**: pessoa com atributos: `id`, `nome`, `roles_permitidos`, `disponibilidade` (recorrente + exceções), `max_por_semana`, `max_por_dia`, `preferencias` (dias/horários), `prioridade` (p.ex. líder > regular), `ultimo_servico` (data última vez), `contadores_rotacao`.
* **Disponibilidade**: pode ser **recorrente** (ex.: todo domingo manhã; terça e quinta à noite) + **exceções** (férias, feriados, indisponível em data específica). Também pode haver **blocos temporários** que o voluntário adiciona.
* **Assignment**: ligação entre `shift` e `voluntário` com estado (`tentative`, `confirmed`, `covered_by_substitute`, `cancelled`, `no_show`).
* **Requisição de Substituição (SubstitutionRequest)**: gerada quando um voluntário não pode comparecer; inclui `shift_id`, `requestor_id`, `tipo` (auto, manual), `deadline_for_cover`, `status`.
* **Regra**: restrição global (role obrigatória, limite semanal, cooldown entre serviços seguidos, não servir em dias consecutivos para posições específicas, proibição de servir com X).

---

## Invariantes do sistema (regras que nunca podem ser violadas)

1. **RoleFilled**: cada shift que exige N voluntários deve ter pelo menos N assignments `confirmed` ou `tentative` antes da janela de lock (p.ex. 24h antes).
2. **MaxPerVolunteer**: nenhum voluntário pode ser escalado para além do seu `max_por_semana` / `max_por_dia` ou dentro de janelas proibidas.
3. **Skill/Training**: se uma role exige treinamento, só pode ser designada a voluntários com `nivel_treinamento >= exigido` ou com `exemption` aprovada.
4. **NoDoubleBooking**: um voluntário não pode ter dois shifts sobrepostos.
5. **AuditTrail**: toda mudança em uma assignment deve gerar um log imutável para auditoria.
6. **FallbackPolicy**: se um shift ficar sem substituto dentro do prazo definido, o sistema deve executar a política de fallback (escalonar para líder, abrir para banco de suplentes, pagar plantão, etc.).

---

## Visão geral da solução (duas camadas)

1. **Generator (batch)** — Gera a escala principal para um período (semana/mês) combinando disponibilidade, regras e objetivos de fairness. Executado periodicamente (e.g., 7 dias antes do período). Entrega: `assignments_tentative`.
2. **Runtime (event-driven)** — Lida com eventos: cancelamentos, trocas, substituições e confirmações. Mantém a consistência com invariantes e oferece sugestões automáticas e fluxos guiados para intervenções manuais.

Ambas camadas compartilham o mesmo modelo de regras e função de score para avaliar candidatos a substituição.

---

## Objetivos do algoritmo de match

* **Cobertura** máxima dos shifts.
* **Equidade**: distribuir carga de forma balanceada (rotacionar quem serve nos mesmos dias/horários).
* **Preferência**: atender preferências sempre que possível (p.ex. voluntário prefere manhãs).
* **Restrições**: obedecer limites e certificações.
* **Estabilidade**: evitar trocas de última hora repetitivas que criem churn.

---

## Modelagem do problema

* Pense em cada **shift** como um nó do lado esquerdo e cada **voluntário** como nó do lado direito — mas com capacidades (voluntários têm capacidade >1 por período) e múltiplas restrições por aresta (compatibilidade de role, disponibilidade, treinamento). Problema é um **constraint satisfaction + assignment optimization**.
* Para escalas pequenas: heurística greedy com backtracking é suficiente.
* Para escalas grandes (centenas de voluntários / turnos): considerar solver ILP (Integer Linear Programming) ou um algoritmo de fluxo com custos (min-cost max-flow) para otimizar fairness e preferências.

---

## Representação de disponibilidade

* **Disponibilidade recorrente**: padrões `WEEKDAY` (ex.: seg 19:00-22:00) com peso e prioridade.
* **Exceções/Blackouts**: datas específicas ou intervalos bloqueados.
* **Preferência**: marcações "prefere", "evita" (afeta score negativo/positivo).

Transformar disponibilidade em **intervalos discretos** que coincidem com shifts. Para cada shift, calcular `is_available(volunteer, shift)` baseado em recorrentes e exceções.

---

## Função de score (compatibilidade) — como escolher entre múltiplos candidatos

Calcule uma pontuação numérica `score(candidate, shift)` — quanto maior, melhor. Componentes sugeridos:

* **Disponibilidade direta (binary)**: 0 ou 1 (indispensável). Se 0 → candidato inaceitável.
* **Preferência**: +x se o voluntário prefere aquele dia/horário.
* **Proximidade temporal**: penalizar se voluntário já serviu recentemente (cooldown). Ex.: `penalty = f(days_since_last_service)`.
* **Carga atual**: penalizar quem já está com muita carga na semana/mês (`penalty_load = alpha * current_week_count / max_week`).
* **Especialização / treino**: se cargo exige treino, bônus por ter certificação.
* **Conflitos**: penalidades grandes se conflito com outra regra (ex.: não servir com X).
* **Rotação / justiça**: bônus para quem recebeu menos turnos historicamente.

Exemplo: `score = w_pref*pref + w_rot*rot_bonus - w_load*load_penalty - w_recent*recent_penalty + w_cert*cert_bonus`.

Escolha pesos (`w_*`) e teste. Mantenha limites mínimos para aceitar (p.ex. score >= threshold).

---

## Algoritmo do Generator (pseudocódigo simplificado)

```
input: list_shifts, list_volunteers, rules
output: tentative_assignments

sort shifts by criticality (ex.: cultos com mais requisitos, horários com menor pool de candidatos)
for shift in shifts:
  candidates = [v for v in volunteers if is_available(v, shift) and respects_rules(v, shift)]
  if candidates empty:
    mark shift as UNFILLED and continue
  compute score for each candidate
  sort candidates by score desc
  assigned = 0
  for c in candidates:
    if assigned >= shift.min_vagas: break
    if no_double_booking(c, shift) and not exceed_max(c, shift):
      assign c to shift as tentative
      update counters
      assigned += 1
  if assigned < shift.min_vagas:
    attempt backtracking (try different combos up to N tries)
    if still not filled: mark UNFILLED and escalate
```

Notas:

* Faça backtracking com limite (p.ex. tentar até 100 combinações) para evitar custo explosivo.
* Se usar ILP/min-cost-flow, modelar variáveis `x_{v,s} in {0,1}` e minimizar custo total = -score (ou custo = load fairness + preference penalties).

---

## Runtime: cancelamento e substituição (fluxo)

1. **Evento**: voluntário sinaliza indisponibilidade para um shift (ou falha no check-in)
2. **Criar SubstitutionRequest** com `deadline_for_cover` (p.ex. 4 horas antes do shift; configurável)
3. **Buscar candidatos automáticos**: aplicar `is_available` + `respects_rules` + `score` e retornar lista ordenada.
4. **Proposta automática**:

   * Se existe candidato com `score >= auto_accept_threshold` e `auto_replace_allowed = true`, o sistema automaticamente faz a substituição e notifica lider e voluntários envolvidos.
   * Caso contrário, envia push/sms/email para os top-K candidatos pedindo aceitação manual.
5. **Aceitação**:

   * Primeiro que aceitar ganha; sistema confirma assignment e atualiza contadores.
   * Se ninguém aceitar até o `deadline`, executar política de fallback.
6. **Fallback** (configurável): notificações para líderes → lista de suplentes → abrir shift para "voluntários disponíveis agora" → escalonar para dupla cobertura se necessário.

---

## Trocas entre voluntários (swap)

* Voluntários podem propor swap (troca) direto entre eles. Processo:

  1. Voluntário A propõe swap com Voluntário B (A quer trocar seu shift X por Y de B). O sistema valida compatibilidade: B pode assumir X? A pode assumir Y? Respeita limites?
  2. Se ambos aceitarem, swap é executado. Caso contrário, sistema tenta buscar substitutos para o shift aberto.
* O sistema **não** permite swap que viole invariantes (p.ex. exceda max\_por\_semana, deixe role sem treino).

---

## Intervenções manuais guiadas (por líderes/administradores)

Manual overrides são necessários — mas precisam ser **guiados** para não quebrar regras.

### Princípios de intervenção:

* Sempre exibir **impacto** da intervenção (antes/depois): quem fica com mais turnos, quem perde cobertura, violações potenciais.
* Exigir **confirmação explícita** para ações que quebram invariantes (por exemplo, atribuir alguém acima do `max_por_semana`) — registrar justificativa e exigir aprovação extra.
* Fornecer **sugestões seguras**: antes de permitir um override, o sistema deve mostrar alternativas (trocar com X, pedir voluntário Y) e o motivo de cada alternativa.

### Fluxos de UI para líderes (exemplos):

1. **Cobertura manual urgente**: lista de candidatos ordenada por score + checkboxes para selecionar e confirmar. Sistema checa automaticamente conflitos e exibe alertas. Ao confirmar, atualizar logs.
2. **Forçar atributo** (ex.: atribuir alguém sem treinamento): bloquear por padrão; mostrar aviso de risco com checkbox de "entendo o risco" e campo de justificativa. Requer aprovação de outro líder (modo "duplo check").
3. **Rebalancear semana**: UI que mostra heatmap de carga por voluntário; botão "rebalancear" que executa algoritmo local (move assignments de quem tem mais para quem tem menos) respeitando regras.
4. **Anular swap automático**: líderes podem reverter uma substituição feita automaticamente dentro de janela curta (p.ex. 10 minutos) com razão obrigatória.

---

## Gestão de exceções e garantia de consistência

* **Transações e locks**: toda operação de atribuição/substituição deve ocorrer em transação ACID para evitar race conditions (ex.: dois voluntários aceitam o mesmo pedido). Use row-level locks em assignments ou uma fila de eventos.
* **Idempotência**: APIs que processam accept/decline devem ser idempotentes.
* **Rate limiting**: evitar flood de notificações para os mesmos voluntários.
* **Deadlines e timers**: substituições têm timers confiáveis (cron jobs ou worker) para verificar expirados.

---

## Banco de Suplentes e "Open Pool"

* Tenha um pool de `on_call` ou `suplentes` — voluntários que se registraram para cobrir faltas de última hora (podem ter regras específicas: ganhar menos frequência, receber recompensa simbólica, etc.).
* Pool configurável por role e horário (p.ex. suplentes só para domingo manhã).
* Quando buscar substituto, tente na ordem:

  1. voluntários já escalados no mesmo dia que podem trocar (swap)
  2. voluntários disponíveis regulares (por score)
  3. suplentes on\_call
  4. líderes/administradores (fallback)

---

## Logs, auditoria e reversão

* Registrar tudo: `who`, `what`, `why`, `when` e `previous_state` antes da alteração.
* Permitir `undo` limitado (p.ex. reverter em até 24h qualquer substituição automática) com justificativa.
* Expor API para exportar históricos por período.

---

## Métricas importantes (para validar se a lógica é "perfeita")

* **Fill Rate**: porcentagem de shifts preenchidos com antecedência (ex.: 24h antes).
* **Last-Minute Fill Rate**: % de faltas cobertas em menos de X horas.
* **Swap Rate**: quantas trocas por período (muito alto → indica instabilidade).
* **Load Variance**: desvio padrão de turnos por voluntário (queremos baixo).
* **Average Time to Cover**: tempo médio entre abertura de SubstitutionRequest e confirmação de substituto.
* **False Positives/Negatives** do algoritmo (cases where algorithm suggested bad candidate).

Monitore e ajuste pesos da função de score com base nesses KPIs.

---

## Casos de borda (e como tratar)

1. **Nenhum candidato disponível**: marcar UNFILLED, abrir para suplentes, notificar líderes. Se dentro de X horas, abrir para todos os voluntários (push) com prioridade elevada.
2. **Candidatos com conflitos menores**: mostrar alertas e deixar opção para líder aceitar com justificativa.
3. **Voluntário com múltiplos perfis (roles)**: preferir alocar pela prioridade de role (ex.: som > recepção se houver conflito), ou dividir a vaga (co‑lead) se o shift permitir.
4. **Substituição por menos qualificado**: permitir somente com aprovação do líder; registrar risco.
5. **Feriados especiais**: ter regra especial com pool diferente (mais rigor para ministerios infantis) e limite de voluntários por feriado.

---

## Persistência — esquema mínimo (exemplos SQL simplificados)

```sql
-- volunteers
CREATE TABLE volunteers (
  id uuid PRIMARY KEY,
  name text,
  roles text[], -- array de funções
  max_per_week int,
  max_per_day int,
  priority int,
  training_level int,
  last_service_date date
);

-- availability
CREATE TABLE availability (
  id uuid PRIMARY KEY,
  volunteer_id uuid REFERENCES volunteers(id),
  weekday int, -- 0-6, nullable se date_range usado
  start_time time,
  end_time time,
  start_date date, -- para regras temporarias
  end_date date,
  type text -- recurring | exception
);

-- shifts
CREATE TABLE shifts (
  id uuid PRIMARY KEY,
  starts timestamptz,
  ends timestamptz,
  role text,
  min_vagas int,
  max_vagas int,
  training_required int
);

-- assignments
CREATE TABLE assignments (
  id uuid PRIMARY KEY,
  shift_id uuid REFERENCES shifts(id),
  volunteer_id uuid REFERENCES volunteers(id),
  status text, -- tentative | confirmed | cancelled | substituted
  created_at timestamptz,
  updated_at timestamptz
);

-- substitution_requests
CREATE TABLE substitution_requests (
  id uuid PRIMARY KEY,
  shift_id uuid REFERENCES shifts(id),
  requester_id uuid,
  status text,
  deadline timestamptz
);

-- audit_log (simplified)
CREATE TABLE audit_log (
  id uuid PRIMARY KEY,
  who uuid,
  action text,
  payload jsonb,
  created_at timestamptz
);
```

---

## Pseudocódigo: substituição automática com prioridade e fallback

```
on substitution_request(created):
  shift = get_shift(req.shift_id)
  candidates = get_candidates(shift) -- filters availability, training, conflicts
  scored = sort_by_score(candidates)
  for c in scored:
    if c.score >= AUTO_ACCEPT_THRESHOLD and c.auto_replace_opt_in:
      assign(c, shift)
      notify_all()
      return
  -- mandar convite para top-K candidatos
  push_notifications(top_K)
  wait for accepts until deadline
  if accept:
    assign(accepted, shift)
  else:
    execute_fallback_policy(shift)
```

---

## Estratégias para reduzir churn e "buracos" no longo prazo

* **Onboarding e compromissos claros**: limitar quem pode se inscrever como voluntário em roles críticas sem compromisso mínimo.
* **Incentivo para suplentes**: gamificação/moderação: bônus de pontos, visibilidade, reconhecimento.
* **Previsibilidade**: oferecer calendário com antecedência e janela mínima para mudanças (ex.: cancelamentos sem penalidade até 48h antes; depois gera alerta/penalidade).
* **Rotação inteligente**: evitar sempre as mesmas pessoas nos mesmos dias, mas respeitar preferências.

---

## Testes e validação

* **Unit tests**: validação de regras individuais (no\_double\_booking, max\_per\_week, treinamento).
* **Integration tests**: simular geração de escala completa para um período com dados sintéticos (diversos perfis e disponibilidade) e verificar invariantes.
* **Stress tests**: simular 10k eventos de cancelamento em curto período e validar latência e consistência.
* **A/B tests**: experimentar diferentes pesos em função de score e comparar KPIs (Fill Rate, Time to Cover, Load Variance).

---

## Checklist de segurança operacional (para não quebrar)

* Operações de atribuição em transação. rollback em caso de erro.
* Escalonamento humano transparente (logs + notificações).
* Limites de auto‑replace para evitar trocas em cascata.
* Monitoração de métricas e alertas quando Fill Rate < X%.

---

## Roadmap de implementação (prioridades técnicas)

1. Modelo de dados e API básica (shifts, volunteers, availability, assignments).
2. Engine de disponibilidade + função de score (simples, tunável via pesos).
3. Generator batch (heurística greedy + backtracking limitado).
4. Runtime substitution pipeline + notificações.
5. UI de intervenções manuais com visual de impacto e logs.
6. Melhorias: ILP/min-cost-flow se escala justificar, features de gamificação/suplentes.

---

## Anexos: exemplos rápidos

### Exemplo: regra "cooldown"

* Regra: voluntário não pode servir 2 vezes em 24h para roles que exigem alta energia.
* Implementação: ao calcular is\_available, verificar `last_service_date` e horas desde último turno; se <24h e role alta energia → disponibilidade=false.

### Exemplo: threshold automático

* `AUTO_ACCEPT_THRESHOLD = 80` (0-100). Candidatos com score >=80 e que marcaram `auto_replace_opt_in` são substitutos automáticos sem intervenção.

---

## Conclusão

A complexidade real está em equilibrar regras de negócio (treinamento, limites) com objetivos humanos (justiça, previsibilidade). A arquitetura sugerida (generator batch + runtime event-driven), combinada com uma **função de score transparente** e **intervenções manuais guiadas**, oferece um caminho robusto e auditável para construir seu web app.

Se quiser, já posso:

* Gerar o **pseudo‑código detalhado** do generator em TypeScript/Node (com exemplos de queries SQL),
* Ou montar o **diagrama de sequência** para os fluxos: (1) criação de escala, (2) cancelamento e substituição automática, (3) intervenção manual do líder.

Diz aí qual próximo artefato quer que eu gere e eu já preparo.
