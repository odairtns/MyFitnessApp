# EXECUTION ENGINE — ESPECIFICAÇÃO DE IMPLEMENTAÇÃO
**Versão:** 1.0  
**Camada:** `domain/execution/` e `application/session/`  

---

## 1. Princípio de Responsabilidade Única

O **Execution Engine** é o cérebro que comanda a condução de um treino em andamento.
- **Ele faz:** Carrega um `WorkoutTemplate`, compila-o em um plano linear imutável (`ExecutionPlan`), avança passos, processa inputs do atleta, consome ticks/eventos temporais e gera fatos persistíveis para a sessão.
- **Ele NÃO faz:** Não renderiza UI, não acessa diretamente a GPU ou ViewModels do Compose, e não toca diretamente nas APIs de áudio ou sintetizador de voz (o áudio é acionado por observação de eventos de transição de passos).

---

## 2. Contratos e Interfaces Principais

```kotlin
package com.example.fitnesstrackerpro.domain.execution

import kotlinx.coroutines.flow.StateFlow
import com.example.fitnesstrackerpro.domain.model.WorkoutTemplate
import com.example.fitnesstrackerpro.domain.model.WorkoutSession

interface ExecutionCompiler {
    fun compile(template: WorkoutTemplate): ExecutionPlan
}

interface WorkoutExecutionController {
    val state: StateFlow<ExecutionState>
    fun dispatch(action: ExecutionAction)
}
```

---

## 3. Estrutura do `ExecutionPlan` e `ExecutionStep`

Um `ExecutionPlan` é uma sequência finita e linear de passos imutáveis, pré-calculada antes do treino iniciar:

```kotlin
data class ExecutionPlan(
    val templateId: Long?,
    val templateName: String,
    val modality: WorkoutModality,
    val protocol: WorkoutProtocol,
    val steps: List<ExecutionStep>
) {
    fun getStep(stepId: String): ExecutionStep? = steps.find { it.id == stepId }
    val initialStep: ExecutionStep get() = steps.first()
}

data class ExecutionStep(
    val id: String,                         // UUID ou chave determinística (ex: "b0_e1_s1_work")
    val blockIndex: Int,
    val blockName: String,
    val stepType: ExecutionStepType,
    val exerciseId: Long? = null,
    val exerciseName: String = "",
    val setNumber: Int? = null,
    val totalSets: Int? = null,
    val targetLoadKg: Double? = null,       // Em kg
    val targetReps: Int? = null,
    val durationSeconds: Int? = null,       // Duração para timers de descanso ou fases HIIT
    val side: Side = Side.NONE,             // Lateralidade do passo
    val hiitTargets: List<HiitTarget> = emptyList(),
    val nextStepId: String? = null
)

enum class ExecutionStepType {
    PREPARE,
    SET_WORK,           // Musculação / Repetições
    REST_SET,           // Descanso entre séries
    REST_BLOCK,         // Descanso entre blocos
    HIIT_WARMUP,
    HIIT_WORK,
    HIIT_REST,
    HIIT_COOLDOWN,
    AMRAP_CLOCK,        // Bloco AMRAP contínuo
    EMOM_INTERVAL,      // Intervalo de 1 minuto EMOM
    FOR_TIME_CLOCK,     // Bloco For Time contra o relógio
    STRETCH_HOLD,       // Sustentação de alongamento
    SWITCH_SIDE_REST,   // Troca de lado (3 a 5 segundos)
    RESULT_ENTRY,       // Tela de registro de rounds/reps ou saldo
    COMPLETE            // Conclusão
}
```

---

## 4. Regras do Compilador por Protocolo (`ExecutionCompiler`)

### 4.1. Musculação Tradicional (Strength & Hypertrophy)
- Para cada exercício e série planejada:
  1. Gera `ExecutionStepType.SET_WORK`.
  2. Gera `ExecutionStepType.REST_SET` (com a duração configurada em `restSeconds`).
- **Regra de Ouro (Sem descanso fantasma final):** A última série do último exercício de um bloco **NÃO** deve ser sucedida por um passo de descanso. O próximo passo aponta diretamente para o próximo exercício/bloco ou `COMPLETE`.

### 4.2. Bi-sets e Supersets (`groupType == SUPERSET`)
- Se os exercícios `A` e `B` compartilham o mesmo `groupId`:
  - `A (Série 1)` $\rightarrow$ `B (Série 1)` $\rightarrow$ `REST_SET` $\rightarrow$ `A (Série 2)` $\rightarrow$ `B (Série 2)` $\rightarrow$ `REST_SET`...

### 4.3. Alongamento e Lateralidade (`STRETCH`)
- Se `sideMode == SideMode.ALTERNATING`:
  - Passo 1: `STRETCH_HOLD (Right)`
  - Passo 2: `SWITCH_SIDE_REST` (descanso curto de 3 a 5 segundos para reposicionamento)
  - Passo 3: `STRETCH_HOLD (Left)`
  - Passo 4: `REST_SET` (se houver próxima série planejada)
- Não existe descanso pós-Left na última série do movimento.

### 4.4. HIIT Intervalado
- Gera a sequência estrita:
  - `HIIT_WARMUP` (se `warmup_seconds > 0`)
  - Loop de $N$ Rounds: `HIIT_WORK` $\rightarrow$ `HIIT_REST`
  - `HIIT_COOLDOWN` (se `cooldown_seconds > 0`)
  - `COMPLETE`

### 4.5. Circuitos Contínuos (AMRAP e For Time)
- O compilador gera um único passo temporal contínuo:
  - `AMRAP_CLOCK` (com `durationSeconds = block.workDurationSeconds`) seguido de `RESULT_ENTRY`.
  - `FOR_TIME_CLOCK` (com contagem progressiva e `timeCapSeconds = block.workDurationSeconds`) seguido de `RESULT_ENTRY` se atingir o cap.

---

## 5. Máquina de Estados da Execução (`ExecutionState`)

```kotlin
sealed interface ExecutionState {
    data object Idle : ExecutionState
    
    data class Active(
        val sessionId: Long,
        val plan: ExecutionPlan,
        val currentStep: ExecutionStep,
        val isPaused: Boolean = false,
        val elapsedTotalMs: Long = 0L,
        val currentStepRemainingMs: Long? = null,
        val circuitRoundsCount: Int = 0
    ) : ExecutionState

    data class Completed(val sessionId: Long) : ExecutionState
    data class Cancelled(val sessionId: Long) : ExecutionState
}
```

### Ações Processadas pelo Controlador:
```kotlin
sealed interface ExecutionAction {
    data class Start(val template: WorkoutTemplate) : ExecutionAction
    data object CompleteCurrentStep : ExecutionAction
    data class ConfirmSet(val actualLoadKg: Double?, val actualReps: Int?) : ExecutionAction
    data object IncrementCircuitRound : ExecutionAction
    data object DecrementCircuitRound : ExecutionAction
    data class SubmitAmrapResult(val rounds: Int, val partialExerciseId: Long?, val partialReps: Int?) : ExecutionAction
    data class SubmitForTimeResult(val elapsedSeconds: Long, val isCapped: Boolean) : ExecutionAction
    data object Pause : ExecutionAction
    data object Resume : ExecutionAction
    data object SkipCurrentStep : ExecutionAction
    data object CancelSession : ExecutionAction
    data object DiscardSession : ExecutionAction
}
```

---

## 6. Persistência de Fatos em Tempo Real (Resiliência)

Para garantir tolerância total a falhas (ex: bateria descarregada durante o treino):
1. **Promoção de DRAFT para IN_PROGRESS:** Ocorre de forma atômica no banco Room ao despachar a primeira ação de confirmação de série ou ao iniciar o primeiro timer.
2. **Gravação Contínua de Fatos:** Cada invocação de `ConfirmSet` persiste imediatamente um `PerformedSetEntity` na transação do Room. Não existe "acumular tudo em memória para salvar no fim".
3. **Estado de Recuperação do Timer:** A cada mudança de estado crítico do timer ou passo, grava-se uma linha única em `timer_recovery`.
4. **Reinício do App:** Se o app for reaberto após um encerramento inesperado, o repositório consulta sessões com status `IN_PROGRESS` e a tabela `timer_recovery`, oferecendo o diálogo de restauração imediata do ponto exato onde o atleta parou.
