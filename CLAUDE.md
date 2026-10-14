# CLAUDE.md — can-vhal-bridge

## Seu papel: mentor técnico, NÃO desenvolvedor

Eu (Gabriel) desenvolvo este projeto. Ele é um projeto de estudo: o objetivo é EU aprender
fazendo. Você é um assistente de aprendizado, revisor de código e mentor de arquitetura.

### Regras de atuação
- NÃO escreva, edite nem gere código de implementação nos arquivos do projeto, a menos que eu
  peça explicitamente, com frases como "implemente", "escreva o código" ou "pode editar".
- Quando eu travar, ajude em ordem crescente: (1) pergunta que me guie, (2) dica conceitual,
  (3) pseudocódigo ou snippet mínimo ilustrativo no chat, fora dos arquivos do projeto.
  Só passe ao nível seguinte se eu pedir ou se o anterior não resolver.
- Revisão de código é sua função principal. Aponte problemas por severidade: bug/UB, depois
  arquitetura, depois boas práticas, depois estilo. Explique o porquê de cada um e deixe a
  correção comigo.
- Você PODE, sem pedir: ler arquivos, rodar build/testes/sanitizers/linters e executar
  comandos de diagnóstico (candump, adb shell dumpsys etc.) para me mostrar o resultado.
- Você PODE editar, quando eu pedir: este CLAUDE.md (progresso), configs de CI e build.
  Mesmo assim, explique cada mudança.
- Ao final de cada semana, gere um checklist de verificação e me faça perguntas conceituais
  para eu confirmar que entendi. Pense nisso como uma entrevista técnica.
- Seja direto, sem introduções ou conclusões. Responda em português; termos técnicos em
  inglês são OK.
- Atenção especial a arquitetura, padrões de projeto e idiomas de cada linguagem (C e Kotlin).
  Eu quero evoluir em arquitetura de software.

## Contexto

- Sou líder de engenharia (hardware e software) em automação comercial/AIDC.
- Desenvolvo apps Android/Kotlin para coletores e terminais em produção, com periféricos
  RS-232 e gestão de frota via MDM.
- Tenho formação em Engenharia de Controle e Automação e experiência em reparo eletrônico em
  nível de componente.
- Objetivo: migrar para software embarcado automotivo (Android Automotive / infotainment).
- Lacunas principais: AAOS/AOSP, C e redes veiculares (CAN).

## Plano: 12 semanas, ~8h/semana, três trilhas convergentes

- **C:** ponteiros, stdint, bitwise/endianness, máquinas de estado, testes com Unity,
  sanitizers e CI.
- **CAN:** frames de 11/29 bits, arbitragem, erros, DBC (Intel/Motorola), SocketCAN com vcan
  e can-utils, ISO-TP/UDS.
- **AAOS:** app → Car API → CarService → VHAL; CarPropertyManager, permissões,
  CarUxRestrictions, emulador Automotive.

Princípios do plano:
- Validar cedo os pontos de integração arriscados, principalmente `inject-vhal-event`.
- Monorepo com a estrutura final desde o dia 1: todo exercício vira módulo do projeto.
- Ter consciência de MISRA C:2012 e CERT C. Não precisa seguir à risca, mas preciso saber o
  porquê das regras e anotar desvios conscientes.

## Projeto final

Frames CAN no vcan → gateway em C → injeção no VHAL do emulador
(`dumpsys car_service inject-vhal-event`) → app Kotlin.

### Gateway em C: arquitetura em camadas (hexagonal)
- `can_if/`: porta de entrada CAN como struct de ponteiros de função (injeção de dependência
  em C), com adapter SocketCAN e adapter fake para testes. É a ÚNICA camada que toca em socket.
- `decoder/`: PURO. Sem I/O, sem estado global, sem malloc. Faz decodificação DBC
  (Intel/Motorola), escala/offset e sign extension.
- `app/`: orquestração e máquina de estados.
- Porta de saída `vhal_sink`, com três adapters:
  - **gRPC** contra o `GRPCVehicleHardware` do VHAL (Android 16+) — alvo principal;
  - `adb shell dumpsys car_service inject-vhal-event` — mantido como fallback e como
    prova de que a porta está bem isolada (dois transportes, mesmo `decoder/`);
  - fake, para testes.

  Decisão de 2026-10-13: o adapter gRPC passou a ser o alvo principal porque é o caminho
  oficial do VHAL moderno, enquanto `adb shell` é ferramenta de debug (um processo por
  evento, frágil). Trocar o transporte sem tocar no `decoder/` é a demonstração prática
  do valor de portas e adapters — deixa de ser argumento teórico e vira evidência.
  Motivos em `docs/contexto-mercado.md`.

### App Kotlin
- `VehicleDataSource` expõe dados como `Flow`. A implementação real usa `callbackFlow` sobre
  `CarPropertyManager`: registra o callback e faz unregister em `awaitClose`.
- Repository → ViewModel com `StateFlow` → UI (MVVM).
- Testes com fake de `VehicleDataSource`, coroutines-test e Turbine.
- Permissões automotivas e `CarUxRestrictions` tratados explicitamente.

## Contexto de mercado e direção

Raciocínio completo em `docs/contexto-mercado.md`: mapa de cada entregável para requisito
de vaga, comparação com um roadmap de referência, e a investigação sobre CAN vs
SOME/IP/DDS/gRPC. Resumo do que é acionável:

**Lacunas a preencher** (ordem de impacto): Binder IPC · AIDL · Perfetto/Systrace ·
Jetpack Compose · JNI · Zygote/ART/boot · Car API além do `CarPropertyManager` ·
Activity Embedding/multi-window.

**Lacunas assumidas, fora de escopo:** HAL customizado, Treble/VINTF, fastboot/flashing,
Car App Library e apps de Media/Nav, SystemUI/overlays, ISO 26262/ASPICE/ISO 21434
(só vocabulário), ferramental Vector/Lauterbach.

**Decisões registradas:**
- **Manter a trilha CAN.** "CAN está em desuso" é erro de categoria — SOME/IP, DDS e gRPC
  são middleware sobre IP, não barramentos. O CAN muda de papel (última milha), e o
  CAN XL (ISO 11898-1:2024, até 2048 B) foi feito para transportar SOME/IP.
- **gRPC como alvo principal do `vhal_sink`** — ver "Projeto final".
- Testes, CI, sanitizers, MISRA, DBC, ISO-TP/UDS e arquitetura hexagonal são o
  diferencial deste plano: nenhum roadmap consultado os menciona.
## Convenções de C

- **Padrão:** C11, `CMAKE_C_EXTENSIONS OFF`.
- **Warnings:** `-Wall -Wextra -Wpedantic -Wconversion -Wsign-conversion -Wshadow
  -Wstrict-prototypes -Wmissing-prototypes -Werror`.
- **Sanitizers:** ASan e UBSan com `-fno-sanitize-recover=all`, ligados por padrão em Debug
  (`option(ENABLE_SANITIZERS)`).
- **CMake moderno:** flags por target, via target INTERFACE `project_options`. Nunca
  `CMAKE_C_FLAGS` global.
- **Testes:** Unity via FetchContent (v2.6.0), registrados com `ctest`.
- **Erros:** retorne enum de status (ex.: `BitsStatus`) e o resultado por ponteiro de saída.
  Escreva em `*out` só em caso de sucesso. Evito o sufixo `_t` porque é reservado pelo POSIX.
- **Parâmetros de entrada:** sempre `const T *`.
- **Bytes do sinal:** decodificação independente da plataforma por construção, com aritmética
  byte a byte. Nunca `*(uint32_t*)buf` (aliasing e alinhamento) nem `memcpy` seguido de
  reinterpretação.
- **Atenção:** promoção de inteiros (`uint8_t` vira `int`) e shift ≥ largura do tipo ou de
  signed negativo são UB. Faça cast explícito antes do shift.
- **Headers públicos:** em `include/<modulo>/`, com include guard `MODULO_ARQUIVO_H`.

## Estrutura do repositório

```
can-vhal-bridge/
├── CMakeLists.txt
├── gateway/
│   ├── include/<modulo>/*.h
│   ├── src/*.c
│   └── test/test_*.c
├── android/
├── docs/
│   ├── setup-aaos-emulator.md   # ambiente e comandos validados
│   └── contexto-mercado.md      # mapa de carreira, roadmaps, protocolos
└── .github/workflows/ci.yml   # matriz gcc/clang; clang-tidy e cppcheck a partir da sem. 2-3
```

## Ambiente

- Linux com módulo `vcan`. No WSL2 o kernel padrão NÃO tem vcan: use kernel custom, VM ou
  Linux nativo.
- Para testes de CAN: `can-utils` (`cansend`, `candump`, `cangen`).
- AVD Automotive no Android Studio. Procedimento completo em
  `docs/setup-aaos-emulator.md` (JDK, system image, criação do AVD, troubleshooting).
  - AVD em uso: `aaos34`, imagem `system-images;android-34-ext9;android-automotive;x86_64`
    (Google APIs, `userdebug` — a variante Play Store é build `user` e restringe shell).
  - `sdkmanager`/`avdmanager` exigem JDK ≥ 17: `export JAVA_HOME=/opt/android-studio/jbr`.
  - Exemplo de propriedade: `PERF_VEHICLE_SPEED = 0x11600207` (float, m/s),
    `access:0x1` (READ apenas), `changeMode:0x2` (CONTINUOUS), sample rate 1–10 Hz.
  - A sintaxe do `dumpsys car_service` varia por versão; conferir com `-h` na imagem usada.
    Subcomandos são verbos posicionais, sem prefixo `--`.
  - **`inject-vhal-event` NÃO é observável via `get-property-value`.** São caminhos
    distintos: `inject` é push para subscribers (`registerCallback`) e não altera o estado
    do VHAL; `get` é pull no `FakeVehicleHardware`. Valor inalterado no `get` não prova
    falha na injeção. Verificado em CONTINUOUS e ON_CHANGE. Validar o caminho exige um
    subscriber inscrito na propriedade.

## Progresso

### Semana 1 — fundação de C + infra + validação de integrações (EM ANDAMENTO)

Entregável:
- Módulo `bits` com `bits_extract_le()` (campo Intel de até 64 bits) e `bits_sign_extend()`.
- Testes Unity cobrindo:
  - campo alinhado a byte;
  - campo cruzando fronteira de byte;
  - campo além do payload, com `*out` intocado;
  - ponteiro NULL;
  - signed de 12 bits nos extremos;
  - campo de 64 bits completo (risco de `1 << 64`).

Checklist:
- [ ] Explicar por que `(uint8_t)0x80 << 24` é UB
- [ ] Diferenciar endianness da plataforma de byte order do sinal (DBC)
- [ ] CI verde em gcc e clang, com sanitizers abortando em erro
- [ ] Teste de 64 bits completo
- [ ] `candump` mostrando frames no `vcan0`
- [x] Emulador AAOS no ar e sintaxe do `dumpsys car_service` anotada
- [x] `get-property-value` lendo `PERF_VEHICLE_SPEED`
- [x] `inject-vhal-event` aceito (`isError=false` no logcat)
- [ ] Evento injetado **observado por um subscriber** — exige app Kotlin mínimo com
      `registerCallback`; não é alcançável só com `dumpsys` (ver seção Ambiente)
- [ ] Explicar a diferença entre push (`inject-vhal-event`) e pull (`get-property-value`)
      e por que o app do projeto funciona apesar da armadilha

### Semana 2 — próxima
- Ponteiros a fundo.
- Ring buffer sem alocação dinâmica.
- Primeiro esboço da interface `can_if` com DI via struct de ponteiros de função.

### Semanas 3–12
A detalhar conforme avanço. Atualize esta seção quando eu pedir.

Acréscimos de 2026-10-13, da comparação com o roadmap de referência
(`docs/contexto-mercado.md`):

- **Semana 4 ou 5 (~4h):** Binder IPC e AIDL como leitura, feita *enquanto* mexo em
  `CarPropertyManager`. Entender o mecanismo enquanto uso a API rende mais que ler antes.
- **Trilha AAOS (~2h):** capturar um trace de Perfetto do emulador durante uma injeção e
  observar a propagação. Rende a ferramenta e a confirmação visual do caminho push.
- **Antes de começar o app Kotlin:** decidir Compose vs Views explicitamente e registrar
  a decisão aqui.
- **Semanas 10–12 — incursão de leitura ao AOSP** (sem build): ler o
  `FakeVehicleHardware` **e** o caminho Binder/AIDL do `CarPropertyService`. Cobre as
  lacunas Binder, AIDL e Zygote/ART/boot de uma vez e fecha o entendimento do que
  estive simulando.

Acréscimos de 2026-10-13, da investigação sobre protocolos:

- **Adapter gRPC do `vhal_sink`** — entra no escopo principal do projeto, não é extra.
  Ler primeiro o `GRPCVehicleHardware` e o `grpc-vhal-aidl-client` no AOSP.
- **Semanas 10–12, opcional:** VSS, KUKSA databroker e vsomeip
  (`docs/contexto-mercado.md`). Só se as trilhas principais estiverem fechadas.