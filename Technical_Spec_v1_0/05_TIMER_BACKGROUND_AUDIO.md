# TIMER, FOREGROUND SERVICE, ÁUDIO & HÁPTICOS
**Versão:** 1.0  
**Camada:** `platform/` (Android Framework & Hardware)  

---

## 1. Contrato do Motor de Tempo Monotônico (`TimerEngine`)

```kotlin
package com.example.fitnesstrackerpro.platform.timer

import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.SharedFlow
import com.example.fitnesstrackerpro.domain.timer.TimerState
import com.example.fitnesstrackerpro.domain.timer.TimerEvent

interface TimerEngine {
    val state: StateFlow<TimerState>
    val events: SharedFlow<TimerEvent>
    val remainingMs: StateFlow<Long>
    val elapsedMs: StateFlow<Long>

    fun startCountdown(durationMs: Long)
    fun startStopwatch()
    fun pause()
    fun resume()
    fun cancel()
}
```

---

## 2. Aritmética Temporal Monotônica (Zero Deriva)

Para eliminar desvios provocados por atrasos de agendamento de Coroutines ou alterações no relógio do sistema operacional (ajuste de fuso ou NTP):

```kotlin
class MonotonicTimerEngineImpl : TimerEngine {
    private var targetEndElapsedRealtime: Long = 0L
    private var pausedRemainingMs: Long = 0L
    private var isStopwatch: Boolean = false
    private var stopwatchStartElapsedRealtime: Long = 0L

    override fun startCountdown(durationMs: Long) {
        pausedRemainingMs = durationMs
        targetEndElapsedRealtime = SystemClock.elapsedRealtime() + durationMs
        // Dispara loop de emissão de eventos e atualização de visualização
    }

    override fun pause() {
        if (_state.value == TimerState.RUNNING) {
            pausedRemainingMs = maxOf(0L, targetEndElapsedRealtime - SystemClock.elapsedRealtime())
            _state.value = TimerState.PAUSED
        }
    }

    override fun resume() {
        if (_state.value == TimerState.PAUSED) {
            targetEndElapsedRealtime = SystemClock.elapsedRealtime() + pausedRemainingMs
            _state.value = TimerState.RUNNING
        }
    }
}
```

### Regras Mandatórias de Relógio:
- **`SystemClock.elapsedRealtime()`** é a **única** fonte autoritativa para calcular durações ativas, regressões ou paradas.
- **`System.currentTimeMillis()`** é estritamente proibido para medir intervalos de tempo. Ele serve exclusivamente como carimbo de calendário histórico para `startedAtEpochMs` e `loggedAtEpochMs`.

---

## 3. Eventos Temporais e a Regra da Guarda de 10 Segundos

```kotlin
sealed interface TimerEvent {
    data object Started : TimerEvent
    data object Paused : TimerEvent
    data object Resumed : TimerEvent
    data object Warning10Seconds : TimerEvent
    data class CountdownTick(val second: Int) : TimerEvent // 3, 2, 1
    data object Completed : TimerEvent
    data object Cancelled : TimerEvent
}
```

### Regra da Guarda do Alerta de 10 Segundos:
```kotlin
fun checkWarningEvents(remainingSeconds: Long, totalPhaseDurationSeconds: Long) {
    // Guarda de Duração Curta: Suprime TTS de 10s se a fase for <= 15s
    if (remainingSeconds == 10L && totalPhaseDurationSeconds > 15L && !warning10Emitted) {
        warning10Emitted = true
        emitEvent(TimerEvent.Warning10Seconds)
    }

    // Contagem regressiva acústica final (3, 2, 1) emite sempre
    if (remainingSeconds in 1L..3L && !countdownTicksEmitted.contains(remainingSeconds)) {
        countdownTicksEmitted.add(remainingSeconds)
        emitEvent(TimerEvent.CountdownTick(remainingSeconds.toInt()))
    }
}
```
**Justificativa de Engenharia:** Em treinos como o Tabata, onde o descanso dura apenas 10 segundos, emitir a frase em português *"Atenção: faltam 10 segundos"* consome cerca de 3 segundos de áudio, colidindo e sobrepondo-se à contagem regressiva crítica de 3-2-1 segundos e ao anúncio da próxima estação. Em fases $\le 15$ segundos, o sintetizador vocal permanece em silêncio e o atleta recebe apenas os bips sonoros e vibrações de 3, 2 e 1.

---

## 4. `WorkoutForegroundService` como Detentor Canônico de Estado

O serviço de primeiro plano (`WorkoutForegroundService`) reside na camada `platform/service/` e atua como o **guardião do ciclo de vida temporal ativo**:

```kotlin
class WorkoutForegroundService : Service() {

    private val binder = LocalBinder()
    val timerEngine: TimerEngine = MonotonicTimerEngineImpl()

    inner class LocalBinder : Binder() {
        fun getService(): WorkoutForegroundService = this@WorkoutForegroundService
    }

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        val notification = buildWorkoutNotification()
        startForeground(NOTIFICATION_ID, notification)
        return START_STICKY
    }

    // Gerencia botões de ação disparados diretamente da notificação persistente
    fun onNotificationAction(action: String) {
        when (action) {
            ACTION_PAUSE -> timerEngine.pause()
            ACTION_RESUME -> timerEngine.resume()
            ACTION_SKIP -> dispatchSkip()
            ACTION_FINISH -> dispatchFinish()
        }
    }
}
```

### Benefícios dessa Arquitetura:
1. **Imunidade à Recreação de Telas:** A Activity pode ser destruída e recriada pelo Android (rotação de tela, mudança para modo escuro ou pressão de memória) sem que o relógio ou o áudio percam uma fração de segundo.
2. **Operação com Tela Bloqueada:** O serviço mantém o processo ativo com notificação interativa contínua e prioridade de execução no Android, exibindo a contagem e os controles na tela de bloqueio.

---

## 5. Gerenciamento de Audio Focus e Síntese Vocal (TTS)

```kotlin
class AudioFocusController(private val context: Context) {

    private val audioManager = context.getSystemService(Context.AUDIO_SERVICE) as AudioManager

    fun requestFocus(onLossTransient: () -> Unit, onGain: () -> Unit) {
        val request = AudioFocusRequest.Builder(AudioManager.AUDIOFOCUS_GAIN_TRANSIENT_MAY_DUCK)
            .setAudioAttributes(
                AudioAttributes.Builder()
                    .setUsage(AudioAttributes.USAGE_ASSISTANCE_SONIFICATION)
                    .setContentType(AudioAttributes.CONTENT_TYPE_SPEECH)
                    .build()
            )
            .setOnAudioFocusChangeListener { focusChange ->
                when (focusChange) {
                    AudioManager.AUDIOFOCUS_LOSS_TRANSIENT_CAN_DUCK -> {
                        // Música do Spotify/YouTube abaixa de volume; mantém TTS ativo
                    }
                    AudioManager.AUDIOFOCUS_LOSS_TRANSIENT -> {
                        // Outro app tomou o áudio exclusivamente (ex: mensagem de voz); pausa preventiva
                        onLossTransient()
                    }
                    AudioManager.AUDIOFOCUS_GAIN -> {
                        onGain()
                    }
                }
            }
            .build()
        audioManager.requestAudioFocus(request)
    }
}
```

### Políticas de Áudio:
1. **Sem Permissão Intrusiva:** O aplicativo não requer `READ_PHONE_STATE`. A interrupção por ligações ou alarmes é tratada diretamente pelos callbacks padrão do `AudioManager`.
2. **Voz pt-BR Operacional:** O `TextToSpeech` nativo é configurado com `Locale("pt", "BR")` para anunciar nomes de exercícios, troca de estações e alertas de tempo.
3. **Independência de Ajustes:** O usuário pode configurar no perfil se deseja Som (ToneGenerator), Voz (TTS) e Háptico (Vibração) de forma 100% independente (ex: treinar ouvindo podcast somente com vibração e bips, sem voz).
