# Варианты разработки ПО

## кросс-разработка на хост-машине.
### (РЕКОМЕНДУЕТСЯ)**

### Вариант 1: Кросс-разработка на Debian 12
* **Шаг 1: Подготовка хост-системы**
```bash 
sudo apt update
sudo apt install build-essential cmake git wget curl python3 python3-pip
sudo apt install gcc-arm-linux-gnueabihf binutils-arm-linux-gnueabihf
```
* **Шаг 2: Скачивание и установка Legato Application Framework**
```bash
cd ~/
wget https://downloads.sierrawireless.com/tools_and_apps/legato/releases/21_05_0/legato-linux64-21.05.0.tar.bz2
tar -xjf legato-linux64-21.05.0.tar.bz2
cd legato-21.05.0
```
* **Шаг 3: Сборка Legato для FX30S**
```bash 
make fx30s
```
* **Шаг 4: Настройка переменных окружения**
```bash 
echo 'export LEGATO_ROOT=~/legato-21.05.0' >> ~/.bashrc
echo 'export PATH=$LEGATO_ROOT/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```
* **Шаг 5: Установка Developer Studio (опционально)**
```bash 
wget https://downloads.sierrawireless.com/tools_and_apps/developer-studio/linux/developer-studio-linux-6.0.0.tar.bz2
tar -xjf developer-studio-linux-6.0.0.tar.bz2
cd developer-studio-6.0.0
./install.sh
```
* **Шаг 6: Подключение FX30S к компьютеру**
  * Подключите FX30S через USB Устройство появится как сетевой интерфейс (обычно 192.168.2.2)
```bash
ping 192.168.2.2
```
* **Шаг 7: Настройка SSH доступа к FX30S Пароль по умолчанию: swi**
```bash
ssh root@192.168.2.2
```
* **Шаг 8: Создание тестового приложения**
```bash 
mkdir ~/fx30s_projects
cd ~/fx30s_projects
mkdir helloWorld
cd helloWorld
```

* **hello.c**
```C
#include "legato.h"

COMPONENT_INIT
{
    LE_INFO("Hello World from FX30S!");
    
    // Тестовый таймер
    le_timer_Ref_t timer = le_timer_Create("HelloTimer");
    le_timer_SetMsInterval(timer, 5000);
    le_timer_SetRepeat(timer, 0);
    
    le_timer_SetHandler(timer, [](le_timer_Ref_t t) {
        LE_INFO("Timer tick - FX30S is working!");
    });
    
    le_timer_Start(timer);
}
```

* **hello.c**
```C
#include "legato.h"

COMPONENT_INIT
{
    LE_INFO("Hello World from FX30S!");
    
    // Тестовый таймер
    le_timer_Ref_t timer = le_timer_Create("HelloTimer");
    le_timer_SetMsInterval(timer, 5000);
    le_timer_SetRepeat(timer, 0);
    
    le_timer_SetHandler(timer, [](le_timer_Ref_t t) {
        LE_INFO("Timer tick - FX30S is working!");
    });
    
    le_timer_Start(timer);
}
```
* **Component.cdef**
```
sources:
{
    hello.c
}
```
* **helloWorld.adef**
``` 
version: 1.0.0

executables:
{
    helloWorld = ( helloComponent )
}

processes:
{
    run:
    {
        (helloWorld)
    }
    faultAction: restart
}
```

### Шаг 9: Сборка приложения
```bash
mkapp -t fx30s helloWorld.adef
```

### Шаг 10: Установка на FX30S
```bash
app install helloWorld.fx30s.update 192.168.2.2
```

### Шаг 11: Запуск и мониторинг
```bash
app start helloWorld --ip=192.168.2.2
ssh root@192.168.2.2 "logread -f"
```

## Вариант 2: Разработка непосредственно на FX30S
### (НЕ РЕКОМЕНДУЕТСЯ)
### Шаг 1: Подключение к FX30S
```bash
ssh root@192.168.2.2
```

### Шаг 2: Установка инструментов разработки на FX30S
```bash
opkg update
opkg install gcc make
```

### Шаг 3: Создание простого приложения
```C
// test.c
#include <stdio.h>
#include <unistd.h>

int main() {
    while(1) {
        printf("Hello from FX30S!\n");
        sleep(5);
    }
    return 0;
}
```

### Шаг 4: Компиляция на устройстве
```bash
gcc -o test test.c
./test
```

## Сравнение подходов
| Аспект	| Кросс-разработка	| Разработка на устройстве |
|-----------|-------------------|--------------------------|
| Производительность	| ✅ Быстрая сборка	| ❌ Медленная сборка |
| Ресурсы	| ✅ Не нагружает FX30S	| ❌ Использует ресурсы FX30S |
| Отладка	| ✅ Полные инструменты	| ❌ Ограниченные возможности |
| Legato API	| ✅ Полная поддержка	| ❌ Ограниченная поддержка |
| Версионирование	| ✅ Git, IDE	| ❌ Сложно |

## Рекомендуемый workflow для разработки
### 1. Структура проекта на хост-машине
```bash
mkdir ~/fx30s_modbus_gateway
cd ~/fx30s_modbus_gateway
git init
mkdir -p src/{components,apps,config}
```

### 2. Автоматизация сборки
```Makefile
TARGET := fx30s
DEVICE_IP := 192.168.2.2
APP_NAME := modbusGateway

.PHONY: build install start stop logs clean

build:
	mkapp -t $(TARGET) apps/$(APP_NAME).adef

install: build
	app install $(APP_NAME).$(TARGET).update $(DEVICE_IP)

start:
	app start $(APP_NAME) --ip=$(DEVICE_IP)

stop:
	app stop $(APP_NAME) --ip=$(DEVICE_IP)

logs:
	ssh root@$(DEVICE_IP) "logread -f | grep $(APP_NAME)"

clean:
	rm -rf _build_* *.update

deploy: install start

debug: logs
```
* **Варианты запуска Makefile**
```bash 
make build
make install
make start
make logs
```

## Дополнительные инструменты
* **VS Code настройка - settings.json**
```json
{
    "C_Cpp.default.compilerPath": "/usr/bin/arm-linux-gnueabihf-gcc",
    "C_Cpp.default.includePath": [
        "${workspaceFolder}/**",
        "~/legato-21.05.0/framework/include",
        "~/legato-21.05.0/build/fx30s/framework/include"
    ],
    "C_Cpp.default.defines": [
        "TARGET_fx30s"
    ]
}
```
* **Скрипт для быстрого развертывания - deploy.sh**
```bash 
#!/bin/bash
# deploy.sh
set -e

APP_NAME="modbusGateway"
DEVICE_IP="192.168.2.2"
TARGET="fx30s"

echo "Building application..."
mkapp -t $TARGET apps/$APP_NAME.adef

echo "Installing to device..."
app install $APP_NAME.$TARGET.update $DEVICE_IP

echo "Starting application..."
app start $APP_NAME --ip=$DEVICE_IP

echo "Deployment complete!"
echo "View logs with: ssh root@$DEVICE_IP 'logread -f'"
```
```bash
chmod +x deploy.sh
```

## Используйте кросс-разработку на Debian 12 - это:
* **профессиональный подход**
* **обеспечивает лучшую:**
  * производительность,
  * отладку и
  * интеграцию с Legato Framework.