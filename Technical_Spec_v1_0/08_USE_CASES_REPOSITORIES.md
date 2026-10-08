# CASOS DE USO, REPOSITÓRIOS E CONTRATOS DE CONSULTA
**Versão:** 1.0  
**Camada:** `application/` e `domain/repository/`  

---

## 1. Catálogo Completo de Casos de Uso (Use Cases)

Os casos de uso encapsulam regras de orquestração puras, sendo injetados nos ViewModels:

### 1.1. Gerenciamento de Treinos (`application/workout/`)
- **`CreateWorkoutTemplateUseCase`**: Valida a estrutura de blocos e exercícios e salva um novo template com UUID v4.
- **`UpdateWorkoutTemplateUseCase`**: Atualiza metadados e blocos de um template existente (não afeta nenhuma sessão passada).
- **`DuplicateWorkoutTemplateUseCase`**: Clona um template, gerando um novo UUID v4 e adicionando o sufixo " (Cópia)".
- **`ArchiveWorkoutTemplateUseCase`**: Marca `isArchived = true` para ocultar o template da lista sem violar histórico.
- **`DeleteWorkoutTemplateUseCase`**: Permite delete físico somente se o template nunca tiver sido executado; caso contrário, força arquivamento.

### 1.2. Execução e Sessão (`application/session/`)
- **`StartWorkoutUseCase`**: Compila o template via `ExecutionCompiler` e inicia o `WorkoutForegroundService`.
- **`CreateDraftSessionUseCase`**: Cria um registro com status `DRAFT` contendo os snapshots dos blocos e exercícios.
- **`ConfirmSetUseCase`**: Promove a sessão para `IN_PROGRESS` (se ainda for DRAFT) e persiste um `PerformedSetEntity` no Room.
- **`IncrementAmrapRoundUseCase`** / **`DecrementAmrapRoundUseCase`**: Modifica a contagem em memória com trava em zero.
- **`SaveAmrapResultUseCase`**: Persiste o `AmrapResultEntity` consolidado garantindo a invariante de reps parciais.
- **`FinishWorkoutUseCase`**: Atualiza a sessão para `COMPLETED`, calcula métricas consolidadas e limpa o estado de recuperação.
- **`CancelWorkoutUseCase`**: Atualiza a sessão para `CANCELLED`, preservando todo o esforço parcial realizado.
- **`DiscardWorkoutUseCase`**: Executa `DELETE CASCADE` físico de toda a árvore da sessão atual (utilizado para testes ou descarte voluntário de rascunhos).
- **`RecoverSessionUseCase`**: Restaura o estado da sessão interrompida inesperadamente a partir da tabela `timer_recovery`.

### 1.3. Histórico e Métricas (`application/history/`)
- **`ObserveSessionsUseCase`**: Retorna `Flow<List<WorkoutSessionSummary>>` ordenado cronologicamente decrescente.
- **`GetSessionDetailUseCase`**: Carrega todos os blocos, séries, resultados e notas de uma sessão específica.
- **`GetLastLoadByExerciseAndSetUseCase`**: Executa a busca em 2 etapas da última carga (mesmo exercício + mesma série; fallback para última carga do exercício).
- **`ObserveWeeklyVolumeUseCase`**: Calcula a tonelagem agregada (`Σ(carga_kg × reps)`) da semana corrente.

### 1.4. Ferramentas Avulsas (`application/timer/`)
- **`ControlIndependentTimerUseCase`**: Controla cronômetro, contagem regressiva e timers Tabata/EMOM da aba *Tools*.
  - **Regra Rígida de Engenharia:** Este caso de uso **nunca** injeta nem invoca o `SessionRepository`. Nenhuma sessão é gravada no banco de dados histórico.

---

## 2. Contratos de Repositório (Domain Interfaces)

```kotlin
package com.example.fitnesstrackerpro.domain.repository

import kotlinx.coroutines.flow.Flow
import com.example.fitnesstrackerpro.domain.model.*

interface WorkoutRepository {
    fun observeActiveTemplates(): Flow<List<WorkoutTemplate>>
    suspend fun getTemplateById(id: Long): WorkoutTemplate?
    suspend fun getTemplateByUuid(uuid: String): WorkoutTemplate?
    suspend fun saveTemplate(template: WorkoutTemplate): Long
    suspend fun archiveTemplate(id: Long)
    suspend fun deleteTemplate(id: Long): Boolean // Retorna false se impedido por histórico
}

interface SessionRepository {
    suspend fun createDraft(plan: ExecutionPlan): Long
    suspend fun promoteToInProgress(sessionId: Long)
    suspend fun recordPerformedSet(sessionId: Long, set: PerformedSet): Long
    suspend fun saveAmrapResult(sessionId: Long, result: AmrapResult)
    suspend fun saveForTimeResult(sessionId: Long, result: ForTimeResult)
    suspend fun completeSession(sessionId: Long, completedAtEpochMs: Long, notes: String)
    suspend fun cancelSession(sessionId: Long, cancelledAtEpochMs: Long)
    suspend fun discardSession(sessionId: Long) // DELETE CASCADE físico
    suspend fun getRecoverableSession(): RecoverableSessionData?
    suspend fun clearRecoveryState(sessionId: Long)
}

interface HistoryRepository {
    fun observeRecentSessions(limit: Int): Flow<List<WorkoutSession>>
    suspend fun getSessionDetail(sessionId: Long): WorkoutSession?
    suspend fun getLastLoad(exerciseId: Long?, exerciseName: String, setNumber: Int): Double?
    fun observeWeeklyTonnage(startOfWeekEpochMs: Long): Flow<Double>
    fun observePersonalRecords(): Flow<List<PersonalRecord>>
}

interface ExerciseRepository {
    fun observeAllExercises(): Flow<List<Exercise>>
    suspend fun searchExercises(query: String): List<Exercise>
    suspend fun getExerciseByStableKey(key: String): Exercise?
    suspend fun saveExercise(exercise: Exercise): Long
    suspend fun archiveExercise(id: Long)
}
```

---

## 3. Isolamento e Blindagem Arquitetural

1. **Repositórios Retornam Modelos de Domínio:** Todas as funções públicas dos repositórios retornam objetos puros do pacote `domain.model`. Qualquer conversão de `*Entity` ou `*Relation` é feita internamente via mappers dedicados no pacote `data/mapper/`.
2. **Tratamento de Exceções Previsível:** Erros de banco de dados ou violações de integridade são capturados e mapeados para subclasses de `DomainException` (ex: `TemplateNotFoundException`, `ActiveHistoryConstraintException`), impedindo que exceções SQLite vazem para os ViewModels.
