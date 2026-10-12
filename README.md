Приложение состоит из двух сервисов:

- frontend: nginx отдаёт HTML и перенаправляет запросы `/api/` на backend;
- backend: Python возвращает JSON с сообщением и именем контейнера.

## Статус

Приложение проверено через Docker Compose на Mac:
страница открывается, запрос к бэкенду возвращает сообщение
«Привет от бэкенда!» и hostname.

Для KVM подготовлен план создания Linux VM и порядок проверки.
Создание VM и запуск приложения внутри неё не выполнялись.

## Запуск на Mac

Требуется запущенный Docker Desktop.

При первом скачивании репозитория:

```bash
git clone --branch lab-2 https://github.com/Debade41/SDA-lab-1.git
cd SDA-lab-1
```

Из корня репозитория выполнить:

```bash
docker compose config --quiet
docker compose up -d --build --wait
docker compose ps
```

Открыть [приложение](http://localhost:8080).

Схема локального подключения:

```text
Ноутбук → localhost:8080 → nginx frontend:80 → backend:8000
```

Порт frontend опубликован на ноутбуке.
Backend доступен внутри сети Docker по имени `backend`.

## Проверка приложения

Проверить конфигурацию работающего nginx:

```bash
docker compose exec -T frontend nginx -t
```

Ожидаются сообщения `syntax is ok` и `test is successful`.

Проверить запрос через frontend к backend:

```bash
curl -fsS http://localhost:8080/api/message
```

Ожидается успешный ответ с JSON, содержащим:

- `message`: «Привет от бэкенда!»;
- `hostname`: непустое имя контейнера backend.

В браузере должны отображаться те же сообщение и hostname.
Это подтверждает получение данных бэкенда страницей.

В выводе `docker compose ps` оба сервиса должны быть запущены,
а backend должен иметь состояние `healthy`.

Посмотреть журналы:

```bash
docker compose logs --tail=80 frontend backend
```

Остановить приложение:

```bash
docker compose down
```

## Ограничение KVM на ноутбуке

Среда выполненной диагностики:

- macOS 26.6.2;
- процессор Apple M2;
- архитектура arm64;
- `kern.hv_support: 1`;
- `/dev/kvm` отсутствует.

Поддержка виртуализации Mac не означает наличие KVM.
KVM является механизмом виртуализации Linux.

[Результаты диагностики](docs/host-diagnostics.txt).

[План создания VM на подходящем Linux-хосте](docs/kvm-plan.md).

## Предлагаемая проверка приложения внутри KVM VM

Этот раздел описывает проверку на Linux-хосте.
Успешный запуск в KVM не подтверждён.

Предполагается Linux-хост с KVM и гостевая Ubuntu Server 24.04 LTS.
Создание VM описано в плане выше.

Перед выполнением команд заменить:

- `LINUX_USER` — пользователем Linux-хоста;
- `LINUX_HOST` — адресом Linux-хоста;
- `VM_USER` — пользователем VM, обозначенным как `[NAME]` в плане;
- `VM_IP` — адресом VM.

У пользователя VM должны быть права sudo.
Необходимо настроить SSH-доступ с ноутбука через Linux-хост
и доступ VM к интернету для установки пакетов и сборки образов.

### 1. Доставить приложение с ноутбука

В терминале ноутбука, из корня репозитория:

```bash
cd ~/SDA-lab-1
ssh -J LINUX_USER@LINUX_HOST VM_USER@VM_IP 'mkdir -p ~/sda-lab2'
scp -r -J LINUX_USER@LINUX_HOST backend frontend compose.yaml VM_USER@VM_IP:sda-lab2/
ssh -J LINUX_USER@LINUX_HOST VM_USER@VM_IP
```

### 2. Установить инструменты и запустить приложение в VM

Следующие команды выполнить внутри VM:

```bash
sudo apt update
sudo apt install -y docker.io docker-compose-v2 docker-buildx curl
sudo systemctl enable --now docker

cd ~/sda-lab2
sudo docker compose config --quiet
sudo docker compose up -d --build --wait
sudo docker compose ps
sudo docker compose exec -T frontend nginx -t
curl -fsS http://127.0.0.1:8080/api/message
```

Ожидаются работающие контейнеры, backend в состоянии `healthy`,
успешная проверка nginx и JSON с сообщением и hostname.

### 3. Проверить доступ с ноутбука

Сеть libvirt `default` использует NAT.
Для доступа к приложению открыть SSH-туннель
в отдельном терминале ноутбука:

```bash
ssh -N -J LINUX_USER@LINUX_HOST -L 127.0.0.1:8081:127.0.0.1:8080 VM_USER@VM_IP
```

Оставить терминал с туннелем открытым.
Порт 8081 выбран отдельно от локального приложения на порту 8080.

Открыть [приложение внутри VM](http://localhost:8081).

В другом терминале ноутбука проверить:

```bash
curl -fsS http://localhost:8081/api/message
```

Страница и API должны показывать сообщение и hostname
бэкенда, запущенного внутри VM.

Схема подключения:

```text
Ноутбук localhost:8081
  → SSH-туннель через Linux-хост
  → KVM VM localhost:8080
  → nginx frontend:80
  → backend:8000
```

Для подтверждения проверки сохранить вывод команд,
состояние контейнеров и снимок страницы приложения.
