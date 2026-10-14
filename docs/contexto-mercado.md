# Contexto de mercado e decisões de direção

Material de consulta, extraído do `CLAUDE.md` para mantê-lo enxuto. Não é instrução de
trabalho: é o raciocínio por trás das escolhas do plano, para eu não precisar
reconstruí-lo nem repetir a pesquisa.

Índice:
- Mapa para o mercado: o que cada entregável prova
- Comparação com roadmap de referência
- Cenário de protocolos: CAN está em desuso?

---

## Mapa para o mercado: o que cada entregável prova

Serve para dois usos: priorizar o estudo quando o tempo apertar e ter resposta pronta ao
falar do projeto em entrevista. A coluna da direita é o requisito que aparece em vaga.

### Trilha C

| Entregável | Requisito de vaga que cobre |
|---|---|
| `bits_extract_le` / `bits_sign_extend` com testes de borda | manipulação bitwise, `stdint`, consciência de UB |
| Warnings como erro + ASan/UBSan + CI em gcc e clang | rigor de toolchain; base do que MISRA formaliza |
| Desvios de MISRA anotados com justificativa | ASPICE: decisão técnica documentada e rastreável |
| Ring buffer sem `malloc`, stack com bound conhecido | determinismo; proibição de alocação dinâmica em runtime |
| Máquina de estados explícita | padrão dominante em firmware automotivo |
| Testes com Unity rodando em `ctest` | teste unitário de código embarcado sem hardware |

### Trilha CAN

| Entregável | Requisito de vaga que cobre |
|---|---|
| Frames 11/29 bits, arbitragem, frames de erro | fundamento de CAN exigido em toda vaga do setor |
| Decodificação DBC Intel e Motorola, escala/offset | ler especificação de sinal de OEM — trabalho diário |
| SocketCAN + `vcan` + can-utils | substituto de CANoe/CANalyzer; dizer isso explicitamente |
| ISO-TP (ISO 15765) e UDS (ISO 14229) | item muito citado em vaga e que pouca gente domina |
| Timeout de frame, sinal obsoleto, contador de erro | **o filtro nº 1 em entrevista**: o que acontece quando o sinal falha |

### Trilha AAOS

| Entregável | Requisito de vaga que cobre |
|---|---|
| Cadeia app → Car API → CarService → VHAL entendida | arquitetura AAOS de ponta a ponta |
| `CarPropertyManager` com `registerCallback` | API central de IVI em Android Automotive |
| Property IDs, `areaId`, `access`, `changeMode`, sample rate | semântica do VHAL, não só chamada de API |
| Permissões automotivas e `CarUxRestrictions` | requisito regulatório de distração do motorista |
| `callbackFlow` com `awaitClose`, `StateFlow`, MVVM | Kotlin moderno; diferencia de quem só sabe C |

### Arquitetura (o diferencial transversal)

| Entregável | Requisito de vaga que cobre |
|---|---|
| `can_if` como struct de ponteiros de função | DI em C; trocar hardware sem tocar na lógica |
| `decoder/` puro, sem I/O nem estado global | testabilidade sem bancada; separação de responsabilidades |
| Adapters fake para `can_if` e `vhal_sink` | SIL testing — como o setor testa sem veículo |
| Porta de saída `vhal_sink` isolada | variant management: mesma lógica, saídas diferentes |

### Lacunas que este projeto NÃO cobre

Reconhecer é melhor que ignorar. Nenhuma exige certificação — exige vocabulário.

- **ISO 26262 / ASPICE:** saber o que é ASIL, requisito rastreado, FMEA. Infotainment
  costuma ser QM ou ASIL-A, então basta nível de conversa (~10h de leitura).
- **ISO 21434 (cybersecurity):** cada vez mais obrigatório.
- **AUTOSAR Classic e Adaptive:** pouco relevante para infotainment, central fora dele.
- **Ferramental proprietário:** Vector (CANoe, CANape), Lauterbach TRACE32, dSPACE. Sem
  licença acessível; assumir e apontar o equivalente aberto que usei.
- **AOSP build real:** Soong, SELinux, HAL/AIDL. Ver incursão de leitura nas semanas 10–12.
- **Inglês técnico de reunião:** projeto automotivo é global; filtro frequente.

### Diferenciais do meu perfil a usar ativamente

- **Hardware em nível de componente:** raro entre devs de software. Camada física de CAN,
  terminação de 120 Ω, ruído, sinal diferencial.
- **Produto em campo com periférico serial:** RS-232, coletores, frota via MDM. Mesmo
  problema conceitual de ECU: ambiente hostil, acesso difícil, atualização de risco.
- **Liderança de engenharia:** abre posição sênior/tech lead em vez de pleno; escasso em
  consultoria automotiva.

## Comparação com roadmap de referência

Confrontado em 2026-10-13 com o "Android Automotive Developer Roadmap" de Anand Gaur
(Medium, nov/2025, 6 fases). Conclusão: documentos de naturezas diferentes, não
concorrentes. Mantenho meu plano; uso o roadmap como mapa de cobertura.

| | Roadmap de referência | Meu plano |
|---|---|---|
| Pergunta que responde | o que estudar | o que construir |
| Organização | 6 fases sequenciais | 3 trilhas paralelas convergentes |
| Prazo e entregável | nenhum | 12 semanas; gateway + app + CI |
| Critério de conclusão | nenhum | checklist verificável por semana |
| Força | amplitude | executabilidade e portfólio |

### Lacunas do meu plano que o roadmap expôs (preencher)

Em ordem de impacto para o objetivo (infotainment/AAOS):

1. **Binder IPC** — a cadeia app → CarService → VHAL do meu plano *é* Binder. Usar
   `CarPropertyManager` sem saber o que roda por baixo é pergunta de entrevista quase
   garantida. Lacuna mais séria.
2. **AIDL** — linguagem de declaração das interfaces entre app, system service e HAL.
   Em AAOS moderno o VHAL é AIDL. Complementa o Binder (mecanismo vs declaração).
3. **Perfetto / Systrace** — profiling de sistema. Tenho ASan/UBSan (correção), não
   tenho nada de performance. Infotainment tem requisito duro de boot time e jank.
4. **Jetpack Compose** — meu plano diz "UI (MVVM)" sem fixar o toolkit; Compose é o
   padrão atual, e Compose + ViewModel + Flow é exatamente a arquitetura que descrevi.
5. **JNI** — meu gateway fala com o Android por `adb shell`, ponte frágil e externa.
   JNI é como se faria de dentro. Saber por que NÃO escolhi é resposta forte.
6. **Zygote, ART, boot/init** — vocabulário de plataforma, barato por leitura.
7. **Car API além do `CarPropertyManager`** — `CarInfoManager`, `CarSensorManager`,
   `CarAppFocusManager`.
8. **Activity Embedding / multi-window** — AAOS tem telas grandes, ultrawide e
   múltiplos displays (central, cluster, passageiro). Coletor é single-window: lacuna real.

### O que o roadmap cobre e eu ignoro por decisão

Não é esquecimento; é orçamento de 96h e alvo em infotainment.

- **Escrever HAL customizado, Treble, VINTF, vendor partition** — trabalho de
  plataforma/BSP de OEM, exige build do AOSP.
- **Fastboot / flashing** — precisa de hardware real.
- **Car App Library, CarContext, Template APIs, apps de Media/Nav/POI, voz** — é
  **caminho de carreira distinto**: app automotivo de terceiros com API restrita e
  templates, não plataforma/integração. Saber que existe, escolher consciente.
- **SystemUI, launchers, overlays de branding** — OEM customization.
- **Fase 1 do roadmap** (fundamentos de app Android) — já domino, exceto os itens 4 e 8
  da lista de lacunas acima.

### O que o roadmap valida no meu plano

- `CarPropertyManager` aparece como system service principal e como item central do
  Car API — meu eixo está certo.
- "Work with Vehicle HAL" e "Understand CAN Bus / ECU communication" aparecem juntos:
  é exatamente a junção que meu projeto final faz.
- C/C++ e Kotlin coexistindo em camadas diferentes é a premissa das minhas duas trilhas.
- O roadmap lista "Implement new CarProperty values" — o caminho "de verdade" que o
  `inject-vhal-event` contorna, e que exigiria build do AOSP. Confirma que minha
  injeção é o atalho correto para o orçamento, **e** que é um atalho.

### O que meu plano tem e o roadmap não menciona

Registro porque inverte a avaliação: testes (unitário, Unity, JUnit, Turbine), CI,
sanitizers, MISRA C:2012 / CERT C, arquitetura de software (hexagonal, DI,
portas/adapters), DBC, ISO-TP / UDS, SocketCAN / vcan / can-utils, bitwise /
endianness / sign extension, e qualquer projeto prático.

O roadmap descreve como ser um desenvolvedor Android que trabalha em carro. Não
descreve como ser um engenheiro de software embarcado — que é o objetivo declarado.

## Cenário de protocolos: CAN está em desuso?

Investigado em 2026-10-13, a partir da afirmação corrente de que SOME/IP, DDS e gRPC
estão substituindo o CAN.

**Veredito: a premissa é um erro de categoria.** SOME/IP, DDS e gRPC não são barramentos,
são middleware de comunicação sobre IP. Comparar CAN com SOME/IP é comparar RS-485 com
HTTP. O que muda de verdade é a arquitetura E/E do veículo, e nela o CAN muda de papel:
deixa de ser barramento principal e passa a ser a última milha.

Não é sucessão, é estratificação:

```
sensor/atuador simples ──► CAN, CAN-FD, LIN  (CAN XL entrando)
                              │
                    zonal controller (agrega e tunela)
                              │
backbone ──────────────► Automotive Ethernet + TSN
                              │
camada de serviço ─────► SOME/IP (AUTOSAR) | DDS (ADAS/AD) | gRPC (HPC, cloud)
```

### Evidência contra "CAN em desuso"

- **ISO 11898-1:2024** (maio/2024) padroniza o **CAN XL**: até **2048 bytes** de dados
  (8 no clássico, 64 no FD) e ~20 Mbit/s. Define opção de implementação para nós que
  suportam clássico + FD + XL simultaneamente.
- ISO 11898-2 com o transceptor CAN SIC XL entrou em balotagem em 2025; controladores e
  transceptores já existem.
- **O CAN XL foi projetado para transportar tráfego IP, incluindo SOME/IP.** Não é
  concorrente da arquitetura orientada a serviços — é transporte para ela onde Ethernet
  sai caro.

Norma nova + plano de conformidade + silício novo não é protocolo morrendo.

### Por que o CAN não sai (física e economia, não inércia)

| Fator | CAN | Automotive Ethernet |
|---|---|---|
| Custo por nó | par trançado, controlador já no MCU | PHY dedicado + porta de switch |
| Topologia | barramento multiponto, sem switch | ponto a ponto, exige switch |
| Arbitragem | por prioridade, no protocolo | precisa de TSN para determinismo |
| Nó trivial (2 bytes/100 ms) | adequado | superdimensionado |

Multiplicar PHY + porta de switch por dezenas de nós triviais e por milhões de unidades
não acontece. Some ciclo de plataforma de 7–10 anos e reposição por décadas.

### Os três protocolos

| | SOME/IP | DDS | gRPC |
|---|---|---|---|
| Origem | AUTOSAR (BMW pioneira) | OMG | Google / cloud |
| Modelo | serviços: RPC + pub/sub + fields | pub/sub data-centric | RPC + streaming |
| Descoberta | SOME/IP-SD | dinâmica (RTPS) | não tem; endpoint fixo |
| QoS | limitado | muito rico | nenhum |
| IDL / serialização | ARXML/FIBEX, própria | IDL do OMG | Protobuf |
| Determinismo | moderado | bom com QoS + TSN | nenhum |
| Domínio típico | ECU ↔ ECU, infotainment | ADAS/AD (ROS 2 usa DDS) | veículo ↔ cloud, HPC interno |

### Qualidade dos dados de adoção

IoT Analytics (SDV Report 2026): >90% dos OEMs comprometidos com arquitetura zonal, 80%
migrando. **Isso é compromisso e migração, não veículos em produção** — a produção real
ainda é fina. Não há fonte que quantifique share de SOME/IP vs DDS em produção; há
apenas evidência de que fornecedores entregam ambos (Neusoft embarca DDS+SOME/IP+MQTT
no mesmo domain controller; NXP usa DDS em zone controller).

Tratar "CAN está sendo substituído" como slogan de SDV repetido sem qualificação. A
transição é real; o prazo e a extensão são bem mais lentos que o discurso.

### Achado que afeta este projeto: gRPC já está no VHAL

A partir do **Android 16** existe o **`GRPCVehicleHardware`**, implementação de
referência de `IVehicleHardware` ao lado do `FakeVehicleHardware`. Permite que o VHAL
receba valores de propriedade de uma fonte gRPC externa. Em emuladores, um servidor gRPC
no device aceita injeção de um cliente que pode rodar **no host**. Existe cliente de
exemplo: `grpc-vhal-aidl-client`.

Consequência: o caminho oficial e moderno para o que o gateway faz é **gRPC**, não
`adb shell dumpsys`. Ver decisão registrada em "Projeto final".

### Por que a trilha CAN continua certa

1. Os conceitos transferem integralmente: bit packing, endianness, escala/offset, sign
   extension, timeout, dado obsoleto, contador de erro. Nada disso é específico de CAN.
2. **DBC não morre com o CAN** — especificação de sinal continua em DBC/ARXML/FIBEX
   mesmo com transporte Ethernet.
3. **Gateway CAN↔SOME/IP é um dos cargos mais comuns hoje** — a transição *cria* a vaga.
   Exige os dois lados; este projeto já é metade dele.
4. **UDS é o mesmo dos dois lados**: ISO 14229 sobre ISO-TP (CAN) e sobre DoIP (IP).
5. Vagas de infotainment/AAOS continuam listando CAN, porque o VHAL de produção lê do
   barramento.

**Não trocar a trilha CAN por SOME/IP/DDS.** Com 96h, CAN dá fundamentos de decodificação,
ferramenta gratuita (`vcan`), formato real (DBC) e diagnóstico (ISO-TP/UDS). SOME/IP sem
CAN dá middleware sem entender o dado que ele transporta.

### Exploração opcional (semanas 10–12)

- **VSS (COVESA Vehicle Signal Specification)** — modelo de sinal agnóstico de transporte;
  o vocabulário que a indústria adota para desacoplar sinal de barramento. Leitura curta.
- **KUKSA databroker (Eclipse/COVESA)** — databroker que segue VSS, em Rust, recebe de CAN
  e SOME/IP e expõe por gRPC. Tem CAN feeder pronto: daria para alimentar com o `vcan0`.
  Ressalvas: licença aparece como Apache 2.0 (material 2022) e MPL 2.0 (material 2024) —
  conferir o repositório; feeder de SOME/IP anunciado como "em breve" em 2022, sem
  confirmação de release.
- **vsomeip (COVESA)** — implementação de SOME/IP contribuída pela BMW, mantida com a
  Critical Software. Roda em Linux; existe no AOSP em `external/sdv/vsomeip`. É a via para
  SOME/IP prático sem licença Vector.
