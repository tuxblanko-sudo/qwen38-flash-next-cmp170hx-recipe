# Установка Qwen3.8-Flash-Next на 2× CMP 170HX

Как включить их в BIOS (UEFI)

Если утилиты показали, что лимиты зажаты на 250–512 МБ, вам нужно перезагрузить сервер, зайти в BIOS (обычно клавишами Del или F2 при включении) и изменить настройки.
В зависимости от производителя материнской платы (ASUS, Gigabyte, MSI, ASRock, Supermicro), эти опции чаще всего находятся в разделах Advanced, PCI Subsystem Settings, Boot или Peripherals.

Шаг 1: Включение Above 4G Decoding

• Найдите параметр: Above 4G Decoding (иногда называется Memory Mapping above 4GB).
• Переведите его в режим: Enabled.
• Зачем: Это разрешает процессору и операционной системе адресовать устройства, использующие более 4 ГБ памяти (что жизненно необходимо для карт с 64 ГБ на борту).
```
sudo dmesg | grep -iE 'bios-e820|root bridge'
```

Шаг 2: Включение Resizable BAR

• Найдите параметр: Re-Size BAR Support (или Smart Access Memory / Clever Access Memory).
• Переведите его в режим: Enabled (или Auto).
• Внимание: На многих платах этот параметр не появится, пока вы не выполните Шаг 1 (Above 4G Decoding должен быть строго Enabled).

```
lspci -v -d 10de: | grep -E 'Region|Memory at'

```
Шаг 3: Отключение CSM (Важно!)

• Найдите параметр: CSM (Compatibility Support Module) или Legacy Boot.
• Переведите его в режим: Disabled (система должна загружаться строго в чистом режиме UEFI).
• Зачем: Resizable BAR физически не умеет работать в старом режиме эмуляции BIOS (Legacy/CSM).


1. Для штатного запуска рецепта нужны две карты с 64 GiB памяти каждая. Не всякая CMP 170HX после разблокировки получает такой объём.
2. Автор тестировал производительность на PCIe x16. При штатной ширине x4 рецепт предупреждает о снижении производительности, но не должен из-за этого останавливать запуск.&#x20;

   [image](https://www.google.com/s2/favicons?domain=https://raw.githubusercontent.com\&sz=32)

   GitHub+2



## 0. Сначала определяем, подходят ли обе карты

На сервере выполните:

```
sudo apt update
sudo apt install -y pciutils

lspci -nn | grep -iE 'NVIDIA|10de:20b0|10de:20c2|10de:2082'
```

Для `cmpunlocker` важны следующие идентификаторы:

| PCI ID      | Вариант карты     | Результат разблокировки         |
| ----------- | ----------------- | ------------------------------- |
| `10de:20c2` | CMP 170HX, 8 GB   | 64 GiB                          |
| `10de:2082` | CMP 170HX, 10 GB  | 40 GiB                          |
| `10de:20b0` | Другой вариант ID | Этот unlock-патч его пропускает |

Установщик умеет обрабатывать несколько карт и даже смешанную конфигурацию, но смешанный вариант не подходит под требования этого рецепта. Его `start.sh` проверяет, что обе карты имеют не менее 61 440 MiB.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://raw.githubusercontent.com\&sz=32)

GitHub+2



Продолжайте по полной инструкции, если обе карты показывают `10de:20c2`. Если хотя бы одна показывает `10de:2082`, не переходите к запуску Compose: штатная проверка памяти рецепта не пройдёт.

## 1. Подготовка Ubuntu 24.04

Сначала обновите систему и установите инструменты сборки модулей:

```
sudo apt update
sudo apt upgrade -y

sudo apt install -y \
    build-essential \
    dkms \
    linux-headers-$(uname -r) \
    python3 \
    python3-yaml \
    git \
    curl \
    patch \
    pciutils \
    mokutil \
    ca-certificates \
    gnupg \
    tar \
    xz-utils
```

Проверьте Secure Boot:

```
sudo mokutil --sb-state
```

Для этого варианта разблокировки Secure Boot должен быть отключён: `cmpunlocker` устанавливает неподписанные изменённые модули NVIDIA. Отключите Secure Boot в BIOS/UEFI сервера, сохраните изменения и загрузите Ubuntu.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://raw.githubusercontent.com\&sz=32)

GitHub+1



Повторите проверку:

```
sudo mokutil --sb-state
```

Ожидаемый результат:

```
SecureBoot disabled
```

## 2. Установка подходящего драйвера NVIDIA

Здесь важно не устанавливать произвольный драйвер.

В актуальном `cmpunlocker` есть список поддерживаемых версий, включая `615.71.09`. В Ubuntu 24.04 доступен соответствующий пакет `nvidia-driver-615-open` версии `615.71.09-0ubuntu0.24.04.2`. Это открытый вариант kernel-модуля, который подходит под требования unlocker.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://raw.githubusercontent.com\&sz=32)

GitHub+2



Включите компонент Ubuntu Multiverse:

```
sudo apt install -y software-properties-common

sudo add-apt-repository -y multiverse

sudo apt update
```

Посмотрите доступную версию:

```
apt-cache policy nvidia-driver-615-open
```

Ожидаемая версия Candidate:

```
615.71.09-0ubuntu0.24.04.2
```

Устанавливаем именно её:

```
sudo apt install -y \
    nvidia-driver-615-open=615.71.09-0ubuntu0.24.04.2
```

Перезагрузите сервер:

```
sudo reboot
```

После загрузки проверяем драйвер:

```
nvidia-smi
```

И отдельно:

```
cat /proc/driver/nvidia/version
```

Должна определяться версия:

```
615.71.09
```

Пока не переходите дальше, если `nvidia-smi` не видит две карты или версия драйвера отличается от ожидаемой. Установщик разблокировки проверяет точное совпадение версии драйвера с собственным списком поддерживаемых версий.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://raw.githubusercontent.com\&sz=32)

GitHub+1



## 3. Разблокировка обеих CMP 170HX

Используем отдельный проект:

[amoghmunikote/cmpunlocker](https://github.com/amoghmunikote/cmpunlocker?utm_source=chatgpt.com)

Он поддерживает установку на несколько карт, в том числе одновременно на варианты 8 GB и 10 GB. Для каждой карты профиль определяется по PCI ID автоматически. Поэтому вручную задавать один общий `--profile` для двух карт не надо.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://raw.githubusercontent.com\&sz=32)

GitHub+1



Скачиваем:

```
cd ~

git clone https://github.com/amoghmunikote/cmpunlocker.git

cd cmpunlocker
```

Посмотрите список поддерживаемых драйверов:

```
cat driver/VERSION
```

В нём должна быть версия `615.71.09`.

Посмотрите параметры установщика:

```
sudo ./install.sh --help
```

Для физического сервера с непосредственным использованием GPU, а не для PCI passthrough в виртуальные машины, запускаем:

```
sudo ./install.sh --no-iommu --no-passthrough
```

Почему именно так:

- `--no-passthrough` не настраивает карты для передачи в виртуальные машины.
- `--no-iommu` не добавляет параметры IOMMU в GRUB. Для запуска модели непосредственно на Ubuntu это не требуется.
- PCIe Gen2 retrain service оставляем включённым: установщик по умолчанию настраивает его для повторной тренировки PCIe-соединения при загрузке.

Это реальные параметры текущего `install.sh`, а не придуманные ключи.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://raw.githubusercontent.com\&sz=32)

GitHub+2



Во время выполнения проверьте, что установщик определил именно две разблокируемые карты. Для двух `10de:20c2` ожидается примерно такая информация:

```
10de:20c2 → 8gb
10de:20c2 → 8gb

Unlock geometry: 64GB per card
```

При другом PCI ID результат будет отличаться. Не подменяйте автоматически определённый профиль ручным параметром.

Установщик сохраняет журнал в каталоге `cmpunlocker/logs/`.

## 4. Полное выключение и проверка разблокировки

После успешного выполнения установщика требуется не просто перезагрузка, а холодный старт. Именно его рекомендует сам проект.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://raw.githubusercontent.com\&sz=32)

GitHub



Выполните:

```
sudo shutdown -h now
```

Дождитесь полного выключения сервера, затем включите его физически.

После загрузки:

```
nvidia-smi
```

Для двух карт с PCI ID `10de:20c2` ожидаем:

```
GPU 0    65536 MiB
GPU 1    65536 MiB
```

Теперь выполним проверку самого unlocker:

```
cd ~/cmpunlocker

sudo ./verify.sh
```

Проверим PCIe:

```
nvidia-smi \
  --query-gpu=name,memory.total,pci.bus_id,pcie.link.gen.current,pcie.link.gen.max,pcie.link.width.current,pcie.link.width.max \
  --format=csv
```

Для Gen2 ожидается:

```
pcie.link.gen.current = 2
pcie.link.gen.max     = 2
```

Отдельно обратите внимание на ширину PCIe. Если получаете x4, это не исправляется одной программной разблокировкой памяти. В README рецепта прямо указано, что его производительность измерялась на x16, с физической модификацией карт. При x4 скрипт предупреждает о снижении скорости и продолжает работу.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://raw.githubusercontent.com\&sz=32)

GitHub+1



Дополнительно проверьте топологию:

```
nvidia-smi topo -m
```

На этом этапе должны быть выполнены три условия: две карты видны, у каждой 65536 MiB, драйвер не выдаёт ошибок.

## 5. Установка Docker Engine и Compose

Для проекта не требуется вручную устанавливать vLLM, PyTorch и CUDA на саму Ubuntu. Движок поставляется в контейнере, причём версия образа закреплена по digest.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://raw.githubusercontent.com\&sz=32)

GitHub+1



Установим Docker Engine из официального репозитория Docker.

Добавляем ключ и репозиторий:

```
sudo apt update

sudo apt install -y ca-certificates curl

sudo install -m 0755 -d /etc/apt/keyrings

sudo curl -fsSL \
  https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc

sudo chmod a+r /etc/apt/keyrings/docker.asc
```

Добавляем репозиторий:

```
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF
```

Устанавливаем Docker и Compose:

```
sudo apt update

sudo apt install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin
```

Запускаем Docker:

```
sudo systemctl enable --now docker

sudo systemctl status docker
```

Проверяем:

```
sudo docker run hello-world
```

Это официальная последовательность установки Docker Engine для Ubuntu 24.04.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.docker.com\&sz=32)

Docker Docs



## 6. Установка NVIDIA Container Toolkit

Это компонент, который позволяет Docker-контейнерам получать доступ к GPU хоста.

Устанавливаем инструменты:

```
sudo apt-get install -y \
    ca-certificates \
    curl \
    gnupg2
```

Добавляем репозиторий NVIDIA:

```
curl -fsSL \
  https://nvidia.github.io/libnvidia-container/gpgkey | \
  sudo gpg --dearmor \
  -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
```

```
curl -s -L \
  https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
```

Устанавливаем версию из актуальной инструкции NVIDIA:

```
sudo apt-get update

export NVIDIA_CONTAINER_TOOLKIT_VERSION=1.20.1-1

sudo apt-get install -y \
    nvidia-container-toolkit=${NVIDIA_CONTAINER_TOOLKIT_VERSION} \
    nvidia-container-toolkit-base=${NVIDIA_CONTAINER_TOOLKIT_VERSION} \
    libnvidia-container-tools=${NVIDIA_CONTAINER_TOOLKIT_VERSION} \
    libnvidia-container1=${NVIDIA_CONTAINER_TOOLKIT_VERSION}
```

Настраиваем Docker Runtime:

```
sudo nvidia-ctk runtime configure --runtime=docker

sudo systemctl restart docker
```

Команда `nvidia-ctk` изменит конфигурацию Docker в `/etc/docker/daemon.json`.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.nvidia.com\&sz=32)

NVIDIA Container Toolkit



Проверяем доступ к двум GPU из контейнера:

```
sudo docker run --rm --gpus all \
  nvidia/cuda:13.3.1-base-ubuntu24.04 \
  nvidia-smi
```

Ожидаемый результат — две карты с разблокированной памятью, по 65536 MiB каждая.

Это также контрольная команда, которую прямо рекомендует README устанавливаемого проекта.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://raw.githubusercontent.com\&sz=32)

GitHub



## 7. Скачивание и подготовка целевого проекта

Теперь устанавливаем именно репозиторий, который вы предоставили:

[https://github.com/tuxblanko-sudo/qwen38-flash-next-cmp170hx-recipe](https://github.com/tuxblanko-sudo/qwen38-flash-next-cmp170hx-recipe?utm_source=chatgpt.com)

Он является форком проекта Morrowmake. Важный момент: README содержит инструкцию и для upstream-репозитория, но ниже я использую именно ваш URL.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://github.com\&sz=32)

GitHub+1



### 7.1. Подготовка каталога

Для модели потребуется около 115 GiB дискового пространства. Сам рецепт требует примерно 120 GiB свободного места на SSD; кроме модели, понадобятся место под Docker image и кэш компиляции.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://raw.githubusercontent.com\&sz=32)

GitHub+2



Посмотрите доступные диски:

```
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS
```

В следующих командах использую `/opt/llm`. Убедитесь, что этот каталог расположен на SSD с достаточным свободным пространством. При необходимости замените его на точку монтирования вашего SSD.

```
sudo mkdir -p /opt/llm

sudo chown "$USER":"$(id -gn)" /opt/llm

cd /opt/llm
```

Клонируем:

```
git clone https://github.com/tuxblanko-sudo/qwen38-flash-next-cmp170hx-recipe.git

cd qwen38-flash-next-cmp170hx-recipe
```

Проверяем место:

```
df -h .
```

### 7.2. Проверка конфигурации Compose

Посмотрим основные параметры:

```
cat docker/container.env
```

В исходной конфигурации используются:

```
LAYOUT=tp2
CUDA_VISIBLE_DEVICES=0,1
P2P=auto
HOST=0.0.0.0
PORT=8000
```

Это означает:

- `tp2` — модель распределяется на две видеокарты через Tensor Parallel 2.
- `CUDA_VISIBLE_DEVICES=0,1` — используются обе карты.
- `P2P=auto` — проверяется возможность безопасного обмена данными между GPU.
- `PORT=8000` — порт API.

Это параметры из самого проекта, менять их на первоначальном запуске не нужно.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://raw.githubusercontent.com\&sz=32)

GitHub+1



Проверьте Compose:

```
sudo docker compose config --quiet
```

Если команда ничего не вывела и вернула код 0 — конфигурация прошла проверку.

## 8. Запуск модели через Docker Compose

Это основной этап.

Запускаем:

```
sudo docker compose up -d
```

Что происходит внутри проекта:

1. Подготовка контейнера

   Docker скачивает закреплённый образ `vllm-cmp170hx`.
2. Скачивание модели

   Сервис `prepare` скачивает `turboderp/Qwen3.8-Flash-Next-exl3`, вариант 5.05 bpw, с закреплённой ревизией Hugging Face.
3. Подготовка кэша

   Создаются каталоги для кэша и временных файлов.
4. Запуск vLLM

   Стартует сервер с Tensor Parallel 2, BF16 KV cache и MTP speculative decoding.
5. Проверка готовности

   Compose проверяет `/health`. После успешной проверки контейнер получает статус `healthy`.

Последовательность следует из docker-compose.yml, docker/prepare.sh и README.

[image](https://www.google.com/s2/favicons?domain=https://raw.githubusercontent.com\&sz=32)

GitHub+2

Проверяем состояние:

```
sudo docker compose ps -a
```

Смотрим логи:

```
sudo docker compose logs -f
```

Для просмотра только загрузки модели:

```
sudo docker compose logs -f prepare
```

Для просмотра работы сервера:

```
sudo docker compose logs -f qwen38-flash-next
```

Первый запуск может быть значительно медленнее последующих: движок выполняет первоначальную компиляцию, а загрузка checkpoint занимает значительное место на диске. Прерванная загрузка модели поддерживает продолжение, а готовый checkpoint повторно не скачивается.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://raw.githubusercontent.com\&sz=32)

GitHub+1



## 9. Проверка готовности API

Проверяем health endpoint:

```
curl -i http://127.0.0.1:8000/health
```

Проверяем опубликованную модель:

```
curl http://127.0.0.1:8000/v1/models
```

В конфигурации используется имя:

```
qwen3.8-flash-next
```

Теперь отправим тестовый запрос:

```
curl -X POST http://127.0.0.1:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen3.8-flash-next",
    "messages": [
      {
        "role": "user",
        "content": "Привет! Расскажи кратко, что ты умеешь."
      }
    ],
    "max_tokens": 256,
    "temperature": 0.2
  }'
```

При успешном выполнении вы получите JSON с ответом модели.

Обратите внимание: Compose публикует API на `127.0.0.1:8000`, то есть только на localhost сервера. В исходной конфигурации API не защищён ключом. Не открывайте его в сеть без предварительной настройки аутентификации и публикации порта на нужном интерфейсе.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://raw.githubusercontent.com\&sz=32)

GitHub+2



## 10. Управление проектом

| Операция     | Команда                                         |
| ------------ | ----------------------------------------------- |
| Запуск       | `sudo docker compose up -d`                     |
| Остановка    | `sudo docker compose down`                      |
| Состояние    | `sudo docker compose ps`                        |
| Все логи     | `sudo docker compose logs -f`                   |
| Логи сервера | `sudo docker compose logs -f qwen38-flash-next` |
| Перезапуск   | `sudo docker compose restart`                   |

Не запускайте одновременно `./start.sh` и `docker compose up`: оба способа используют один контейнер и одни ресурсы GPU. В вашем случае используем только Compose.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://raw.githubusercontent.com\&sz=32)

GitHub+1



## 11. Что важно понимать относительно производительности

Разработчик публикует следующие результаты:

| Показатель                                 | Результат      |
| ------------------------------------------ | -------------- |
| Один пользователь, структурированный ответ | 152,8 ток/с    |
| Один пользователь, код                     | 152,6 ток/с    |
| Восемь пользователей, код, суммарно        | 674,7 ток/с    |
| Холодный prefill                           | 2651 ток/с     |
| Контекст                                   | 262 144 токена |

Но это не гарантированные результаты для любой конфигурации. Они были получены на двух картах с PCIe x16, P2P включён, при лимите питания 180 W на каждую карту. При x4 автор эти показатели не измерял.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://raw.githubusercontent.com\&sz=32)

GitHub+1



## 12. Итоговая последовательность

## Контрольный список установки

0 из 13 выполнено

0%

Проверены PCI ID обеих CMP 170HX — обе 10de:20c2

Secure Boot отключён

Установлен nvidia-driver-615-open версии 615.71.09

Установлены зависимости и kernel headers

Установлен cmpunlocker, настроены две карты

После холодного старта обе карты показывают 65536 MiB

Проверены PCIe Gen2 и ширина соединений

Установлен Docker Engine и Compose

Установлен NVIDIA Container Toolkit

Docker-контейнер видит две GPU

Скачан целевой репозиторий на SSD

Docker Compose запущен, сервис healthy

Проверены /health и тестовый запрос

Мой главный инженерный вывод: для этой сборки сначала необходимо подтвердить, что у вас именно две карты с PCI ID `10de:20c2`. Иначе можно успешно установить драйвер, разблокировать обе карты, установить Docker — и всё равно получить отказ самого проекта из-за недостаточного объёма памяти на одной из GPU.

Также после обновления ядра Linux нужно повторно собрать patched-модули `cmpunlocker` под новое ядро. Сам установщик предупреждает об этом.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://raw.githubusercontent.com\&sz=32)

GitHub



Следующий практически важный шаг — получить от вас вывод `lspci -nn | grep -i NVIDIA` и `nvidia-smi` до разблокировки. По ним можно точно определить, какой вариант CMP 170HX установлен и можно ли запускать именно эту сборку без изменения её требований.
