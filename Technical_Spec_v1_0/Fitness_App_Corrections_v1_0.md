# FITNESS APP — CORREÇÕES NECESSÁRIAS
**Versão:** 1.0  
**Base:** análise do vídeo de funcionamento do app + requisitos arquiteturais/funcionais do Fitness App  
**Objetivo:** corrigir os problemas observados no build atual antes da expansão de novas funcionalidades.

---

## 1. Objetivo deste documento

Este documento transforma a análise do vídeo de funcionamento do aplicativo em uma lista objetiva de correções de implementação.

O foco não é redesenhar o produto nem adicionar funcionalidades novas. O objetivo é fazer a implementação atual respeitar a arquitetura e os requisitos já definidos.

A correção principal está no fluxo:

```text
Workout Template
       ↓
ExecutionPlan
       ↓
Workout Player
       ↓
Workout Session
       ↓
History
```

A regra fundamental permanece:

> **Template é intenção. Session é fato.**

O Player deve executar o `ExecutionPlan`; não deve tratar o treino apenas como uma sequência linear de timers.

---

# 2. Diagnóstico geral

Os principais problemas observados são:

| ID | Prioridade | Problema | Área |
|---|---|---|---|
| COR-001 | P0 | Não é possível criar treinos de forma consistente | Workout Editor |
| COR-002 | P0 | AMRAP não mostra os exercícios do circuito durante a execução | Execution Engine / Player |
| COR-003 | P0 | Musculação não apresenta o plano completo do treino | Player / UX |
| COR-004 | P0 | Usuário não consegue escolher qual exercício executar | Execution Engine / Player |
| COR-005 | P0 | `ExecutionStep` está sendo tratado como se fosse `Exercise` na experiência do usuário | Execution Engine / UX |
| COR-006 | P0 | Editor de AMRAP usa conceito de séries de musculação | Workout Editor |
| COR-007 | P0 | `Estimated 1RM` aparece apesar de estar fora da V1 | History / PR |
| COR-008 | P0 | `MAX_REPS` apresenta unidade incorreta de kg | History / PR |
| COR-009 | P1 | Falta separar navegação entre exercícios de conclusão de exercícios | Execution Engine |
| COR-010 | P1 | Falta definir visualmente o status dos exercícios no plano | Player / UX |
| COR-011 | P1 | Falta explicitar o comportamento de seleção durante descanso | Player / Timer |
| COR-012 | P1 | Editor deve ser orientado por modalidade/protocolo | Workout Editor |

---

# 3. COR-001 — Corrigir criação de treinos

## Problema

A tela de criação observada apresenta campos genéricos de treino, mas não estabelece de forma suficientemente clara a modalidade e o protocolo que determinam como aquele treino será executado.

Isso gera um editor genérico que não representa adequadamente os diferentes tipos de treino.

## Regra

O fluxo de criação deve ser:

```text
Novo Treino
    ↓
Nome
    ↓
Modalidade
    ↓
Protocolo
    ↓
Editor específico do protocolo
    ↓
Blocos
    ↓
Exercícios / configuração
    ↓
Salvar
```

## Campos mínimos

### Modalidade

Exemplos:

- Strength / Musculação
- Cardio / HIIT
- Functional / CrossFit
- Stretch / Mobility

### Protocolo

Exemplos:

- Standard / Sets
- HIIT Interval
- AMRAP
- EMOM
- For Time
- Stretch
- outros protocolos suportados pelo produto

## Regra de UI

Depois que o protocolo for selecionado, a tela deve mostrar apenas os campos relevantes para aquele protocolo.

### Strength

Mostrar:

- exercícios;
- séries;
- reps;
- carga;
- descanso;
- lateralidade quando aplicável;
- grupos de superset quando aplicável.

### HIIT

Mostrar:

- warmup;
- work;
- rest;
- rounds;
- equipamento;
- metas do equipamento.

### AMRAP

Mostrar:

- duração;
- circuito;
- exercícios;
- reps alvo;
- warmup/cooldown quando aplicável.

### EMOM

Mostrar:

- duração do intervalo;
- número de intervalos;
- exercícios;
- reps alvo;
- carga alvo quando aplicável.

### For Time

Mostrar:

- sequência;
- exercícios;
- reps/distâncias/metas;
- Time Cap;
- classificação.

### Stretch

Mostrar:

- exercício;
- duração;
- número de séries;
- lado;
- descanso/transição.

---

# 4. COR-002 — Corrigir execução do AMRAP

## Problema observado

O AMRAP inicia corretamente o timer e apresenta o contador de rounds, porém os exercícios do circuito não aparecem adequadamente na tela de execução.

Isso significa que o Template contém os exercícios, mas o `ExecutionPlan`/Player não está carregando ou apresentando esses dados.

## Regra funcional

Um AMRAP deve possuir:

```text
duration
+
exercises[]
+
targets
+
timer
+
round result
+
partial result
```

O timer controla o tempo total do AMRAP.

O Execution Engine controla o conteúdo do circuito.

## Exemplo

Para:

```text
AMRAP 10 min

10 Burpees
15 Kettlebell Swings
```

o Player deve apresentar algo semelhante a:

```text
AMRAP

09:42

ROUND 3

CIRCUIT

1. Burpee
   10 reps

2. Kettlebell Swing
   15 reps

       + ROUND
```

## Regra arquitetural

Não transformar cada exercício do AMRAP em um timer independente.

O modelo correto é:

```text
AMRAP
 ├── Timer
 ├── Circuit
 │    ├── Exercise 1
 │    ├── Exercise 2
 │    └── ...
 └── Result
```

## Resultado

Ao terminar:

```text
completedRounds
partialExercise
partialExerciseNameSnapshot
partialReps
plannedDuration
actualDuration
scalingType
```

Exemplo:

```text
9 rounds + 7 reps de Air Squat
```

O exercício das reps parciais deve ser identificado explicitamente.

---

# 5. COR-003 — Criar visão do plano completo no Player de musculação

## Problema

A implementação apresenta principalmente:

```text
Passo X de Y
```

Isso representa o `ExecutionStep`, mas não representa adequadamente o plano que o usuário entende como treino.

O usuário precisa saber:

- quais exercícios existem;
- quais já foram concluídos;
- qual está sendo executado;
- quais ainda faltam;
- quantas séries existem;
- quantas séries foram concluídas.

## Correção

Adicionar uma ação:

```text
PLANO
```

no Player.

Exemplo:

```text
PLANO DO TREINO

✓ Supino Reto
  3/3 séries

→ Supino Inclinado
  2/3 séries

○ Tríceps Testa
  0/3 séries

○ Elevação Lateral
  0/3 séries

○ Remada Curvada
  0/3 séries
```

## Regra

O plano deve ser derivado do `ExecutionPlan` e do estado atual da `Session`.

Não deve ser reconstruído apenas com base no exercício atual.

---

# 6. COR-004 — Permitir escolher qual exercício executar

## Problema

O Player atual parece impor uma sequência linear rígida.

O usuário deve poder abrir o plano e selecionar outro exercício.

## Nova ação do Execution Engine

Adicionar uma ação equivalente a:

```text
selectStep(stepId)
```

ou:

```text
navigateToExercise(exerciseId)
```

## Regra crítica

Selecionar um exercício **não significa concluí-lo**.

Exemplo:

```text
Usuário está no exercício 2.

Seleciona exercício 5.

Resultado:

currentStep = exercício 5

Exercício 2 permanece pendente/incompleto.
Nenhuma série é criada artificialmente.
Nenhum exercício é marcado como concluído.
```

## Estados independentes

A execução deve separar:

```text
currentStep
completedSteps
pendingSteps
skippedSteps
```

A navegação altera apenas o contexto atual.

---

# 7. COR-005 — Separar ExecutionStep de Exercise

## Problema

A implementação apresenta:

```text
Passo 11 de 16
```

como principal referência para o usuário.

Internamente isso pode continuar existindo.

Porém:

> `ExecutionStep` não é `Exercise`.

Um treino pode possuir:

```text
3 exercícios
```

e:

```text
16 execution steps
```

porque os steps podem representar:

- série;
- descanso;
- exercício;
- transição;
- timer;
- mudança de lado;
- outros eventos.

## Regra

### Internamente

```text
ExecutionPlan
    ↓
ExecutionStep
```

### Na UX

Mostrar conceitos do domínio:

```text
Exercício 2 de 5
Série 3 de 3
```

e, quando relevante:

```text
Passo técnico 11 de 16
```

não como informação principal.

---

# 8. COR-006 — Corrigir semântica do editor de AMRAP

## Problema

O editor atual apresenta exercícios de AMRAP com conceito de:

```text
1 série planejada
```

Esse modelo é adequado para musculação, mas não representa corretamente um circuito AMRAP.

## Correção

Usar:

```text
CIRCUITO AMRAP

1. Burpee
   10 reps

2. Kettlebell Swing
   15 reps

[+ Adicionar exercício]
```

O conceito principal é:

> sequência de exercícios + alvo por exercício.

Não:

> exercício + séries.

## Regra de domínio

AMRAP deve utilizar estrutura própria no editor e no ExecutionPlan.

---

# 9. COR-007 — Remover Estimated 1RM da V1

## Problema

A tela de PR apresenta `ESTIMATED_1RM`.

Isso contradiz a especificação da V1.

## Correção

Remover:

```text
ESTIMATED_1RM
```

de:

- UI;
- modelo;
- cálculo;
- histórico;
- PR;
- métricas.

## PRs permitidos

```text
MAX_LOAD
MAX_REPS
MAX_VOLUME
BEST_TIME
BEST_PROTOCOL_RESULT
```

Somente recordes observados devem ser registrados.

---

# 10. COR-008 — Corrigir unidade do MAX_REPS

## Problema

A tela apresenta algo equivalente a:

```text
MAX_REPS
10.0 kg
```

Isso é incorreto.

## Regra

A unidade deve ser determinada pelo tipo do recorde.

| Record Type | Unidade |
|---|---|
| MAX_LOAD | kg |
| MAX_REPS | reps |
| MAX_VOLUME | kg |
| BEST_TIME | min:s |
| BEST_PROTOCOL_RESULT | unidade específica do protocolo |

## Regra técnica

Nunca usar a unidade de carga como unidade genérica do componente `PersonalRecord`.

O renderer deve interpretar:

```text
recordType
+
value
+
unit
```

---

# 11. COR-009 — Separar navegação de execução

Adicionar ao Execution Engine uma ação explícita:

```text
selectStep(stepId)
```

Essa ação deve:

1. validar que o step pertence à Session atual;
2. alterar o contexto atual;
3. não alterar resultados já registrados;
4. não marcar steps como concluídos;
5. não apagar resultados;
6. não alterar o Template.

---

# 12. COR-010 — Definir estados visuais do plano

Cada exercício deve apresentar pelo menos um dos seguintes estados:

```text
PENDING
CURRENT
IN_PROGRESS
COMPLETED
SKIPPED
```

Sugestão visual:

```text
✓ COMPLETED
→ CURRENT
○ PENDING
↷ SKIPPED
```

A cor não deve ser o único mecanismo de diferenciação.

---

# 13. COR-011 — Seleção de exercício durante descanso

## Cenário

```text
Supino — série concluída
↓
Rest Timer 90s
↓
Usuário abre Plano
↓
Seleciona Tríceps
```

## Regra V1

O descanso continua rodando.

O usuário pode navegar para outro exercício sem destruir o estado do Timer Engine.

Ao retornar ao contexto do exercício:

- o timer continua com o tempo correto;
- a Session permanece consistente;
- nenhum descanso é criado ou removido silenciosamente.

---

# 14. COR-012 — Editor orientado por protocolo

O editor não deve apresentar todos os campos possíveis de todas as modalidades.

A tela deve ser composta por componentes específicos:

```text
WorkoutEditor
 ├── StrengthEditor
 ├── HiitEditor
 ├── AmrapEditor
 ├── EmomEditor
 ├── ForTimeEditor
 └── StretchEditor
```

Todos podem compartilhar componentes básicos:

```text
ExerciseSelector
DurationField
RestField
RepsField
LoadField
```

mas a estrutura de cada protocolo deve continuar explícita.

---

# 15. Arquitetura corrigida do Player

A implementação deve evoluir para:

```text
WorkoutPlayer
│
├── PlayerHeader
│   ├── WorkoutName
│   ├── Progress
│   └── PlanButton
│
├── CurrentExecution
│   ├── StrengthPlayer
│   ├── HIITPlayer
│   ├── AMRAPPlayer
│   ├── EMOMPlayer
│   ├── ForTimePlayer
│   └── StretchPlayer
│
├── TimerArea
│
└── PlayerActions
```

O plano deve ser uma camada do Player:

```text
WorkoutPlanSheet
│
├── Block
│   ├── Exercise
│   ├── Status
│   ├── Sets
│   └── Select
│
└── Close
```

---

# 16. Fluxo correto de musculação

```text
START SESSION
      ↓
LOAD TEMPLATE
      ↓
COMPILE EXECUTION PLAN
      ↓
SHOW FIRST EXERCISE
      ↓
USER MAY:
      ├── Complete Set
      ├── Pause
      ├── Skip Set
      ├── Open Plan
      │      ↓
      │   Select Exercise
      │
      └── Finish
```

Ao selecionar outro exercício:

```text
ExecutionPlan.currentStep
        ↓
new selected step
```

Enquanto:

```text
Session.completedSteps
Session.performedSets
```

permanecem intactos.

---

# 17. Fluxo correto de AMRAP

```text
START SESSION
      ↓
LOAD AMRAP TEMPLATE
      ↓
COMPILE AMRAP EXECUTION PLAN
      ↓
SHOW CIRCUIT + TIMER
      ↓
START COUNTDOWN
      ↓
USER EXECUTES CIRCUIT
      ↓
+ ROUND
      ↓
ROUND + 1
      ↓
TIMER = 0
      ↓
PARTIAL RESULT ENTRY
      ↓
SAVE AMRAP RESULT
      ↓
COMPLETED
```

O `+ROUND` nunca deve alterar o Template.

---

# 18. Contratos de implementação

## Template

Responsável pelo planejamento:

```text
WorkoutTemplate
WorkoutBlock
WorkoutExercise
PlannedSet
```

## ExecutionPlan

Responsável pela execução transitória:

```text
ExecutionPlan
ExecutionStep
```

## Timer Engine

Responsável somente pelo tempo:

```text
TimerState
TimerEvent
```

## Session

Responsável pelo fato ocorrido:

```text
WorkoutSession
SessionBlock
ExercisePerformance
PerformedSet
ProtocolResult
```

## Regra

Nenhum desses componentes deve assumir a responsabilidade do outro.

---

# 19. Testes obrigatórios das correções

## 19.1 Criação

- criar Strength;
- criar HIIT;
- criar AMRAP;
- criar EMOM;
- criar For Time;
- criar Stretch;
- reabrir template;
- editar template;
- duplicar template.

## 19.2 AMRAP

- circuito aparece no Player;
- todos os exercícios aparecem;
- ordem é preservada;
- timer inicia automaticamente;
- `+ROUND` incrementa exatamente 1;
- `-ROUND` nunca fica negativo;
- reps parciais identificam o exercício;
- resultado é persistido;
- Template não é alterado.

## 19.3 Strength

- plano completo aparece;
- exercícios aparecem na ordem;
- séries aparecem corretamente;
- exercício atual é identificado;
- usuário pode selecionar outro exercício;
- selecionar exercício não marca o anterior como concluído;
- resultados continuam preservados;
- descanso continua correto.

## 19.4 PR

- MAX_LOAD mostra kg;
- MAX_REPS mostra reps;
- MAX_VOLUME mostra kg;
- BEST_TIME mostra tempo;
- Estimated 1RM não aparece.

---

# 20. Invariantes arquiteturais

Estas regras devem ser testes automatizados sempre que possível.

### INV-001

Executar um treino nunca modifica o `WorkoutTemplate`.

### INV-002

Alterar o Template depois de uma Session concluída nunca altera o histórico.

### INV-003

Selecionar outro exercício nunca cria uma conclusão artificial.

### INV-004

`+ROUND` incrementa exatamente um round.

### INV-005

`-ROUND` nunca gera rounds negativos.

### INV-006

AMRAP sempre mantém os exercícios do circuito disponíveis no Player.

### INV-007

HIIT V1 nunca registra velocidade, RPM, pace, resistência ou inclinação reais.

### INV-008

Timer independente nunca cria `WorkoutSession`.

### INV-009

Estimated 1RM não faz parte da V1.

### INV-010

Unidade de Personal Record depende do `recordType`.

---

# 21. Ordem recomendada de implementação

## P0 — Corrigir antes de qualquer nova funcionalidade

1. Corrigir modelo de criação de treino.
2. Corrigir editor orientado por protocolo.
3. Corrigir `ExecutionPlan` para carregar o conteúdo completo.
4. Corrigir AMRAP para apresentar o circuito.
5. Implementar visão `Plano do Treino`.
6. Implementar seleção de exercício.
7. Separar `ExecutionStep` de `Exercise` na UX.
8. Corrigir editor de AMRAP.
9. Remover Estimated 1RM.
10. Corrigir unidades dos PRs.

## P1 — Refinar execução

11. Status visual dos exercícios.
12. Navegação anterior/próximo.
13. Skip explícito.
14. Seleção durante descanso.
15. Recovery mostrando exercício atual.
16. Melhorar indicação de progresso por exercício.

## P2 — Refinamentos posteriores

17. Microinterações.
18. Animações.
19. Gráficos avançados.
20. Otimizações de performance baseadas em profiling.

---

# 22. Critério de aceite

A correção será considerada concluída quando:

### Criação

O usuário conseguir criar e salvar um treino de cada protocolo suportado sem recorrer a campos de outro protocolo.

### AMRAP

Ao iniciar um AMRAP, o usuário consegue ver:

- tempo restante;
- exercícios;
- ordem do circuito;
- metas/reps;
- rounds;
- botão `+ ROUND`.

### Musculação

Ao iniciar uma sessão, o usuário consegue:

- visualizar o plano completo;
- saber qual exercício está executando;
- saber quais exercícios já foram concluídos;
- saber quais faltam;
- selecionar outro exercício;
- executar suas séries;
- registrar carga e reps;
- retornar ao plano sem perder o estado da sessão.

### Histórico

A Session registra o que realmente aconteceu, independentemente da ordem em que os exercícios foram executados.

### PRs

Os tipos de recorde apresentam unidades corretas e não existe Estimated 1RM na V1.

---

# 23. Resumo para a IA de desenvolvimento

> Corrija a implementação atual respeitando a arquitetura existente do Fitness App.
>
> O principal problema está na execução: o aplicativo está tratando o treino excessivamente como uma sequência linear de steps. Reestruture o Player para trabalhar sobre um `ExecutionPlan` completo, mantendo `ExecutionStep` como conceito interno e `Exercise` como conceito de domínio/UX.
>
> O Player de musculação deve apresentar o plano completo e permitir selecionar qualquer exercício sem marcar exercícios como concluídos artificialmente.
>
> O AMRAP deve carregar e apresentar todos os exercícios do circuito durante a execução. O timer controla o AMRAP inteiro; o Execution Engine controla o circuito e o resultado.
>
> O editor deve ser orientado por protocolo. AMRAP não deve usar a mesma estrutura de séries da musculação.
>
> Remova Estimated 1RM da V1 e corrija a unidade dos Personal Records.
>
> Preserve integralmente:
>
> - Template ≠ Session;
> - ExecutionPlan separado do Template;
> - Timer Engine separado do Execution Engine;
> - HIIT com targets somente;
> - histórico baseado na Session;
> - XML somente para templates;
> - nenhuma progressão automática.
>
> Não adicione novas funcionalidades fora deste escopo.
