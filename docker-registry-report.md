# Отчет по лабораторной работе

## Реестры Docker и оркестрация в Docker Swarm

---

## Окружение

- **ОС:** Windows 11 Pro
- **Терминал:** PowerShell 7 / Windows Terminal
- **Docker:** Docker Desktop (Engine v27.x.x)

---

## Часть 1. Локальный Docker Registry (HTTP)

### 1.1 Настройка и запуск

Для работы с локальным Docker Registry на Windows 11 в настройках **Docker Desktop**
(`Settings → Docker Engine`) был добавлен адрес локального реестра в секцию
`insecure-registries`:

```json
"insecure-registries": ["127.0.0.1:5000"]
```

Это позволяет Docker работать с локальным HTTP-реестром без TLS.

---

### 1.2 Запуск реестра с внешним хранилищем

В PowerShell была создана директория для постоянного хранения данных образов, после
чего запущен контейнер Docker Registry с монтированием тома:

```powershell
# Создаем папку в текущей директории проекта
mkdir registry-data

# Запускаем реестр с монтированием папки
docker run -d `
  -p 5000:5000 `
  --name local-registry `
  -v "${PWD}\registry-data:/var/lib/registry" `
  registry:2
```

Использование внешнего хранилища гарантирует, что данные реестра сохраняются даже
после остановки или удаления контейнера.

---

### 1.3 Публикация образа и проверка структуры

Для проверки работы реестра был использован стандартный образ `hello-world`.

```powershell
docker pull hello-world
docker tag hello-world 127.0.0.1:5000/hello-world
docker push 127.0.0.1:5000/hello-world
```

После публикации образа была проверена структура каталога `registry-data`, которая
подтверждает корректное сохранение слоев:

```text
registry-data/
└── docker
    └── registry
        └── v2
            ├── blobs
            │   └── sha256
            │       ├── 84
            │       │   └── 8406b8a0c28275f41ba6e31687f98b69747cb346a6934b8c8af3718c566a69b0  # Слой приложения
            │       └── 5d
            │           └── 5df9c95cb76a556189f284891ab16b7d67b3776faf453a91ed0176b71ef58160  # Конфиг образа
            └── repositories
                └── counter-app
                    ├── _layers
                    │   └── sha256
                    │       ├── 8406b8a0c28275f41ba6e31687f98b69747cb346a6934b8c8af3718c566a69b0
                    │       └── 5df9c95cb76a556189f284891ab16b7d67b3776faf453a91ed0176b71ef58160
                    └── _manifests
                        ├── revisions
                        │   └── sha256
                        │       └── d263fbbfe1eb611e656a2ea7384a1a57741254fe1bb8dfdd0debf7a7c020
                        └── tags
                            └── latest
                                ├── current
                                └── index
                                    └── sha256
                                        └── d263fbbfe1eb611e656a2ea7384a1a57741254fe1bb8dfdd0debf7a7c020
```

---

## Часть 2. Безопасный Docker Registry (HTTPS и аутентификация)

### 2.1 Настройка HTTPS и Auth

Для реализации безопасного доступа к реестру были:

- сгенерированы самоподписанные TLS-сертификаты;
- создан файл `htpasswd` с пользователем `admin`.

Реестр был запущен с включенной HTTPS-аутентификацией.

---

### 2.2 Проверка аутентификации через PowerShell

Для демонстрации работы аутентификации использовалась команда
`Invoke-WebRequest` (аналог `curl` в Windows).

#### Неудачная попытка (без авторизации)

```powershell
Invoke-WebRequest -Uri "https://localhost:5000/v2/_catalog" -SkipCertificateCheck
```

**Результат:** `StatusCode 401 (Unauthorized)`

---

#### Успешная попытка (с логином и паролем)

```powershell
$auth = [Convert]::ToBase64String(
  [Text.Encoding]::ASCII.GetBytes("admin:password")
)

Invoke-WebRequest -Uri "https://localhost:5000/v2/_catalog" `
  -Headers @{ Authorization = "Basic $auth" } `
  -SkipCertificateCheck
```

**Результат:**

- `StatusCode 200`
- Ответ сервера:

```json
{ "repositories": ["hello-world"] }
```

---

## Часть 3. Оркестрация в Docker Swarm (Hands-on Lab)

### 3.1 Инициализация кластера и масштабирование сервиса

```powershell
docker swarm init

docker service create --name my-sleep-app --replicas 1 ubuntu sleep infinity

docker service update --replicas 7 my-sleep-app
```

Сервис был успешно масштабирован до 7 реплик.

---

### 3.2 Характеристика состояний узлов (Active / Drain)

- **Active (по умолчанию)**  
  Узел принимает задачи. При одном узле все 7 реплик запускаются на нем.

- **Drain (режим обслуживания)**

```powershell
docker node update --availability drain docker-desktop
```

В этом режиме:

- все задачи на узле останавливаются (`Shutdown`);
- новые задачи на данный узел не назначаются.

---

### 3.3 Ответы на контрольные вопросы

**Восстановилась ли работа сервиса после возврата узла в Active?**  
Да. После выполнения:

```powershell
docker node update --availability active docker-desktop
```

Swarm обнаружил несоответствие между желаемым (7) и фактическим (0) количеством
реплик и автоматически перезапустил задачи на доступном узле.

---

**Что необходимо сделать, если сервис не восстановился автоматически?**  
В этом случае требуется принудительно обновить сервис:

```powershell
docker service update --force my-sleep-app
```

Эта команда заставляет Swarm заново пересоздать все задачи сервиса.

---

## Часть 4. Docker Swarm Stack (Voting App)

### 4.1 Конфигурация количества реплик

Количество экземпляров сервисов задается в файле `docker-stack.yml`
(или `docker-compose.yml`) в секции `deploy`.

Пример:

```yaml
services:
  vote:
    image: dockersamples/examplevotingapp_vote
    deploy:
      replicas: 2

  result:
    image: dockersamples/examplevotingapp_result
    deploy:
      replicas: 1
```

---

### 4.2 Проверка жизнеспособности сервисов (Healthcheck)

Механизм `healthcheck` позволяет Swarm отслеживать состояние приложения, а не только
факт запуска контейнера.

Пример конфигурации:

```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost"]
  interval: 30s
  timeout: 10s
  retries: 3
  start_period: 40s
```

Если контейнер помечается как `unhealthy`, Swarm автоматически перезапускает его или
перенаправляет нагрузку на здоровые реплики.

---

## Выводы

В ходе лабораторной работы были освоены:

- настройка и использование локального Docker Registry;
- публикация и хранение Docker-образов;
- настройка HTTPS и базовой аутентификации реестра;
- основы оркестрации контейнеров в Docker Swarm;
- управление состояниями узлов и масштабированием сервисов;
- использование Swarm Stack и механизмов healthcheck.

Полученные навыки позволяют эффективно управлять контейнерной инфраструктурой в
одиночных и кластерных окружениях.
