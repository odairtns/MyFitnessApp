# BUILD, CONFIGURAÇÃO, MIGRAÇÕES & ENTREGA
**Versão:** 1.0  
**Status:** Critérios de Aceite para Release de Produção  

---

## 1. Variantes de Build e Configuração do Gradle

O projeto adota uma matriz de build simples e focada em qualidade local:

```kotlin
android {
    namespace = "com.example.fitnesstrackerpro"
    compileSdk = 35

    defaultConfig {
        applicationId = "com.example.fitnesstrackerpro"
        minSdk = 26
        targetSdk = 35
        versionCode = 1
        versionName = "1.0.0"

        testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"

        // Configuração do KSP para exportação do esquema do Room
        ksp {
            arg("room.schemaLocation", "$projectDir/schemas")
        }
    }

    buildTypes {
        release {
            isMinifyEnabled = true
            isShrinkResources = true
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
            signingConfig = signingConfigs.getByName("release")
        }
        debug {
            applicationIdSuffix = ".debug"
            isDebuggable = true
        }
    }
}
```

### Regras de ProGuard / R8:
- Preservar anotações e classes de domínio serializáveis (`@Serializable`).
- Preservar classes de entidade do Room (`@Entity`) e TypeConverters.
- Nenhuma dependência externa fechada ou telemetria em nuvem é adicionada.

---

## 2. Políticas de Migração do Banco de Dados Room

A integridade do histórico do atleta é o bem mais valioso do aplicativo.

### Regras Mandatórias de Persistência:
1. **`exportSchema = true` Obrigatório:** O diretório `app/schemas/` deve ser versionado no Git para documentar todas as transições de tabelas.
2. **Proibição de Destructive Migration:** O método `.fallbackToDestructiveMigration()` é **terminantemente proibido** na variante de release de produção.
3. **Migrações Explícitas com Testes Automatizados:** Qualquer alteração no schema exige:
   - Um objeto `val MIGRATION_X_Y = object : Migration(X, Y) { ... }` contendo os comandos SQL precisos.
   - Um teste de migração instrumentado utilizando `MigrationTestHelper` do Room, validando que os dados preexistentes permanecem intactos após a migração.

---

## 3. Estrutura de Logging e Observabilidade Local

Para proteger a privacidade do atleta e facilitar a depuração:
- Logs utilizam uma fachada própria (`AppLogger`).
- **Nenhum dado confidencial ou dump completo de treinos é impresso em logcat na variante de release**.
- Erros de runtime e falhas de recuperação são registrados internamente com mensagens estruturadas para auditoria de diagnóstico.

---

## 4. Portões de Liberação de Release (Release Gates)

Nenhuma versão do aplicativo pode ser gerada para distribuição sem satisfazer **100% dos seguintes critérios**:

| Portão | Descrição do Critério | Verificação |
| :--- | :--- | :--- |
| **Gate 1: Zero Perda de Dados** | Morte de processo no meio de um treino em andamento recupera o estado exato ao reabrir. | Teste de Process Death |
| **Gate 2: Imutabilidade de Template** | Nenhuma execução ou descarte altera a estrutura ou pesos do template original. | Teste de Unidade Automatizado |
| **Gate 3: Relógio Monotônico Real** | Timer em segundo plano com tela bloqueada não perde precisão nem pausa indevidamente. | Teste em Dispositivo Físico |
| **Gate 4: Áudio e Convivência Acústica** | Treino com fone Bluetooth e Spotify em segundo plano abaixa a música no aviso e retorna o volume suavemente (*ducking*). | Teste em Dispositivo Físico |
| **Gate 5: Unidade Canônica (kg)** | Todas as cargas são salvas em kg; entradas e saídas em lb convertem sem erro cumulativo de arredondamento. | Teste de Domínio Automatizado |
| **Gate 6: Atomicidade XML** | Tentativa de importar XML corrompido ou inválido não grava nenhuma linha órfã no banco Room. | Teste Transacional do Room |
| **Gate 7: Cancelamento vs Descarte** | Cancelamento salva sessão como `CANCELLED`; descarte executa `DELETE CASCADE` físico completo. | Teste DAO Instrumentado |

---

## 5. Congelamento Arquitetural (Architecture Freeze)

As seguintes premissas são definitivas e imutáveis para a V1:
- Arquitetura **100% Offline-First**, sem necessidade de login ou conectividade.
- O modelo relacional separa rigorosamente **Intenção (Template)** de **Fato (Session)**.
- O **`WorkoutForegroundService`** é o guardião exclusivo da temporização monotônica.
- O aplicativo é uma **ferramenta pessoal de alta precisão**, sem gamificação ruidosa, sem feed social e sem IA invasiva.

Qualquer alteração que viole essas diretrizes deve ser formalmente rejeitada ou submetida a uma nova decisão arquitetural (ADR) justificada tecnicamente.
