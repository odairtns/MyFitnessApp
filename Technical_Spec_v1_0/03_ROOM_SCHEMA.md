# PERSISTÊNCIA ROOM & ESQUEMA SQLITE
**Versão:** 1.0  
**Camada:** `data/local/` (Room Persistence Library 2.6+)  

---

## 1. Declaração do Banco de Dados

```kotlin
package com.example.fitnesstrackerpro.data.local

import androidx.room.Database
import androidx.room.RoomDatabase
import androidx.room.TypeConverters
import com.example.fitnesstrackerpro.data.local.converter.RoomTypeConverters
import com.example.fitnesstrackerpro.data.local.dao.*
import com.example.fitnesstrackerpro.data.local.entity.*

@Database(
    entities = [
        ExerciseEntity::class,
        WorkoutTemplateEntity::class,
        WorkoutBlockEntity::class,
        WorkoutExerciseEntity::class,
        PlannedSetEntity::class,
        WorkoutSessionEntity::class,
        SessionBlockEntity::class,
        ExercisePerformanceEntity::class,
        PerformedSetEntity::class,
        AmrapResultEntity::class,
        EmomIntervalResultEntity::class,
        ForTimeResultEntity::class,
        ForTimeRemainingItemEntity::class,
        PersonalRecordEntity::class,
        TimerRecoveryEntity::class
    ],
    version = 1,
    exportSchema = true
)
@TypeConverters(RoomTypeConverters::class)
abstract class FitnessDatabase : RoomDatabase() {
    abstract fun exerciseDao(): ExerciseDao
    abstract fun workoutTemplateDao(): WorkoutTemplateDao
    abstract fun workoutBlockDao(): WorkoutBlockDao
    abstract fun workoutSessionDao(): WorkoutSessionDao
    abstract fun historyDao(): HistoryDao
    abstract fun timerRecoveryDao(): TimerRecoveryDao

    companion object {
        const val DATABASE_NAME = "fitness_tracker_pro_v1.db"
    }
}
```

---

## 2. Entidades Relacionais e Índices

### 2.1. Planejamento (Templates)

```kotlin
@Entity(
    tableName = "exercises",
    indices = [
        Index(value = ["stable_key"], unique = true),
        Index(value = ["name"]),
        Index(value = ["category"])
    ]
)
data class ExerciseEntity(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    @ColumnInfo(name = "stable_key") val stableKey: String?,
    @ColumnInfo(name = "name") val name: String,
    @ColumnInfo(name = "category") val category: String,
    @ColumnInfo(name = "muscle_group") val muscleGroup: String,
    @ColumnInfo(name = "equipment") val equipment: String,
    @ColumnInfo(name = "movement_pattern") val movementPattern: String?,
    @ColumnInfo(name = "description") val description: String,
    @ColumnInfo(name = "default_rest_seconds") val defaultRestSeconds: Int,
    @ColumnInfo(name = "is_bodyweight") val isBodyweight: Boolean,
    @ColumnInfo(name = "is_custom") val isCustom: Boolean,
    @ColumnInfo(name = "is_archived") val isArchived: Boolean = false
)

@Entity(
    tableName = "workout_templates",
    indices = [Index(value = ["template_uuid"], unique = true)]
)
data class WorkoutTemplateEntity(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    @ColumnInfo(name = "template_uuid") val templateUuid: String,
    @ColumnInfo(name = "name") val name: String,
    @ColumnInfo(name = "modality") val modality: String,
    @ColumnInfo(name = "protocol") val protocol: String,
    @ColumnInfo(name = "description") val description: String,
    @ColumnInfo(name = "estimated_duration_seconds") val estimatedDurationSeconds: Int?,
    @ColumnInfo(name = "is_archived") val isArchived: Boolean = false,
    @ColumnInfo(name = "created_at_epoch_ms") val createdAtEpochMs: Long,
    @ColumnInfo(name = "updated_at_epoch_ms") val updatedAtEpochMs: Long
)

@Entity(
    tableName = "workout_blocks",
    foreignKeys = [
        ForeignKey(
            entity = WorkoutTemplateEntity::class,
            parentColumns = ["id"],
            childColumns = ["template_id"],
            onDelete = ForeignKey.CASCADE
        )
    ],
    indices = [Index(value = ["template_id"])]
)
data class WorkoutBlockEntity(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    @ColumnInfo(name = "template_id") val templateId: Long,
    @ColumnInfo(name = "block_type") val blockType: String,
    @ColumnInfo(name = "position") val position: Int,
    @ColumnInfo(name = "name") val name: String,
    @ColumnInfo(name = "rounds") val rounds: Int,
    // Coluna universal de trabalho em segundos (HIIT work, AMRAP duration, EMOM interval, For Time cap)
    @ColumnInfo(name = "work_duration_seconds") val workDurationSeconds: Int?,
    // Coluna universal de descanso
    @ColumnInfo(name = "rest_duration_seconds") val restDurationSeconds: Int?,
    // JSON tipado estrito via Room TypeConverter (List<TargetMetric> ou scaling configs)
    @ColumnInfo(name = "protocol_config_json") val protocolConfigJson: String?
)

@Entity(
    tableName = "workout_exercises",
    foreignKeys = [
        ForeignKey(
            entity = WorkoutBlockEntity::class,
            parentColumns = ["id"],
            childColumns = ["block_id"],
            onDelete = ForeignKey.CASCADE
        ),
        ForeignKey(
            entity = ExerciseEntity::class,
            parentColumns = ["id"],
            childColumns = ["exercise_id"],
            onDelete = ForeignKey.RESTRICT
        )
    ],
    indices = [Index(value = ["block_id"]), Index(value = ["exercise_id"])]
)
data class WorkoutExerciseEntity(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    @ColumnInfo(name = "block_id") val blockId: Long,
    @ColumnInfo(name = "exercise_id") val exerciseId: Long?,
    @ColumnInfo(name = "exercise_name_custom") val exerciseNameCustom: String?,
    @ColumnInfo(name = "position") val position: Int,
    @ColumnInfo(name = "group_id") val groupId: String?,
    @ColumnInfo(name = "group_type") val groupType: String
)

@Entity(
    tableName = "planned_sets",
    foreignKeys = [
        ForeignKey(
            entity = WorkoutExerciseEntity::class,
            parentColumns = ["id"],
            childColumns = ["workout_exercise_id"],
            onDelete = ForeignKey.CASCADE
        )
    ],
    indices = [Index(value = ["workout_exercise_id"])]
)
data class PlannedSetEntity(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    @ColumnInfo(name = "workout_exercise_id") val workoutExerciseId: Long,
    @ColumnInfo(name = "set_number") val setNumber: Int,
    @ColumnInfo(name = "target_reps") val targetReps: Int?,
    @ColumnInfo(name = "target_load_kg") val targetLoadKg: Double?, // Canônico em kg
    @ColumnInfo(name = "target_duration_seconds") val targetDurationSeconds: Int?,
    @ColumnInfo(name = "rest_seconds") val restSeconds: Int?,
    @ColumnInfo(name = "side_mode") val sideMode: String
)
```

---

### 2.2. Execução (Sessions e Snapshots)

```kotlin
@Entity(
    tableName = "workout_sessions",
    foreignKeys = [
        ForeignKey(
            entity = WorkoutTemplateEntity::class,
            parentColumns = ["id"],
            childColumns = ["template_id"],
            onDelete = ForeignKey.SET_NULL
        )
    ],
    indices = [Index(value = ["template_id"]), Index(value = ["started_at_epoch_ms"])]
)
data class WorkoutSessionEntity(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    @ColumnInfo(name = "template_id") val templateId: Long?,
    @ColumnInfo(name = "template_name_snapshot") val templateNameSnapshot: String,
    @ColumnInfo(name = "modality_snapshot") val modalitySnapshot: String,
    @ColumnInfo(name = "protocol_snapshot") val protocolSnapshot: String,
    @ColumnInfo(name = "started_at_epoch_ms") val startedAtEpochMs: Long,
    @ColumnInfo(name = "completed_at_epoch_ms") val completedAtEpochMs: Long?,
    @ColumnInfo(name = "status") val status: String, // DRAFT / IN_PROGRESS / COMPLETED / CANCELLED
    @ColumnInfo(name = "notes") val notes: String
)

@Entity(
    tableName = "session_blocks",
    foreignKeys = [
        ForeignKey(
            entity = WorkoutSessionEntity::class,
            parentColumns = ["id"],
            childColumns = ["session_id"],
            onDelete = ForeignKey.CASCADE
        )
    ],
    indices = [Index(value = ["session_id"])]
)
data class SessionBlockEntity(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    @ColumnInfo(name = "session_id") val sessionId: Long,
    @ColumnInfo(name = "source_block_id") val sourceBlockId: Long?,
    @ColumnInfo(name = "block_name_snapshot") val blockNameSnapshot: String,
    @ColumnInfo(name = "block_type_snapshot") val blockTypeSnapshot: String,
    @ColumnInfo(name = "position") val position: Int,
    @ColumnInfo(name = "status") val status: String
)

@Entity(
    tableName = "exercise_performances",
    foreignKeys = [
        ForeignKey(
            entity = SessionBlockEntity::class,
            parentColumns = ["id"],
            childColumns = ["session_block_id"],
            onDelete = ForeignKey.CASCADE
        ),
        ForeignKey(
            entity = ExerciseEntity::class,
            parentColumns = ["id"],
            childColumns = ["exercise_id"],
            onDelete = ForeignKey.SET_NULL
        )
    ],
    indices = [Index(value = ["session_block_id"]), Index(value = ["exercise_id"])]
)
data class ExercisePerformanceEntity(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    @ColumnInfo(name = "session_block_id") val sessionBlockId: Long,
    @ColumnInfo(name = "exercise_id") val exerciseId: Long?,
    @ColumnInfo(name = "exercise_name_snapshot") val exerciseNameSnapshot: String,
    @ColumnInfo(name = "position") val position: Int
)

@Entity(
    tableName = "performed_sets",
    foreignKeys = [
        ForeignKey(
            entity = ExercisePerformanceEntity::class,
            parentColumns = ["id"],
            childColumns = ["performance_id"],
            onDelete = ForeignKey.CASCADE
        )
    ],
    indices = [Index(value = ["performance_id"])]
)
data class PerformedSetEntity(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    @ColumnInfo(name = "performance_id") val performanceId: Long,
    @ColumnInfo(name = "set_number") val setNumber: Int,
    @ColumnInfo(name = "actual_load_kg") val actualLoadKg: Double?, // Estritamente em kg
    @ColumnInfo(name = "actual_reps") val actualReps: Int?,
    @ColumnInfo(name = "duration_seconds") val durationSeconds: Int?,
    @ColumnInfo(name = "side") val side: String,
    @ColumnInfo(name = "is_completed") val isCompleted: Boolean,
    @ColumnInfo(name = "logged_at_epoch_ms") val loggedAtEpochMs: Long,
    @ColumnInfo(name = "notes") val notes: String
)
```

---

### 2.3. Resultados de Circuitos e Recuperação de Timer

```kotlin
@Entity(
    tableName = "amrap_results",
    foreignKeys = [
        ForeignKey(
            entity = SessionBlockEntity::class,
            parentColumns = ["id"],
            childColumns = ["session_block_id"],
            onDelete = ForeignKey.CASCADE
        )
    ],
    indices = [Index(value = ["session_block_id"], unique = true)]
)
data class AmrapResultEntity(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    @ColumnInfo(name = "session_block_id") val sessionBlockId: Long,
    @ColumnInfo(name = "completed_rounds") val completedRounds: Int,
    @ColumnInfo(name = "partial_exercise_id") val partialExerciseId: Long?,
    @ColumnInfo(name = "partial_exercise_name_snapshot") val partialExerciseNameSnapshot: String?,
    @ColumnInfo(name = "partial_reps") val partialReps: Int?,
    @ColumnInfo(name = "planned_duration_seconds") val plannedDurationSeconds: Long,
    @ColumnInfo(name = "actual_duration_seconds") val actualDurationSeconds: Long,
    @ColumnInfo(name = "scaling_type") val scalingType: String?
)

@Entity(
    tableName = "for_time_results",
    foreignKeys = [
        ForeignKey(
            entity = SessionBlockEntity::class,
            parentColumns = ["id"],
            childColumns = ["session_block_id"],
            onDelete = ForeignKey.CASCADE
        )
    ],
    indices = [Index(value = ["session_block_id"], unique = true)]
)
data class ForTimeResultEntity(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    @ColumnInfo(name = "session_block_id") val sessionBlockId: Long,
    @ColumnInfo(name = "elapsed_seconds") val elapsedSeconds: Long,
    @ColumnInfo(name = "time_cap_seconds") val timeCapSeconds: Long?,
    @ColumnInfo(name = "status") val status: String,
    @ColumnInfo(name = "scaling_type") val scalingType: String?
)

@Entity(
    tableName = "for_time_remaining_items",
    foreignKeys = [
        ForeignKey(
            entity = ForTimeResultEntity::class,
            parentColumns = ["id"],
            childColumns = ["result_id"],
            onDelete = ForeignKey.CASCADE
        )
    ],
    indices = [Index(value = ["result_id"])]
)
data class ForTimeRemainingItemEntity(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    @ColumnInfo(name = "result_id") val resultId: Long,
    @ColumnInfo(name = "exercise_id") val exerciseId: Long?,
    @ColumnInfo(name = "exercise_name_snapshot") val exerciseNameSnapshot: String,
    @ColumnInfo(name = "target_amount") val targetAmount: Int,
    @ColumnInfo(name = "completed_amount") val completedAmount: Int,
    @ColumnInfo(name = "remaining_amount") val remainingAmount: Int,
    @ColumnInfo(name = "unit") val unit: String
)

@Entity(
    tableName = "timer_recovery",
    indices = [Index(value = ["session_id"], unique = true)]
)
data class TimerRecoveryEntity(
    @PrimaryKey val sessionId: Long,
    @ColumnInfo(name = "current_step_id") val currentStepId: String,
    @ColumnInfo(name = "target_end_elapsed_realtime") val targetEndElapsedRealtime: Long,
    @ColumnInfo(name = "remaining_duration_ms") val remainingDurationMs: Long,
    @ColumnInfo(name = "is_paused") val isPaused: Boolean,
    @ColumnInfo(name = "updated_at_epoch_ms") val updatedAtEpochMs: Long
)
```

---

## 3. Consultas Críticas e Resolução de Cargas

### Query Canônica para Última Carga Executada:

```kotlin
@Dao
interface HistoryDao {

    // 1. Resolução Primária: Mesmo Exercício + Mesmo Número da Série
    @Query("""
        SELECT ps.actual_load_kg 
        FROM performed_sets ps
        INNER JOIN exercise_performances ep ON ps.performance_id = ep.id
        INNER JOIN session_blocks sb ON ep.session_block_id = sb.id
        INNER JOIN workout_sessions ws ON sb.session_id = ws.id
        WHERE (ep.exercise_id = :exerciseId OR ep.exercise_name_snapshot = :exerciseName)
          AND ps.set_number = :setNumber
          AND ps.is_completed = 1
          AND ps.actual_load_kg IS NOT NULL
          AND ws.status IN ('COMPLETED', 'CANCELLED')
        ORDER BY ws.started_at_epoch_ms DESC, ps.logged_at_epoch_ms DESC
        LIMIT 1
    """)
    suspend fun getLastLoadForSet(
        exerciseId: Long?,
        exerciseName: String,
        setNumber: Int
    ): Double?

    // 2. Resolução Secundária (Fallback): Última Carga Válida de Qualquer Série do Exercício
    @Query("""
        SELECT ps.actual_load_kg 
        FROM performed_sets ps
        INNER JOIN exercise_performances ep ON ps.performance_id = ep.id
        INNER JOIN session_blocks sb ON ep.session_block_id = sb.id
        INNER JOIN workout_sessions ws ON sb.session_id = ws.id
        WHERE (ep.exercise_id = :exerciseId OR ep.exercise_name_snapshot = :exerciseName)
          AND ps.is_completed = 1
          AND ps.actual_load_kg IS NOT NULL
          AND ws.status IN ('COMPLETED', 'CANCELLED')
        ORDER BY ws.started_at_epoch_ms DESC, ps.logged_at_epoch_ms DESC
        LIMIT 1
    """)
    suspend fun getLastLoadFallback(
        exerciseId: Long?,
        exerciseName: String
    ): Double?
}
```

---

## 4. Gerenciamento de Ciclo de Vida: Cancelamento vs Descarte

```kotlin
@Dao
interface WorkoutSessionDao {

    // Cancelamento: Mantém registro para preservar dados do histórico parcial
    @Query("""
        UPDATE workout_sessions 
        SET status = 'CANCELLED', completed_at_epoch_ms = :completedAt 
        WHERE id = :sessionId
    """)
    suspend fun markAsCancelled(sessionId: Long, completedAt: Long)

    // Descarte Físico: DELETE CASCADE completo de toda a árvore
    @Query("DELETE FROM workout_sessions WHERE id = :sessionId")
    suspend fun discardSessionPhysically(sessionId: Long)
}
```
Ao invocar `discardSessionPhysically`, as Foreign Keys com `onDelete = ForeignKey.CASCADE` removem automaticamente todos os registros correspondentes em `session_blocks`, `exercise_performances`, `performed_sets`, `amrap_results` e `for_time_results`.
