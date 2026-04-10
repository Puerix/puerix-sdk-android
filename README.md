<h1 align="center">PuerixSDK — Android</h1>

<p align="center">
  <img src="https://img.shields.io/badge/platform-Android%20API%2021%2B-blue" />
  <img src="https://img.shields.io/badge/kotlin-1.9-purple" />
  <img src="https://img.shields.io/badge/version-0.1.0-green" />
  <img src="https://img.shields.io/badge/license-Proprietary-red" />
</p>

<p align="center">
  SDK nativo Android para verificação de idade com detecção de vida (liveness) e captura de documentos.
</p>

---

## Funcionalidades

- **Liveness Detection** — Verificação facial com rastreamento de rosto e movimentos da cabeça
- **Captura de Documento** — Frente e verso com guia visual e verificação de qualidade
- **OCR** — Extração automática do CPF via ML Kit Text Recognition
- **Integração com API** — Sessão, upload de frames, validação de documento
- **UI nativa** — Telas programáticas com branding Puerix (sem XML)

---

## Instalação

### 1. Adicionar o repositório Maven

```gradle
// settings.gradle ou root build.gradle
allprojects {
    repositories {
        google()
        mavenCentral()
        maven { url 'https://raw.githubusercontent.com/Puerix/puerix-sdk-android/main/maven-repo' }
    }
}
```

### 2. Adicionar a dependência

```gradle
// app/build.gradle
dependencies {
    implementation 'com.puerix:puerix-sdk:0.1.0'
}
```

### 3. Sincronizar o projeto

```bash
./gradlew sync
```

> **Nota:** Se estiver usando Kotlin 2.0+, adicione ao `app/build.gradle`:
> ```gradle
> kotlinOptions {
>     freeCompilerArgs += ['-Xskip-metadata-version-check']
> }
> ```

---

## Uso rápido

### 1. Inicializar o SDK

Chame uma vez, idealmente no `Application.onCreate()`:

```kotlin
import com.puerix.puerix_sdk.PuerixSDK
import com.puerix.puerix_sdk.PuerixConfig
import com.puerix.puerix_sdk.PuerixEnvironment

PuerixSDK.initialize(PuerixConfig(
    apiKey = "SUA_API_KEY",
    environment = PuerixEnvironment.PRODUCTION,  // ou DEVELOPMENT
    enableLogging = true                          // false em produção
))
```

### 2. Verificação completa (recomendado)

Inicia o fluxo completo: sessão → liveness → upload → documento (se necessário) → resultado.

```kotlin
class MainActivity : AppCompatActivity() {

    companion object {
        private const val RC_VERIFICATION = 1234
    }

    private fun startVerification() {
        PuerixSDK.startVerification(
            activity = this,
            requestCode = RC_VERIFICATION,
            subject = "user-123",
            ageLimit = 18,
        )
    }

    override fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?) {
        super.onActivityResult(requestCode, resultCode, data)
        PuerixSDK.handleVerificationResult(
            requestCode, resultCode, data,
            myRequestCode = RC_VERIFICATION
        ) { result ->
            if (result.isApproved) {
                Log.d("Puerix", "Aprovado! Session: ${result.sessionId}")
            } else {
                Log.d("Puerix", "Não aprovado: ${result.status}")
                result.errorMessage?.let { Log.e("Puerix", "Erro: $it") }
            }
        }
    }
}
```

### 3. Apenas liveness (sem API)

Se quiser usar só a detecção facial sem integração com a API:

```kotlin
companion object {
    private const val RC_LIVENESS = 5678
}

// Iniciar
PuerixSDK.startLiveness(
    activity = this,
    requestCode = RC_LIVENESS,
)

// Receber resultado
override fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?) {
    super.onActivityResult(requestCode, resultCode, data)
    PuerixSDK.handleLivenessResult(
        requestCode, resultCode, data,
        myRequestCode = RC_LIVENESS
    ) { result ->
        if (result.isComplete) {
            Log.d("Puerix", "Capturas: ${result.captureData.keys}")
            // result.captureData["lookAtCamera"] → ByteArray (JPEG)
        }
    }
}
```

---

## Referência da API

### PuerixConfig

| Parâmetro | Tipo | Padrão | Descrição |
|-----------|------|--------|-----------|
| `apiKey` | `String` | — | Chave de API (obrigatório) |
| `environment` | `PRODUCTION` / `DEVELOPMENT` | `PRODUCTION` | Ambiente da API |
| `baseUrl` | `String?` | `null` | URL customizada (usa o padrão do environment) |
| `timeoutMs` | `Long` | `30000` | Timeout de rede em milissegundos |
| `enableLogging` | `Boolean` | `false` | Habilita logs no Logcat |

### startVerification

```kotlin
fun startVerification(
    activity: Activity,
    requestCode: Int,
    subject: String,                    // Identificador do usuário
    ageLimit: Int = 18,                 // Idade mínima (10-21)
    steps: List<PuerixLivenessStep>,    // Passos do liveness
    stepDuration: Long = 10_000L,       // Duração por passo (ms)
)
```

### PuerixVerificationResult

| Propriedade | Tipo | Descrição |
|-------------|------|-----------|
| `sessionId` | `String` | ID da sessão no backend |
| `status` | `String` | `approved`, `denied`, `requires_doc`, `cancelled` |
| `isApproved` | `Boolean` | Se a verificação foi aprovada |
| `errorMessage` | `String?` | Mensagem de erro (se houver) |

### PuerixLivenessStep

| Step | Key | Descrição |
|------|-----|-----------|
| `LOOK_AT_CAMERA` | `lookAtCamera` | Olhar para a câmera |
| `TURN_HEAD_LEFT` | `turnHeadLeft` | Virar a cabeça para a esquerda |
| `TURN_HEAD_RIGHT` | `turnHeadRight` | Virar a cabeça para a direita |

### PuerixLivenessResult

| Propriedade | Tipo | Descrição |
|-------------|------|-----------|
| `isComplete` | `Boolean` | Se completou todos os passos |
| `isApproved` | `Boolean` | Se o backend aprovou |
| `captureData` | `Map<String, ByteArray>` | JPEG data por step key |

---

## Fluxo de verificação

```
┌─────────────┐     ┌──────────┐     ┌──────────────┐     ┌──────────────┐
│  Iniciar    │────>│ Liveness │────>│   Upload     │────>│  Resultado   │
│ Verificação │     │ (3 steps)│     │   Frames     │     │  approved/   │
└─────────────┘     └──────────┘     └──────┬───────┘     │  denied      │
                                            │              └──────────────┘
                                            │ requires_doc
                                            v
                                     ┌──────────────┐     ┌──────────────┐
                                     │  Documento   │────>│  Validação   │
                                     │ (frente+verso│     │  CPF + foto  │
                                     │  + OCR CPF)  │     └──────────────┘
                                     └──────────────┘
```

---

## Permissões

O SDK declara as permissões automaticamente no `AndroidManifest.xml`:

```xml
<uses-permission android:name="android.permission.CAMERA" />
<uses-permission android:name="android.permission.INTERNET" />
```

A permissão de câmera é solicitada em runtime automaticamente pelo SDK.

---

## Troubleshooting

| Erro | Causa | Solução |
|------|-------|---------|
| `401 Unauthorized` | API key inválida | Verifique a chave no painel Puerix |
| `403 Forbidden` | Limite atingido ou conta bloqueada | Verifique seu plano |
| `Incompatible classes in dependencies` | Versão Kotlin incompatível | Adicione `-Xskip-metadata-version-check` |
| `Session token não disponível` | `startVerification` sem `initialize` | Chame `initialize()` primeiro |
| Camera permission denied | Usuário negou acesso | O SDK solicita automaticamente |

---

## Requisitos

- Android API 21+ (Android 5.0)
- Kotlin 1.8+
- Java 17
- API key — solicite em [puerix.com](https://puerix.com)

---

## Changelog

### 0.1.0
- Liveness detection (3 steps: olhar, virar esquerda, virar direita)
- Captura de documento com OCR (CPF)
- Verificação de qualidade de imagem
- Integração completa com API Puerix
- Distribuição via AAR (closed-source)

---

## Licença

Copyright (c) 2026 Puerix. Todos os direitos reservados. Licença proprietária.
