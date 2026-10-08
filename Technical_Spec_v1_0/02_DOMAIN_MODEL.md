# MODELO DE DOMÍNIO E MÁQUINAS DE ESTADO
**Versão:** 1.0  
**Camada:** `domain/` (Kotlin Puro, Agnóstico de Plataforma)  

---

## 1. Entidades Fundamentais de Domínio

### 1.1. Catálogo e Planejamento (Intenção)

```kotlin
package com.example.fitnesstrackerpro.domain.model

data class Exercise(
    val id: Long = 0,
    val stableKey: String? = null, // Chave semântica para catálogo nativo (ex: "barbell-bench-press")
    val name: String,
    val category: ExerciseCategory,
    val muscleGroup: MuscleGroup,
    val equipment: EquipmentType,
    val movementPattern: MovementPattern? = null,
    val description: String = "",
    val defaultRestSeconds: Int = 60,
    val isBodyweight: Boolean = false,
    val isCustom: Boolean = false,
    val isArchived: Boolean = false
)

data class WorkoutTemplate(
    val id: Long = 0,
    val templateUuid: String, // UUID v4 único e exportável via XML
    val name: String,
    val modality: WorkoutModality,
    val protocol: WorkoutProtocol,
    val description: String = "",
    val estimatedDurationSeconds: Int? = null,
    val blocks: List<WorkoutBlock> = emptyList(),
    val isArchived: Boolean = false,
    val createdAtEpochMs: Long,
    val updatedAtEpochMs: Long
)

data class WorkoutBlock(
    val id: Long = 0,
    val templateId: Long = 0,
    val blockType: BlockType,
    val position: Int,
    val name: String,
    val rounds: Int = 1,
    // Coluna universal de tempo de execução/trabalho em segundos
    val workDurationSeconds: Int? = null, 
    // Coluna universal de descanso entre séries/estações
    val restDurationSeconds: Int? = null, 
    // Configuração tipada específica do protocolo (ex: alvos HIIT, regras de escala)
    val protocolConfig: ProtocolConfig? = null, 
    val exercises: List<WorkoutExercise> = emptyList()
)

data class WorkoutExercise(
    val id: Long = 0,
    val blockId: Long = 0,
    val exerciseId: Long?,
    val exerciseNameCustom: String? = null,
    val position: Int,
    val groupId: String? = null,
    val groupType: GroupType = GroupType.NONE,
    val plannedSets: List<PlannedSet> = emptyList()
)

data class PlannedSet(
    val id: Long = 0,
    val workoutExerciseId: Long = 0,
    val setNumber: Int,
    val targetReps: Int? = null,
    val targetLoadKg: Double? = null, // Estritamente em kg
    val targetDurationSeconds: Int? = null,
    val restSeconds: Int? = null,
    val sideMode: SideMode = SideMode.BILATERAL
)
```

---

### 1.2. Execução Real e Histórico (Fato Biológico)

```kotlin
data class WorkoutSession(
    val id: Long = 0,
    val templateId: Long? = null, // Opcional: mera referência, sem acoplamento rígido
    val templateNameSnapshot: String,
    val modalitySnapshot: WorkoutModality,
    val protocolSnapshot: WorkoutProtocol,
    val startedAtEpochMs: Long,
    val completedAtEpochMs: Long? = null,
    val status: SessionStatus = SessionStatus.DRAFT,
    val notes: String = "",
    val blocks: List<SessionBlock> = emptyList()
)

data class SessionBlock(
    val id: Long = 0,
    val sessionId: Long = 0,
    val sourceBlockId: Long? = null,
    val blockNameSnapshot: String,
    val blockTypeSnapshot: BlockType,
    val position: Int,
    val status: BlockExecutionStatus = BlockExecutionStatus.PENDING,
    val performances: List<ExercisePerformance> = emptyList(),
    // Resultados de protocolos contínuos
    val amrapResult: AmrapResult? = null,
    val forTimeResult: ForTimeResult? = null,
    val emomResults: List<EmomIntervalResult> = emptyList()
)

data class ExercisePerformance(
    val id: Long = 0,
    val sessionBlockId: Long = 0,
    val exerciseId: Long? = null,
    val exerciseNameSnapshot: String,
    val position: Int,
    val performedSets: List<PerformedSet> = emptyList()
)

data class PerformedSet(
    val id: Long = 0,
    val performanceId: Long = 0,
    val setNumber: Int,
    val actualLoadKg: Double? = null, // Estritamente armazenado em kg
    val actualReps: Int? = null,
    val durationSeconds: Int? = null,
    val side: Side = Side.NONE,
    val isCompleted: Boolean = true,
    val loggedAtEpochMs: Long,
    val notes: String = ""
)
```

---

### 1.3. Resultados de Protocolos Específicos

```kotlin
data class AmrapResult(
    val completedRounds: Int,
    val partial: AmrapPartialResult?, // Só preenchido se houver reps parciais válidas
    val plannedDurationSeconds: Long,
    val actualDurationSeconds: Long,
    val scalingType: ScalingType?
) {
    init {
        require(completedRounds >= 0) { "completedRounds não pode ser negativo." }
    }
}

data class AmrapPartialResult(
    val exerciseId: Long?,
    val exerciseNameSnapshot: String,
    val partialReps: Int
) {
    init {
        require(partialReps > 0) { "partialReps deve ser maior que zero." }
        require(exerciseNameSnapshot.isNotBlank()) { "exerciseNameSnapshot obrigatório para reps parciais." }
    }
}

data class ForTimeResult(
    val elapsedSeconds: Long,
    val timeCapSeconds: Long?,
    val status: ForTimeStatus,
    val scalingType: ScalingType?,
    val remainingItems: List<ForTimeRemainingItem> = emptyList()
)

data class ForTimeRemainingItem(
    val exerciseId: Long?,
    val exerciseNameSnapshot: String,
    val targetAmount: Int,
    val completedAmount: Int,
    val remainingAmount: Int,
    val unit: String = "reps"
)

data class EmomIntervalResult(
    val id: Long = 0,
    val sessionBlockId: Long = 0,
    val intervalNumber: Int,
    val targetReps: Int,
    val actualReps: Int? = null,
    val targetLoadKg: Double? = null,
    val actualLoadKg: Double? = null,
    val status: EmomIntervalStatus = EmomIntervalStatus.NOT_LOGGED
)
```

---

### 1.4. Modelo de Metas HIIT (Targets Apenas)

```kotlin
data class HiitTarget(
    val type: HiitTargetType,
    val value: String, // String para comportar formatações como Pace "04:30" ou velocidades "14.5"
    val unit: HiitTargetUnit
)

enum class HiitTargetType {
    SPEED, INCLINE, PACE, RPM, SPM, RESISTANCE, DAMPER, HEART_RATE_ZONE
}

enum class HiitTargetUnit {
    KM_H, MPH, PERCENT, MIN_PER_KM, MIN_PER_MILE, RPM, STROKES_PER_MIN, LEVEL, ZONE
}
```

---

## 2. Enums Canônicos e Vocabulário Controlado

```kotlin
enum class SessionStatus {
    DRAFT,          // Criada ao abrir o player; não confirmou evidência factual
    IN_PROGRESS,    // Converteu-se em treino ativo ao completar primeira série/iniciar relógio
    COMPLETED,      // Treino concluído normalmente pelo atleta
    CANCELLED       // Treino interrompido voluntariamente; preserva histórico parcial
    // Nota: DISCARDED não é um status persistido, é uma ação de DELETE CASCADE físico
}

enum class ExecutionPlayerState {
    READY,
    RUNNING,
    PAUSED,
    COMPLETING,
    COMPLETED,
    CANCELLED,
    RECOVERY_REQUIRED
}

enum class TimerState {
    IDLE,
    RUNNING,
    PAUSED,
    COMPLETED,
    CANCELLED
}

enum class GroupType {
    NONE,
    SUPERSET // Único tipo de agrupamento na V1
}

enum class EmomIntervalStatus {
    COMPLETED,
    PARTIAL,
    SKIPPED,
    NOT_LOGGED
}

enum class ForTimeStatus {
    COMPLETED,
    CAP_REACHED
}

enum class ScalingType {
    RX,
    SCALED,
    CUSTOM
}

enum class SideMode {
    BILATERAL,
    ALTERNATING,
    LEFT,
    RIGHT,
    NONE
}

enum class Side {
    BILATERAL,
    LEFT,
    RIGHT,
    NONE
}

enum class WorkoutModality {
    STRENGTH,
    CARDIO,
    CROSS_TRAINING,
    MOBILITY
}

enum class WorkoutProtocol {
    STANDARD_STRENGTH,
    HYPERTROPHY,
    HIIT_INTERVALS,
    AMRAP,
    EMOM,
    FOR_TIME,
    STRETCH
}

enum class BlockType {
    WARMUP,
    WORK,
    SUPERSET,
    CIRCUIT,
    COOLDOWN
}
```

---

## 3. Invariantes de Domínio e Validações Rígidas

1. **Invariante de Séries Parciais no AMRAP:** Não é permitido instanciar um resultado com reps parciais sem apontar o exercício específico em que o tempo expirou. Se o tempo zerar entre exercícios ou no início do round, `partial` deve ser `null`.
2. **Invariante de Restante no For Time:** Se `status == ForTimeStatus.COMPLETED`, a lista `remainingItems` deve ser vazia. Se `status == ForTimeStatus.CAP_REACHED`, a lista deve detalhar o saldo restante por exercício individual.
3. **Invariante de Lateralidade:** Séries em `SideMode.ALTERNATING` compilam obrigatoriamente para pares de `ExecutionStep` com `Side.RIGHT` seguido de `Side.LEFT`.
4. **Invariante de Carga Canônica:** Nenhuma entidade de domínio aceita valores de carga em Libras. As propriedades `targetLoadKg` e `actualLoadKg` validam rigorosamente $\ge 0.0\text{ kg}$.
