# Sierra Wireless FX30S - Legato OS
### позволяет разрабатывать собственные приложения:

## Legato OS
* **Legato - Linux-based платформа для IoT устройств Sierra Wireless**
* **построенная поверх Yocto Linux.**
    
### Она предоставляет:
- API для работы с модемом, GPIO, SPI, I2C
- Встроенную поддержку облачных сервисов
- Систему управления приложениями

## Настройка среды разработки
### 1. Установка Legato Application Framework
```bash
wget https://downloads.sierrawireless.com/tools_and_apps/legato/releases/latest/legato-linux64-bin-18.10.0.tar.bz2
tar -xjf legato-linux64-bin-18.10.0.tar.bz2
cd legato-18.10.0
make fx30s
```

## 2. Настройка переменных окружения
```bash
export LEGATO_ROOT=/path/to/legato
export PATH=$LEGATO_ROOT/bin:$PATH
```

### Структура приложения
```C
// modbusGateway.c
#include "legato.h"
#include "interfaces.h"
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>

// Modbus RTU функции
static int modbusSocket = -1;
static le_timer_Ref_t pollTimer;

// Структура для хранения данных устройства
typedef struct {
    uint16_t deviceId;
    uint16_t registerAddr;
    uint16_t value;
    time_t timestamp;
} ModbusData_t;

// Инициализация Modbus соединения
static le_result_t InitModbus(void)
{
    // Настройка UART для Modbus RTU
    le_uart_DeviceRef_t uartRef = le_uart_Open("ttyHS0");
    if (!uartRef) {
        LE_ERROR("Failed to open UART");
        return LE_FAULT;
    }
    
    le_uart_SetBaudRate(uartRef, LE_UART_BAUD_RATE_9600);
    le_uart_SetNumDataBits(uartRef, 8);
    le_uart_SetParity(uartRef, LE_UART_PARITY_NONE);
    le_uart_SetNumStopBits(uartRef, 1);
    
    return LE_OK;
}

// Чтение данных по Modbus
static le_result_t ReadModbusData(uint8_t deviceId, uint16_t regAddr, uint16_t* value)
{
    // Формирование Modbus RTU запроса
    uint8_t request[8];
    request[0] = deviceId;           // Slave ID
    request[1] = 0x03;              // Function code (Read Holding Registers)
    request[2] = (regAddr >> 8) & 0xFF;  // Register address high byte
    request[3] = regAddr & 0xFF;         // Register address low byte
    request[4] = 0x00;              // Number of registers high byte
    request[5] = 0x01;              // Number of registers low byte
    
    // Добавление CRC
    uint16_t crc = CalculateCRC(request, 6);
    request[6] = crc & 0xFF;
    request[7] = (crc >> 8) & 0xFF;
    
    // Отправка запроса и получение ответа
    // Здесь должна быть реализация отправки через UART
    
    return LE_OK;
}

// Отправка данных на сервер по HTTP
static void SendDataHTTP(ModbusData_t* data)
{
    char jsonData[256];
    snprintf(jsonData, sizeof(jsonData), 
        "{\"deviceId\":%d,\"register\":%d,\"value\":%d,\"timestamp\":%ld}",
        data->deviceId, data->registerAddr, data->value, data->timestamp);
    
    // HTTP POST запрос
    le_httpClient_RequestRef_t requestRef = le_httpClient_CreateRequest();
    le_httpClient_SetMethod(requestRef, LE_HTTPCLIENT_METHOD_POST);
    le_httpClient_SetUrl(requestRef, "http://your-server.com/api/data");
    le_httpClient_SetHeader(requestRef, "Content-Type", "application/json");
    le_httpClient_SetBody(requestRef, jsonData, strlen(jsonData));
    
    le_httpClient_Send(requestRef);
}

// Отправка данных по MQTT
static void SendDataMQTT(ModbusData_t* data)
{
    char topic[64];
    char payload[256];
    
    snprintf(topic, sizeof(topic), "devices/%d/data", data->deviceId);
    snprintf(payload, sizeof(payload), 
        "{\"register\":%d,\"value\":%d,\"timestamp\":%ld}",
        data->registerAddr, data->value, data->timestamp);
    
    // Публикация MQTT сообщения
    le_mqtt_Publish(topic, payload, strlen(payload), LE_MQTT_QOS_0, false);
}

// Таймер для периодического опроса устройств
static void PollDevices(le_timer_Ref_t timerRef)
{
    ModbusData_t data;
    
    // Опрос устройств
    for (int deviceId = 1; deviceId <= 10; deviceId++) {
        if (ReadModbusData(deviceId, 0x0001, &data.value) == LE_OK) {
            data.deviceId = deviceId;
            data.registerAddr = 0x0001;
            data.timestamp = time(NULL);
            
            // Отправка данных
            SendDataHTTP(&data);
            SendDataMQTT(&data);
        }
    }
}
```

```C
//COMPONENT_INIT
{
    LE_INFO("Modbus Gateway starting...");
    
    // Инициализация Modbus
    if (InitModbus() != LE_OK) {
        LE_ERROR("Failed to initialize Modbus");
        return;
    }
    
    // Создание таймера для периодического опроса
    pollTimer = le_timer_Create("PollTimer");
    le_timer_SetMsInterval(pollTimer, 5000); // 5 секунд
    le_timer_SetRepeat(pollTimer, 0); // Бесконечное повторение
    le_timer_SetHandler(pollTimer, PollDevices);
    le_timer_Start(pollTimer);
    
    LE_INFO("Modbus Gateway started successfully");
}
```

### Файл определения компонента
```
//Component.cdef
sources:
{
    modbusGateway.c
}

requires:
{
    api:
    {
        le_uart.api
        le_httpClient.api
        le_mqtt.api
    }
}

cflags:
{
    -std=c99
    -Wall
}
```

### Файл определения приложения
```
// modbusGateway.adef
version: 1.0.0

executables:
{
    modbusGateway = ( modbusGatewayComponent )
}

processes:
{
    run:
    {
        (modbusGateway)
    }
    
    faultAction: restart
    maxCoreDumpFileBytes: 100K
    maxFileBytes: 50K
    maxFileDescriptors: 20
}

bindings:
{
    modbusGateway.modbusGatewayComponent.le_uart -> modemService.le_uart
    modbusGateway.modbusGatewayComponent.le_httpClient -> dataConnectionService.le_httpClient
}

requires:
{
    device:
    {
        [rw] /dev/ttyHS0 /dev/ttyHS0
    }
}
```


## Сборка и развертывание
### 1. Сборка приложения
```bash
mkapp -t fx30s modbusGateway.adef
```

### 2. Установка на устройство
```bash
app install modbusGateway.fx30s.update 192.168.2.2
```

### 3. Запуск приложения
```bash
app start modbusGateway
```

### Execute
* **Дополнительные возможности**
  * **gateway_config.json** 
```json
{
    "modbus": {
        "port": "/dev/ttyHS0",
        "baudRate": 9600,
        "parity": "none",
        "stopBits": 1,
        "timeout": 1000
    },
    "devices": [
        {
            "id": 1,
            "registers": [1, 2, 3, 4]
        },
        {
            "id": 2,
            "registers": [10, 11, 12]
        }
    ],
    "server": {
        "http": {
            "url": "http://your-server.com/api/data",
            "timeout": 5000
        },
        "mqtt": {
            "broker": "mqtt.your-server.com",
            "port": 1883,
            "clientId": "fx30s_gateway"
        }
    }
}
```

### Отладка и мониторинг
* **Просмотр логов**
```bash
logread -f
```
* **Мониторинг приложения**
```bash 
app status
app info modbusGateway
```

## Кросс-компиляция
    Legato использует встроенную систему кросс-компиляции для ARM архитектуры FX30S. Все необходимые инструменты включены в Legato Application Framework.

* **Для использования внешних библиотек создайте файл:**
```Makefile
export LEGATO_ROOT := /path/to/legato
export TARGET := fx30s
include $(LEGATO_ROOT)/framework/tools/mk/legato.mk

# Дополнительные библиотеки
LDFLAGS += -lmodbus -ljson-c
```

    Этот подход позволяет создать полнофункциональный шлюз для сбора данных по Modbus и их передачи на сервер через различные протоколы.