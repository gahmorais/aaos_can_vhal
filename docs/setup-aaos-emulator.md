# Setup do emulador Android Automotive (AAOS)

Procedimento para subir um AVD Automotive no Linux e validar o ponto de integração
`dumpsys car_service inject-vhal-event`, que é a saída do gateway em C.

Ambiente de referência: Pop!_OS, AMD Ryzen 5 5600X, 15 GiB RAM, SDK em `~/Android/Sdk`,
Android Studio em `/opt/android-studio`.

---

## 0. Pré-checagem do ambiente

Rode e confira cada linha antes de baixar 2 GB de imagem.

```sh
# virtualização: precisa do device, não só do módulo carregado
ls -l /dev/kvm            # esperado: crw-rw----+ 1 root kvm
id | tr ',' '\n' | grep kvm   # seu usuário precisa estar no grupo kvm
test -r /dev/kvm && test -w /dev/kvm && echo "KVM OK"

# ferramentas
ls ~/Android/Sdk/cmdline-tools/latest/bin   # precisa ter sdkmanager e avdmanager
~/Android/Sdk/emulator/emulator -version | head -1

# memória livre: AAOS quer ~4 GB só para ele
free -h
```

Se não estiver no grupo `kvm`:

```sh
sudo usermod -aG kvm "$USER"   # exige logout/login para valer
```

---

## 1. Java: o JDK do Android Studio

As `cmdline-tools` modernas exigem **JDK 17 ou mais novo**. Com Zulu 8 ativo, tanto
`sdkmanager` quanto `avdmanager` abortam:

```
This tool requires JDK 17 or later. Your version was detected as 1.8.0_442.
```

O Android Studio embute um JDK 21 (`jbr` = JetBrains Runtime). Use esse, sem instalar nada:

```sh
export JAVA_HOME=/opt/android-studio/jbr
export PATH="$JAVA_HOME/bin:$HOME/Android/Sdk/cmdline-tools/latest/bin:$HOME/Android/Sdk/platform-tools:$HOME/Android/Sdk/emulator:$PATH"
```

Verifique:

```sh
java -version        # esperado: openjdk version "21..."
sdkmanager --version
```

### Cuidado com o mise

O `mise` reescreve `PATH` e `JAVA_HOME` por diretório, então um `export` solto no
`~/.zshrc` pode ser sobrescrito. Duas saídas:

- colocar os `export` **depois** do hook de ativação do mise no `~/.zshrc`; ou
- criar um `.mise.toml` neste repositório fixando um JDK ≥ 17 para o projeto.

### Cuidado com `adb` duplicado

Se houver outro `adb` no `PATH` (ex.: `~/scrcpy/adb`), dois binários de builds
diferentes disputam o servidor na porta 5037 e produzem erros confusos de
`adb server version doesn't match this client`. O `PATH` acima coloca o
`platform-tools` do SDK na frente. Confirme com:

```sh
which adb      # esperado: ~/Android/Sdk/platform-tools/adb
```

---

## 2. Escolher e baixar a system image

Listar o que existe:

```sh
sdkmanager --list | grep -i automotive
```

Instalar a escolhida:

```sh
sdkmanager --install "system-images;android-34-ext9;android-automotive;x86_64"
```

### Por que essa imagem

**`android-automotive` (Google APIs), nunca `android-automotive-playstore`.**
As imagens com Play Store são builds `user`: sem `adb root`, com verified boot e com
restrições em comandos de shell privilegiados. As "Google APIs" são `userdebug`.
Como todo o projeto depende de `dumpsys car_service inject-vhal-event` responder via
`adb shell`, usar a imagem Play Store é arriscar descobrir tarde que o ponto de
integração está bloqueado por política de build.

**API 34** é o AAOS mais usado em produção e o melhor documentado. Alternativas
disponíveis: `android-35-ext15` (mais nova) e `android-33` (mais antiga).
O sufixo `-ext9` é apenas o nível de SDK Extensions; não muda nada aqui.

**`x86_64`**, não `arm64-v8a` — a imagem precisa casar com a arquitetura do host para
rodar sob KVM. Imagem ARM em host x86 cai em emulação por software e fica inutilizável.

---

## 3. Criar o AVD

Perfis Automotive disponíveis (`avdmanager list device | grep -i automotive`):
`automotive_1024p_landscape`, `automotive_1080p_landscape`,
`automotive_1408p_landscape_with_google_apis`, `automotive_portrait`,
`automotive_large_portrait`, `automotive_ultrawide`, `automotive_distant_display`.

```sh
avdmanager create avd \
  -n aaos34 \
  -k "system-images;android-34-ext9;android-automotive;x86_64" \
  -d automotive_1080p_landscape
```

Responda `no` à pergunta sobre hardware profile customizado.

Ajuste `~/.android/avd/aaos34.avd/config.ini`:

```ini
hw.ramSize=4096
vm.heapSize=576
hw.cpu.ncore=4
disk.dataPartition.size=6G
```

AAOS sobe CarService, cluster, HVAC e vários system apps — é bem mais pesado que
Android de celular. Abaixo de ~3,5 GB ele entra em thrashing e a lentidão parece
problema do emulador quando é falta de memória. Feche o que puder antes de subir.

Conferir:

```sh
emulator -list-avds
```

---

## 4. Subir o emulador

```sh
emulator -avd aaos34 -gpu host
```

Flags úteis:

| Flag | Quando usar |
|---|---|
| `-gpu host` | padrão; usa a GPU real |
| `-gpu swiftshader_indirect` | se a GPU host der tela preta ou crash |
| `-no-snapshot-load` | força cold boot, quando o estado salvo corrompeu |
| `-wipe-data` | reseta a partição de dados |
| `-writable-system` | se depois for preciso remontar `/system` rw |
| `-no-window` | headless, para CI |

O primeiro boot leva vários minutos (cold boot + otimização do ART). Em outro terminal:

```sh
adb wait-for-device shell getprop sys.boot_completed   # esperado: 1
adb devices
```

---

## 5. Validar o ponto de integração (o objetivo real)

Bootar não é o entregável. O entregável é saber a **sintaxe exata** do comando de
injeção nesta imagem — ela mudou entre versões de AAOS, tanto na ordem dos argumentos
quanto no formato do `area_id`.

```sh
# descobrir os subcomandos disponíveis nesta build
adb shell dumpsys car_service -h
```

Nenhum subcomando do `dumpsys car_service` usa prefixo `--`; são verbos posicionais.
Os quatro que interessam aqui:

```
get-carpropertyconfig [PROPERTY_ID in Hex or Decimal]
get-property-value    [PROPERTY_ID in Hex or Decimal] [areaId]
set-property-value    <PROPERTY_ID in Hex or Decimal> <areaId> <data>
inject-vhal-event     <PROPERTY_ID in Hex or Decimal> [zone] data [-t delay_seconds]
```

Propriedade de referência:

```
PERF_VEHICLE_SPEED = 0x11600207   # float, em m/s
```

```sh
# config: tipo, acesso e modo de mudança
adb shell dumpsys car_service get-carpropertyconfig 0x11600207

# leitura (pull)
adb shell dumpsys car_service get-property-value 0x11600207 0

# injeção (push para subscribers)
adb shell dumpsys car_service inject-vhal-event 0x11600207 0 20
```

### ARMADILHA: `inject-vhal-event` NÃO é observável via `get-property-value`

Verificado neste ambiente. Os dois comandos percorrem caminhos distintos que não se
comunicam:

- **`inject-vhal-event` é push.** Entrega um evento de propriedade aos *subscribers*
  (quem chamou `CarPropertyManager.registerCallback`). Simula "chegou dado novo do
  barramento". **Não escreve no estado do VHAL.**
- **`get-property-value` é pull.** Consulta o `FakeVehicleHardware` do emulador, que
  devolve o valor dele — valor que nunca viu a injeção.

Comprovado em duas propriedades com `changeMode` diferentes:

| Propriedade | changeMode | Injetado | `get` depois |
|---|---|---|---|
| `PERF_VEHICLE_SPEED` | `0x2` CONTINUOUS | 20 | `0.0` — inalterado |
| `GEAR_SELECTION` | `0x1` ON_CHANGE | 8 (DRIVE) | `4` (PARK) — inalterado |

A injeção **é aceita** — o logcat confirma:

```
CarShellCommand: Injecting VHAL event: prop=0x11600207, zone=0, value=20, isError=false
```

Consequência: **um valor inalterado no `get` não prova que a injeção falhou.**
`get-property-value` é a ferramenta errada para verificar injeção. Validar o caminho
exige algo inscrito na propriedade.

`set-property-value`, que escreveria o estado, está barrado nesta propriedade:

```
Cannot set a property: ServiceSpecificException: failed to set value for propID: 291504647 (code 4)
```

O config explica:

```
Property name:PERF_VEHICLE_SPEED, access:0x1, changeMode:0x2, fs min:1.0, fs max:10.0
```

| Campo | Leitura |
|---|---|
| `access:0x1` | READ apenas (`0x1`=READ, `0x2`=WRITE, `0x3`=READ_WRITE) — velocidade é sensor |
| `changeMode:0x2` | CONTINUOUS (`0x0`=STATIC, `0x1`=ON_CHANGE, `0x2`=CONTINUOUS) |
| `fs min/max` | faixa de sample rate permitida: 1 a 10 Hz |

**Isso não quebra o projeto.** O `VehicleDataSource` com `callbackFlow` sobre
`registerCallback` é exatamente um subscriber — é para ele que a injeção entrega.
O que muda é o instrumento de verificação, não a arquitetura.

### Anatomia da saída do `get-property-value`

```
Property:0x11600207, status: 0, timestamp: 6444400078094, zone: 0x0,
floatValues: [0.0], int32Values: [], int64Values: [], bytes: [], string:
```

| Campo | Significado |
|---|---|
| `status` | `0` = disponível; diferente de zero é sensor indisponível ou em erro — o gateway precisa tratar |
| `timestamp` | nanossegundos de `CLOCK_BOOTTIME`, não epoch; serve para detectar dado obsoleto |
| `zone` | o `areaId`; `0x0` em propriedades globais |
| `floatValues` / `int32Values` / … | só **um** é preenchido, conforme o tipo codificado no property ID |

Os buckets paralelos vazios vêm do `VehiclePropValue` do HIDL/AIDL: uma struct única que
carrega qualquer tipo. O mesmo bit-packing que você vai decodificar na trilha C.

### Verificação por log

```sh
adb logcat -s CarService CarPropertyService VehicleHal CarShellCommand
```

O `CarShellCommand` loga toda injeção com `isError`, e é a confirmação mais confiável
de que o comando foi aceito.

### Anotar aqui o comando que funcionou

> Imagem usada: `system-images;android-34-ext9;android-automotive;x86_64`
> Data: 2026-10-11
>
> Leitura:
> ```sh
> adb shell dumpsys car_service get-property-value 0x11600207 0
> ```
>
> Injeção:
>
>	'inject-vhal-event <PROPERTY_ID in Hex or Decimal> [zone] data(can > be comma separated list) [-t delay_time_seconds]
>	  Inject a vehicle property for testing.
>	  delay_time_seconds: the event timestamp is increased by certain second.
>	  If not specified, it will be 0.'
>
>
> Desvios da documentação / observações:

### Critério de conclusão do item da Semana 1

O item "Propriedade injetada e lida no VHAL do emulador, com comandos anotados" exige
**três** coisas, não duas:

- [x] sintaxe de `inject-vhal-event` anotada e comando aceito (`isError=false` no logcat)
- [x] sintaxe de `get-property-value` anotada e funcionando
- [ ] **evento injetado observado por um subscriber** — pendente

O terceiro item é o que de fato valida o ponto de integração do projeto, e não é
alcançável só com `dumpsys`, pela armadilha push/pull acima. Precisa de um subscriber
inscrito na propriedade: um app Kotlin mínimo com `registerCallback` + `Log.d`.

Esse app mínimo não é o app do projeto final — é instrumento de medição. Serve de
osciloscópio no canal que o gateway em C vai usar.

---

## 6. Troubleshooting

| Sintoma | Causa provável | Ação |
|---|---|---|
| `This tool requires JDK 17 or later` | JDK 8/11 ativo | `export JAVA_HOME=/opt/android-studio/jbr` (seção 1) |
| `/dev/kvm` não existe | módulo não carregado ou virtualização off na BIOS/UEFI | `sudo modprobe kvm_amd` (ou `kvm_intel`); habilitar SVM/VT-x na UEFI |
| `/dev/kvm permission denied` | fora do grupo `kvm` | `sudo usermod -aG kvm "$USER"` + logout |
| `adb server version doesn't match` | dois `adb` diferentes no `PATH` | `adb kill-server`; corrigir `PATH` (seção 1) |
| Boot trava no logo, muito lento | RAM insuficiente | subir `hw.ramSize`; fechar aplicações do host |
| Tela preta ao subir | driver de GPU | `-gpu swiftshader_indirect` |
| Emulador sobe e morre sem log | estado salvo corrompido | `-no-snapshot-load` ou `-wipe-data` |
| `dumpsys car_service` não lista `inject-vhal-event` | imagem Play Store (build `user`) | recriar AVD com a imagem Google APIs (seção 2) |

---

## Referência rápida

```sh
export JAVA_HOME=/opt/android-studio/jbr
export PATH="$JAVA_HOME/bin:$HOME/Android/Sdk/cmdline-tools/latest/bin:$HOME/Android/Sdk/platform-tools:$HOME/Android/Sdk/emulator:$PATH"

emulator -avd aaos34 -gpu host &
adb wait-for-device shell getprop sys.boot_completed
adb shell dumpsys car_service -h
```
