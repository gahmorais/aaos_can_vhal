### Lista todas as configurações observáveis da imagem
```sh
adb shell dumpsys car_service get-carpropertyconfig
```
Ao listar todas as propriedades, a principio utilize grep para filtrar uma em especifico

```sh
adb shell dumpsys car_service get-carpropertyconfig | grep SPEED

Property:0x11600207, Property name:PERF_VEHICLE_SPEED, access:0x1, changeMode:0x2, config:[], fs min:1.000000, fs max:10.000000
Property:0x11600208, Property name:PERF_VEHICLE_SPEED_DISPLAY, access:0x1, changeMode:0x2, config:[], fs min:1.000000, fs max:10.000000
Property:0x11601013, Property name:CRUISE_CONTROL_TARGET_SPEED, access:0x1, changeMode:0x1, config:[], fs min:0.000000, fs max:0.000000
Property:0x15400500, Property name:HVAC_FAN_SPEED, access:0x3, changeMode:0x1, config:[], fs min:0.000000, fs max:0.000000
Property:0x1540050f, Property name:HVAC_ACTUAL_FAN_SPEED_RPM, access:0x1, changeMode:0x1, config:[], fs min:0.000000, fs max:0.000000

```


Para buscar o valor da propriedade utilize o valor de Property, é um hexadecimal, como abaixo:

> Property:0x11600207 

### Propriedade que representa a velocidade do veículo
> Property name:PERF_VEHICLE_SPEED, access:0x1, changeMode:0x2, config:[], fs min:1.000000, fs max:10.000000

### Execução do comando
```sh
adb shell dumpsys car_service get-carpropertyconfig 0x11600207
```
---
### Como criar e inicializar a interface CAN no PopOS

```sh
modinfo vcan #Verifica se o módulo do kernel existe
lsmod | grep vcan ##Verifica se já foi carregado
ls /lib/modules/$(uname -r)/kernel/net/can #vcan.ko.zst deve estar neste caminho
```

### Carregando o módulo
```sh
sudo modprobe vcan
lsmod | grep vcan #Confirma se o módulo carregou
```

### Criar e iniciar a interface de comunicação
```sh
sudo ip link add dev vcan0 type vcan
sudo ip link set up vcan0

ip -details link show vcan0 ## Verifica se a interface foi criada e está ativa
```

### Software para teste comunicação can-utils

```sh
sudo apt install can-utils
```
Terminal 1
```sh
candump vcan0 ##Faz a leitura dos dados
```
Terminal 2
```sh
cansend vcan0 '123#DEADBEEF' ## Faz o envio dos dados
```
candump deve mostrar
> vcan0  123   [4]  DE AD BE EF

Variações possíveis
```sh
cansend vcan0 '123#DEADBEEF' #ID 11 bits, 4 bytes
cansend vcan0 '1F334455#1122334455667788' #ID 29 bits, 8 bytes
cansend vcan0 '123#'                      # DLC = 0, frame sem dados
cansend vcan0 '123#R'                     # remote transmission request
cangen vcan0 -g 100 -I 123 -L 8         # tráfego contínuo, 100 ms, ID fixo, 8 bytes
candump -ta vcan0           # timestamp absoluto
candump -td vcan0           # delta entre frames — para medir período
candump vcan0,123:7FF       # filtro: só ID 0x123 (valor:máscara)
candump -x vcan0            # mostra direção (TX/RX)
```

# carregar o módulo no boot
```sh
echo vcan | sudo tee /etc/modules-load.d/vcan.conf
```

