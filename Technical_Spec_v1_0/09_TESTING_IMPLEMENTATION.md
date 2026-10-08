# PLANO DE TESTES & ENGENHARIA DE QUALIDADE
**Versão:** 1.0  
**Camada:** `test/` (JVM Unit Tests) e `androidTest/` (Instrumented / Room Tests)  

---

## 1. Pirâmide e Estratégia de Testes

```text
       / \
      /   \     Testes de Dispositivo Real (Background, Audio Focus, Bateria, Lock Screen)
     /     \
    /-------\   Testes Instrumentados & Room (Transações atômicas, FKs, Migrações SQLite)
   /         \
  /-----------\ Testes de Unidade do Domínio (ExecutionCompiler, Aritmética do Timer, UseCases)
```

1. **Testes de Unidade Puros (JVM):** Executam em milissegundos sem emulador, validando 100% da lógica matemática do compilador, máquinas de estado e regras de domínio.
2. **Testes de Persistência (Room In-Memory):** Validam integridade referencial, queries de última carga e rollbacks transacionais.
3. **Testes de Integração de Runtime:** Simulam a execução ponta a ponta do `WorkoutExecutionController` conectado a um dublê de tempo monotônico.

---

## 2. Testes Obrigatórios de Invariantes de Arquitetura

O pipeline de integração contínua (CI) deve obrigatoriamente validar as seguintes asserções:

1. **Imutabilidade do Template:**
   ```kotlin
   @Test
   fun executionNeverMutatesWorkoutTemplate() = runTest {
       val templateOriginal = createSampleStrengthTemplate()
       val plan = compiler.compile(templateOriginal)
       
       // Simula atleta executando série com peso e reps completamente diferentes
       playerController.start(plan)
       playerController.confirmSet(actualLoadKg = 120.0, actualReps = 12)
       playerController.finish()

       val templateAposTreino = workoutRepository.getTemplateById(templateOriginal.id)
       assertEquals(templateOriginal, templateAposTreino) // Template permanece idêntico
   }
   ```

2. **Independência Histórica de Snapshots:**
   ```kotlin
   @Test
   fun completedSessionPreservesSnapshotWhenExerciseIsRenamedOrArchived() = runTest {
       val exId = exerciseRepository.saveExercise(Exercise(name = "Supino Reto"))
       val sessionId = runAndCompleteSampleWorkout(exId)

       // Modifica o exercício original
       exerciseRepository.archiveExercise(exId)

       val session = historyRepository.getSessionDetail(sessionId)
       assertEquals("Supino Reto", session.blocks.first().performances.first().exerciseNameSnapshot)
   }
   ```

3. **Cancelamento vs Descarte:**
   ```kotlin
   @Test
   fun cancelledWorkoutPreservesPartialSetsWhileDiscardedDeletesCascade() = runTest {
       // Caso 1: Cancelar treino em andamento
       val session1Id = startWorkoutAndLogTwoSets()
       sessionRepository.cancelSession(session1Id, System.currentTimeMillis())
       val sessionCancelada = sessionRepository.getRecoverableSession() // Não mais recuperável
       val historico1 = historyRepository.getSessionDetail(session1Id)
       assertEquals(SessionStatus.CANCELLED, historico1?.status)
       assertEquals(2, historico1?.blocks?.first()?.performances?.first()?.performedSets?.size)

       // Caso 2: Descartar treino de teste
       val session2Id = startWorkoutAndLogTwoSets()
       sessionRepository.discardSession(session2Id)
       val historico2 = historyRepository.getSessionDetail(session2Id)
       assertNull(historico2) // Removido fisicamente pelo DELETE CASCADE
   }
   ```

4. **Guarda de Duração do Alerta de 10 Segundos:**
   ```kotlin
   @Test
   fun warning10SecondsIsSuppressedWhenPhaseDurationIsLessOrEqualTo15Seconds() = runTest {
       val timerEvents = mutableListOf<TimerEvent>()
       val timer = TestableTimerEngine(onEvent = { timerEvents.add(it) })

       // Fase curta (10 segundos - Ex: Tabata Rest)
       timer.startCountdown(durationSeconds = 10)
       timer.advanceTime(seconds = 10)

       assertFalse(timerEvents.contains(TimerEvent.Warning10Seconds))
       assertTrue(timerEvents.contains(TimerEvent.CountdownTick(3)))
       assertTrue(timerEvents.contains(TimerEvent.CountdownTick(2)))
       assertTrue(timerEvents.contains(TimerEvent.CountdownTick(1)))
   }
   ```

5. **Ferramentas Avulsas (Tools) Não Criam Sessão:**
   ```kotlin
   @Test
   fun independentStopwatchNeverCreatesWorkoutSession() = runTest {
       val initialSessionCount = sessionDao.countAllSessions()
       independentTimerUseCase.startStopwatch()
       independentTimerUseCase.advanceTime(120_000)
       independentTimerUseCase.stopStopwatch()

       assertEquals(initialSessionCount, sessionDao.countAllSessions())
   }
   ```

---

## 3. Teste do Compilador de Execução (`ExecutionCompilerTest`)

Valida as regras de negócio de compilação sem depender de Android:

```kotlin
class DefaultExecutionCompilerTest {

    private val compiler = DefaultExecutionCompiler()

    @Test
    fun strengthProtocolHasNoFinalRestAfterLastSet() {
        val template = createStrengthTemplate(setsCount = 3, restSeconds = 90)
        val plan = compiler.compile(template)

        val steps = plan.steps
        assertEquals(ExecutionStepType.SET_WORK, steps[0].stepType)
        assertEquals(ExecutionStepType.REST_SET, steps[1].stepType)
        assertEquals(ExecutionStepType.SET_WORK, steps[2].stepType)
        assertEquals(ExecutionStepType.REST_SET, steps[3].stepType)
        assertEquals(ExecutionStepType.SET_WORK, steps[4].stepType)
        // O passo 5 aponta para COMPLETE sem descanso intermediário
        assertEquals(ExecutionStepType.COMPLETE, steps[5].stepType)
    }

    @Test
    fun alternatingStretchCompilesToRightThenSwitchRestThenLeft() {
        val template = createAlternatingStretchTemplate(holdSeconds = 30)
        val plan = compiler.compile(template)

        val steps = plan.steps
        assertEquals(Side.RIGHT, steps[0].side)
        assertEquals(ExecutionStepType.STRETCH_HOLD, steps[0].stepType)
        
        assertEquals(ExecutionStepType.SWITCH_SIDE_REST, steps[1].stepType)
        
        assertEquals(Side.LEFT, steps[2].side)
        assertEquals(ExecutionStepType.STRETCH_HOLD, steps[2].stepType)
    }
}
```

---

## 4. Testes de Transação e Atomicidade no Room

```kotlin
@RunWith(AndroidJUnit4::class)
class RoomTransactionTest {

    private lateinit var db: FitnessDatabase

    @Before
    fun createDb() {
        val context = ApplicationProvider.getApplicationContext<Context>()
        db = Room.inMemoryDatabaseBuilder(context, FitnessDatabase::class.java).build()
    }

    @After
    fun closeDb() {
        db.close()
    }

    @Test
    fun failedImportRollsBackAllInsertedRecords() = runBlocking {
        try {
            db.withTransaction {
                db.exerciseDao().insert(ExerciseEntity(name = "Ex 1", ...))
                db.workoutTemplateDao().insert(WorkoutTemplateEntity(templateUuid = "uuid-1", ...))
                throw IllegalStateException("Simulação de falha de validação durante o parse")
            }
        } catch (e: Exception) {
            // Ignora erro esperado
        }

        assertEquals(0, db.exerciseDao().count())
        assertEquals(0, db.workoutTemplateDao().count())
    }
}
```
Com esse conjunto de testes, a integridade arquitetural do código é blindada contra regressões durante todo o ciclo de vida do projeto.
