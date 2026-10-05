# 🎬 ffmpeg_serviio — обёртка над FFmpeg с аппаратным ускорением VAAPI и кэшем

**Прозрачная замена `/usr/bin/ffmpeg` для [Serviio Media Server](https://www.serviio.org/)**, добавляющая:

- 🚀 **Аппаратное ускорение** через VAAPI (Intel Quick Sync / AMD AMF)
- 💾 **Кэш готовых файлов** — повторные запросы отдаются мгновенно
- 🔄 **Каскадный fallback** — если не получилось на GPU, пробуем CPU
- 📝 **Двойное логирование** — в файл и в системный журнал (syslog/journald)
- 🛡️ **Корректная обработка** путей с пробелами и `-c:v copy` (remux)

---

## 📋 Возможности

| Возможность | Описание |
|---|---|
| **VAAPI-транскодинг** | Используется `/dev/dri/renderD128` для кодирования в `h264_vaapi` |
| **Каскадный fallback** | 4 уровня: HW+субтитры → HW → SW+субтитры → SW |
| **Кэш результатов** | Готовые файлы складываются в `/fast/cache/converted` |
| **Stream copy** | Команды с `-c:v copy` пробрасываются в ffmpeg без изменений |
| **Syslog-интеграция** | Все события пишутся через `logger` с тегом `ffmpeg_serviio` |
| **Диагностика** | stderr ffmpeg автоматически попадает в лог |

---

## ⚙️ Требования

- **ОС:** Linux (проверено на Debian/Ubuntu/Proxmox)
- **GPU:** Intel или AMD с поддержкой VAAPI
- **Пакеты:**
  ```bash
  apt install ffmpeg libva-drm2 vainlog acl
  ```
- **Пользователь Serviio** должен входить в группы `render` и `video`:
  ```bash
  usermod -aG render,video serviio
  ```
- **Права на устройства:**
  ```bash
  ls -la /dev/dri/renderD128 /dev/dri/card0
  ```

Проверить работоспособность VAAPI:
```bash
vainfo
ffmpeg -hwaccel vaapi -hwaccel_device /dev/dri/renderD128 -i input.mkv -f null -
```

---

## 📦 Установка

### 1. Клонирование репозитория

```bash
git clone https://github.com/rajven/ffmpeg_serviio.git
cd ffmpeg_serviio
```

### 2. Создание рабочих каталогов

```bash
sudo mkdir -p /var/log/ffmpeg /fast/cache/converted
sudo chown serviio:media /var/log/ffmpeg /fast/cache/converted
sudo chmod 755 /var/log/ffmpeg /fast/cache/converted
```

> Замените `serviio:media` на реального пользователя и группу вашего Serviio.


### 3. Установка

В стартовом скрипте /opt/serviio/bin/serviio.sh добавить "-Dffmpeg.location=/usr/local/bin/ffmpeg_serviio" в SERVIIO_JAVA_OPTS

# Setup Serviio specific properties
SERVIIO_JAVA_OPTS="-Djava.net.preferIPv4Stack=true -Djava.awt.headless=true -Dorg.restlet.engine.loggerFacadeClass=org.restlet.ext.slf4j.Slf4jLoggerFacade -Dderby.system.home=$SERVIIO_HOME/library -Dserviio.home=$SERVIIO_HOME -Dffmpeg.location=/usr/local/bin/ffmpeg_serviio -Ddcraw.location=dcraw"

```bash

# Install the wrapper
sudo cp ffmpeg_serviio /usr/local/bin/
sudo chmod 755 /usr/local/bin/ffmpeg_serviio
sudo chown root:root /usr/local/bin/ffmpeg_serviio
```

### 4. Перезапуск Serviio

```bash
sudo systemctl restart serviio
```

---

## 🔧 Настройка Serviio

В консоли Serviio (**Transcoding** → **Video**) убедитесь, что:

- ✅ Включено транскодирование
- ✅ Указан правильный профиль устройства
- ✅ Путь к кэшу транскодинга совпадает с `SERVIIO_CACHE_DIR` в скрипте

По умолчанию в скрипте:
```bash
SERVIIO_CACHE_DIR="/fast/cache/serviio/Serviio"
```

---

## 🔄 Как это работает

```
┌─────────────────────────────────────────────────────────┐
│  Serviio вызывает: ffmpeg [args]                        │
└────────────────────────┬────────────────────────────────┘
                         ▼
              ┌──────────────────────┐
              │ Это Serviio-запрос?  │
              │ (путь содержит       │
              │  SERVIIO_CACHE_DIR)  │
              └──────────┬───────────┘
                    yes /        \ no
                   ▼              ▼
          ┌──────────────┐   ┌──────────────────┐
          │ -c:v copy?   │   │ Запуск ffmpeg    │
          └──────┬───────┘   │ без изменений    │
            yes /     \ no   └──────────────────┘
               ▼        ▼
     ┌────────────┐  ┌──────────────────┐
     │ Прямой     │  │ Проверка кэша    │
     │ remux      │  │ /fast/cache/     │
     └────────────┘  └────────┬─────────┘
                         hit /     \ miss
                           ▼         ▼
                    ┌──────────┐  ┌─────────────────────┐
                    │ Копия    │  │ Каскадный fallback: │
                    │ из кэша  │  │ 1. HW + субтитры    │
                    └──────────┘  │ 2. HW               │
                                  │ 3. SW + субтитры    │
                                  │ 4. SW               │
                                  └──────────┬──────────┘
                                             ▼
                                  ┌──────────────────────┐
                                  │ Сохранение в кэш     │
                                  │ /fast/cache/converted│
                                  └──────────────────────┘
```

---

## 📝 Логирование

Скрипт пишет события в **два места одновременно**:

### Файловый лог
```
/var/log/ffmpeg/ffmpeg_serviio.log
```

### Системный журнал (syslog/journald)
```bash
# Все записи скрипта
journalctl -t ffmpeg_serviio -f

# Только ошибки
journalctl -t ffmpeg_serviio -p err

# Предупреждения и выше
journalctl -t ffmpeg_serviio -p warning

# За последний час
journalctl -t ffmpeg_serviio --since "1 hour ago"
```

### Уровни логирования

| Уровень | Когда используется |
|---|---|
| `info` | Запуск/завершение, успешные операции, cache hit |
| `warning` | Fallback на другой режим, файл уже в кэше |
| `error` | Недостаточно прав, сбой транскодирования |
| `debug` | Пути, команды, группы устройств |

> Для включения `debug` в rsyslog добавьте в `/etc/rsyslog.conf`:
> ```
> :syslogtag, isequal, "ffmpeg_serviio:" /var/log/ffmpeg_serviio_debug.log
> ```

---

## 📂 Структура каталогов

```
/var/log/ffmpeg/
├── ffmpeg_serviio.log      # Основной лог скрипта
└── transcoding.log         # Список обработанных файлов

/fast/cache/
├── serviio/Serviio/        # Временные файлы транскодинга Serviio
└── converted/              # Кэш готовых файлов (долгосрочный)
```

---

## 🩺 Диагностика

### Serviio не запускает видео
```bash
# Проверить последние ошибки
journalctl -t ffmpeg_serviio -p err --since "10 min ago"

# Проверить права на GPU
ls -la /dev/dri/renderD128
groups $(ps -o user= -p $(pgrep -f serviio))
```

### Ошибка `exit code: 254`
Обычно означает проблему с путями. Проверьте:
- Права на исходный файл
- Свободное место на `/fast`
- Корректность монтирования сетевых ресурсов

### GPU не используется
```bash
# Мониторинг Intel GPU
sudo apt install intel-gpu-tools
sudo intel_gpu_top

# Или через sysfs
cat /sys/class/drm/card0/device/gpu_busy_percent
```

### Проверка, что обёртка работает
```bash
# Должен вывести PID скрипта, а не ffmpeg.real
ps aux | grep ffmpeg
```

---

## 🛠️ Кастомизация

Основные параметры в начале скрипта:

```bash
LOG_DIR="/var/log/ffmpeg"                    # Каталог логов
SERVIIO_CACHE_DIR="/fast/cache/serviio/Serviio"  # Временный кэш Serviio
CACHE_DIR="/fast/cache/converted"            # Долговременный кэш
```

Алгоритмы кодирования задаются в функции `build_command`:
- `hardware_with_subs` / `hardware_no_subs` — VAAPI-режимы
- `software_with_subs` / `software_no_subs` — libx264-режимы

---

## ⚠️ Известные ограничения

- Скрипт рассчитан на **VAAPI** (Intel/AMD). Для NVIDIA NVENC нужна адаптация.
- Кэш не имеет автоматической очистки — настройте `cron` или `tmpwatch`.
- При большом количестве одновременных транскодингов возможен конфликт имён в кэше (по имени исходного файла).

### Пример cron для очистки старого кэша
```bash
# Удаление файлов старше 30 дней
0 3 * * * find /fast/cache/converted -type f -mtime +30 -delete
```

---

## 📄 Лицензия

MIT License. Используйте на свой страх и риск.

---

## 🤝 Вклад

Pull request'ы приветствуются! Перед отправкой:
1. Протестируйте на реальном transcoding-запросе
2. Проверьте, что логи читаемы
3. Обновите README при изменении поведения

---

## 🙏 Благодарности

- [Serviio](https://www.serviio.org/) — отличный медиа-сервер
- [FFmpeg](https://ffmpeg.org/) — швейцарский нож видео
- Сообществу [Linux VAAPI](https://trac.ffmpeg.org/wiki/Hardware/QuickSync) за документацию
