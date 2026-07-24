🛡️ Домашняя SOC-лаборатория: Развертывание SIEM на базе ELK Stack
                                                                                                                                                Автор: [Arhy1368]
                                                                                                                                        Дата создания: [07.07.26]
Цель: Создание домашней лаборатории для изучения процессов централизованного сбора,
анализа и корреляции логов, а также для отработки навыков реагирования на инциденты.

📖 О проекте
Этот проект представляет собой полноценную лабораторную среду, в которой развернута SIEM-система
на базе стека ELK (Elasticsearch, Logstash, Kibana). Основная цель — научиться обнаруживать 
и анализировать различные типы атак в реальном времени на изолированном полигоне. 
Лаборатория полностью воспроизводима и служит стендом для отработки навыков Purple Team.

                                                       🧩 Используемые технологии и инструменты
В проекте использовался следующий стек:

Компонент                  Технология	                                        Назначение
SIEM-платформа	           Elastic Stack (Elasticsearch, Logstash, Kibana)	  Централизованный сбор, хранение и визуализация логов
Агент сбора логов	         Filebeat (Linux)	                                  Сбор системных логов и логов приложений
IDS/IPS	                   Suricata                                           Генерация сетевых событий и обнаружение атак
Виртуализация	             VirtualBox	                                        Запуск виртуальных машин
ОС жертвы + сбор логов	   Ubuntu Server 22.04 LTS	                          Генерация логов ОС и сбор через Filebeat
ОС атакующего	             Kali Linux	                                        Симуляция атак

🏗️ Архитектура стенда

<img width="1919" height="1019" alt="image" src="https://github.com/user-attachments/assets/a6e4f049-d068-4129-a999-175be36aa62c" />

Описание ролей машин:

Машина	                Роль	                    IP-адрес	          Назначение
Ubuntu Server 22.04	    Жертва + Сборщик логов	  192.168.100.20	    Генерация системных логов, установка Filebeat и Suricata
Kali Linux              Атакующий                 192.168.100.99	    Симуляция атак (Nmap, Metasploit, Hydra)

💡 Важно: В данной лабораторной среде Windows не участвует. Сбор логов осуществляется через Filebeat на Ubuntu Server, 
а сам Ubuntu Server является жертвой атак. SIEM-сервер (ELK Stack) размещен на Ubuntu Server.

🛠️ Как развернут стенд: Пошаговое руководство
1. Подготовка виртуальных машин
Созданы две виртуальные машины в VirtualBox:

Ubuntu Server 22.04 LTS — жертва + сборщик логов

<img width="1919" height="1018" alt="image" src="https://github.com/user-attachments/assets/3cc0e8fa-354c-4997-8786-14fed5183821" />

Kali Linux — атакующий

<img width="1919" height="1020" alt="image" src="https://github.com/user-attachments/assets/4c55f125-eb8e-494f-98f7-8b1469dd8da3" />

2. Установка и настройка ELK Stack

На Ubuntu Server 22.04 (жертва + сборщик логов) развернут ELK Stack для централизованного сбора и анализа логов.

                             🐧 Полная настройка Ubuntu Server 22.04 LTS для SOC-лаборатории

                                                      📋 Содержание

1. [Базовая настройка системы](#1-базовая-настройка-системы)
2. [Установка Docker и Docker Compose](#2-установка-docker-и-docker-compose)
3. [Установка ELK Stack (через Docker)](#3-установка-elk-stack-через-docker)
4. [Установка и настройка Filebeat](#4-установка-и-настройка-filebeat)
5. [Установка и настройка Suricata (IDS)](#5-установка-и-настройка-suricata-ids)
6. [Настройка Kibana (первичная)](#6-настройка-kibana-первичная)
7. [Создание правил обнаружения](#7-создание-правил-обнаружения)
8. [Проверка работы](#8-проверка-работы)

                                                1. Базовая настройка системы

```bash
# 1.1 Обновление системы
sudo apt update && sudo apt upgrade -y

# 1.2 Установка базовых утилит
sudo apt install -y curl wget git net-tools htop vim nano \
    ufw fail2ban software-properties-common

# 1.3 Настройка времени (важно для логов)
sudo timedatectl set-timezone Europe/Moscow
sudo timedatectl set-ntp true

# 1.4 Проверка имени хоста (должно быть понятное)
sudo hostnamectl set-hostname siem-lab
hostnamectl

# 1.5 Настройка файла hosts
echo "127.0.0.1 siem-lab" | sudo tee -a /etc/hosts

# 1.6 Перезагрузка (опционально)
sudo reboot
```

---

                                                  2. Установка Docker и Docker Compose

```bash
# 2.1 Установка зависимостей
sudo apt install -y ca-certificates curl gnupg lsb-release

# 2.2 Добавление GPG-ключа Docker
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# 2.3 Добавление репозитория Docker
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg]
https://download.docker.com/linux/ubuntu $(lsb_release -cs)
stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 2.4 Установка Docker
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# 2.5 Добавление пользователя в группу docker
sudo usermod -aG docker $USER

# 2.6 Проверка Docker
docker --version
docker compose version

# 2.7 Автозапуск Docker
sudo systemctl enable docker
sudo systemctl start docker
```

> **Важно:** после добавления в группу docker выйдите из сессии и зайдите заново:
> ```bash
> exit
> # Затем перезайдите по SSH или в консоли
> ```

---

                                               3. Установка ELK Stack (через Docker)

```bash
# 3.1 Клонирование репозитория docker-elk
git clone https://github.com/deviantony/docker-elk.git
cd docker-elk

# 3.2 Настройка переменных окружения (опционально)
cp .env.example .env
# При необходимости отредактируйте .env (пароли, версии)

# 3.3 Запуск контейнеров
docker compose up -d

# 3.4 Проверка статуса контейнеров
docker compose ps

# Ожидаемый результат:
# ✅ docker-elk-elasticsearch-1  Running
# ✅ docker-elk-logstash-1       Running
# ✅ docker-elk-kibana-1         Running

# 3.5 Проверка Elasticsearch
curl -s http://localhost:9200 | head -20

# 3.6 Проверка Kibana
# Откройте в браузере: http://IP_вашего_сервера:5601
# Логин: elastic, Пароль: changeme (или посмотрите в docker-compose.yml)

# 3.7 Просмотр логов контейнеров (если проблемы)
docker compose logs -f
```

---

                                              4. Установка и настройка Filebeat

```bash
# 4.1 Добавление репозитория Elastic
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch |
sudo gpg --dearmor -o /usr/share/keyrings/elastic-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/elastic-keyring.gpg]
https://artifacts.elastic.co/packages/8.x/apt stable main" |
sudo tee /etc/apt/sources.list.d/elastic-8.x.list

# 4.2 Установка Filebeat
sudo apt update
sudo apt install -y filebeat

# 4.3 Резервное копирование конфига
sudo cp /etc/filebeat/filebeat.yml /etc/filebeat/filebeat.yml.bak

# 4.4 Редактирование конфига
sudo nano /etc/filebeat/filebeat.yml


### Содержимое `/etc/filebeat/filebeat.yml`:

yaml
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/syslog
      - /var/log/auth.log
      - /var/log/suricata/eve.json
    json.keys_under_root: true
    json.overwrite_keys: true

output.elasticsearch:
  hosts: ["localhost:9200"]
  username: "elastic"
  password: "changeme"

setup.kibana:
  host: "http://localhost:5601"

# Настройка модулей (опционально)
filebeat.modules:
  - module: system
    syslog:
      enabled: true
    auth:
      enabled: true

bash
# 4.5 Настройка модуля system
sudo filebeat modules enable system

# 4.6 Загрузка дашбордов (опционально)
sudo filebeat setup -e

# 4.7 Запуск Filebeat
sudo systemctl start filebeat
sudo systemctl enable filebeat

# 4.8 Проверка статуса
sudo systemctl status filebeat

# 4.9 Проверка логов Filebeat
sudo journalctl -u filebeat -f -n 50

# 4.10 Проверка входящих логов в Kibana
# Откройте Kibana → Management → Index Patterns → создайте filebeat-*

                                                5. Установка и настройка Suricata (IDS)
bash
# 5.1 Установка Suricata
sudo apt install -y suricata

# 5.2 Проверка версии
suricata --version

# 5.3 Резервное копирование конфига
sudo cp /etc/suricata/suricata.yaml /etc/suricata/suricata.yaml.bak

# 5.4 Редактирование конфига
sudo nano /etc/suricata/suricata.yaml

### Основные изменения в `/etc/suricata/suricata.yaml`:

yaml
# Укажите вашу домашнюю сеть
HOME_NET: "[192.168.100.0/24]"

# Настройка вывода в JSON
outputs:
  - eve-log:
      enabled: yes
      filetype: regular
      filename: eve.json
      types:
        - alert
        - http
        - dns
        - tls

bash
# 5.5 Загрузка правил (ET Open)
sudo suricata-update

# 5.6 Проверка конфига
sudo suricata -T -c /etc/suricata/suricata.yaml

# 5.7 Запуск Suricata
sudo systemctl start suricata
sudo systemctl enable suricata

# 5.8 Проверка статуса
sudo systemctl status suricata

# 5.9 Проверка логов Suricata
sudo tail -f /var/log/suricata/eve.json | head -20

# 5.10 Убедитесь, что Filebeat читает логи Suricata
# В filebeat.yml уже добавлен путь /var/log/suricata/eve.json
sudo systemctl restart filebeat

                                                            6. Настройка Kibana (первичная)

6.1 Вход в Kibana

1. Откройте браузер: `http://IP_вашего_сервера:5601`
2. Логин: `elastic`
3. Пароль: `changeme` (или тот, что указан в `docker-elk/.env`)

6.2 Создание индексных шаблонов

1. Kibana → **Management** → **Stack Management** → **Index Patterns**
2. **Create index pattern**
3. Имя: `filebeat-*` → **Next step**
4. Выберите `@timestamp` → **Create index pattern**

6.3 Проверка входящих логов

1. Kibana → **Analytics** → **Discover**
2. Выберите индекс `filebeat-*`
3. Выберите временной диапазон **Last 15 minutes**
4. Должны появиться логи

                                                           7. Создание правил обнаружения

7.1 Правило для SSH брутфорса

1. Kibana → **Analytics** → **Dashboard** → **Create dashboard**
2. Добавьте визуализацию:
   - **Type:** Lens
   - **Data source:** `filebeat-*`
   - **Filter:** `event.dataset: system.auth AND system.auth.ssh.event: Failed`
   - **Group by:** `source.ip`

7.2 Правило для Suricata (Nmap)

1. **Management** → **Detections & Alerts** → **Create new rule**
2. **Name:** Nmap Scan Detection
3. **Index:** `filebeat-*`
4. **Query:**
   
   event.module: "suricata" AND suricata.eve.alert.category: "Scan"
   
6. **Group by:** `source.ip`
7. **Count:** > 10 за 1 минуту

                                                              8. Проверка работы

8.1 Проверка всех служб

bash
# Статус Docker контейнеров
docker compose -f ~/docker-elk/docker-compose.yml ps

# Статус Filebeat
sudo systemctl status filebeat --no-pager

# Статус Suricata
sudo systemctl status suricata --no-pager

8.2 Проверка логов

bash
# Логи Elasticsearch
docker compose -f ~/docker-elk/docker-compose.yml logs elasticsearch --tail=20

# Логи Filebeat
sudo journalctl -u filebeat -n 20

# Логи Suricata
sudo tail -20 /var/log/suricata/eve.json

8.3 Тестовая атака с Kali

bash
# SSH брутфорс (с Kali)
hydra -l root -P /usr/share/wordlists/rockyou.txt ssh://192.168.100.20

# Nmap сканирование (с Kali)
nmap -sS -p 1-1000 192.168.100.20

8.4 Проверка в Kibana

1. **Discover** → `filebeat-*` → проверьте наличие событий
2. **Dashboard** → проверьте созданные дашборды

                                                               📋 Полный чек-лист

| Шаг | Команда / Действие | Статус |
|-----|-------------------|--------|
| 1 | `sudo apt update && sudo apt upgrade -y` | ⬜ |
| 2 | Установка базовых утилит | ⬜ |
| 3 | Установка Docker и Docker Compose | ⬜ |
| 4 | Клонирование и запуск docker-elk | ⬜ |
| 5 | Проверка Elasticsearch (`curl localhost:9200`) | ⬜ |
| 6 | Проверка Kibana (`http://IP:5601`) | ⬜ |
| 7 | Установка Filebeat | ⬜ |
| 8 | Настройка `/etc/filebeat/filebeat.yml` | ⬜ |
| 9 | Запуск Filebeat (`systemctl start filebeat`) | ⬜ |
| 10 | Установка Suricata | ⬜ |
| 11 | Настройка `/etc/suricata/suricata.yaml` | ⬜ |
| 12 | Запуск Suricata (`systemctl start suricata`) | ⬜ |
| 13 | Создание индексного шаблона в Kibana (`filebeat-*`) | ⬜ |
| 14 | Проверка логов в Discover | ⬜ |
| 15 | Создание правил обнаружения | ⬜ |

                                                     🆘 Частые ошибки и решения

| Ошибка | Решение |
|--------|---------|
| `curl: (7) Failed to connect to localhost port 9200` | `docker compose ps`
 — проверьте, что Elasticsearch запущен |
| `No such file or directory: /var/log/suricata/eve.json` | Suricata не запущен или не создаёт лог.
 Проверьте `sudo systemctl status suricata` |
| Filebeat не отправляет логи | Проверьте `sudo journalctl -u filebeat -f`,
 проверьте пароль в `filebeat.yml` |
| Kibana не открывается | Проверьте порт 5601: `ss -tulpn \| grep 5601` |
| Пароль `changeme` не работает | Посмотрите пароль в `docker-elk/.env`
или выполните `docker exec -it docker-elk-elasticsearch-1 bin/elasticsearch-reset-password -u elastic` |

После выполнения всех шагов у вас будет:
- ✅ ELK Stack в Docker
- ✅ Filebeat для сбора логов
- ✅ Suricata для IDS/IPS
- ✅ Правила обнаружения атак
- ✅ Полноценная SOC-лаборатория на одной Ubuntu

**Проверьте каждый пункт чек-листа и идите дальше!** 🔍

Действия злоумышленника (Kali Linux):

bash
hydra -l root -P /usr/share/wordlists/rockyou.txt ssh://192.168.100.20
Что мы видим в SIEM (Kibana):

В логах Filebeat фиксируются множественные события неудачных входов (SSH).
Правило корреляции в Kibana
настроено на подсчет количества таких событий с одного IP-адреса за короткий промежуток времени.

Результат:
Алерт сработал через 10 секунд, указав атакующий IP-адрес (192.168.100.99)
и целевую учетную запись (root).

📌 Сценарий 2: Сканирование сети Nmap
Цель атаки: Разведка сети перед основной атакой.

Действия злоумышленника (Kali Linux):

bash
nmap -sS -p 1-1000 192.168.100.20
Что мы видим в SIEM (Kibana):

На Ubuntu Server 22.04 настроен Suricata (IDS/IPS). В логах Suricata,
 передаваемых через Filebeat, фиксируются события сканирования портов.
Правило в Kibana выделяет множественные SYN-пакеты с одного IP-адреса за короткое время.

Результат:
Обнаружено сканирование портов, создан алерт с указанием атакующего IP (192.168.100.99).

📌 Сценарий 3: Анализ подозрительной активности
Цель атаки: Обнаружение аномалий в системных логах.

Действия: Мониторинг логов аутентификации и системных событий.

Что мы видим в SIEM (Kibana):

В логах Filebeat фиксируются события, связанные с нестандартной активностью.
Правила корреляции в Kibana настроены на выявление подозрительных паттернов.

📊 Дашборды и визуализация
Главный дашборд SOC
[будет скриншот вашего основного дашборда в Kibana]

Описание:
На дашборде выводятся:

Топ-5 атакующих IP-адресов

Общее количество обнаруженных алертов по категориям

Карта мира с геолокацией атак (если настроена)

Дашборд по SSH-атакам
[будет скриншот дашборда с логами SSH]

Описание:
На дашборде отображаются:

Количество неудачных SSH-входов

Топ-5 IP-адресов атакующих

Частота атак по времени

📈 Результаты и выводы
Что получилось:
SIEM-система успешно собирает логи со всех ключевых узлов сети. Настроенные правила корреляции
позволяют в реальном времени выявлять:

✅ Брутфорс-атаки на SSH

✅ Сканирование сети Nmap

✅ Подозрительную активность в системных логах

Создан базовый дашборд для мониторинга.

Что дала лаборатория:
Получены практические навыки работы с Elastic Stack, написания правил корреляции и анализа логов.
Лаборатория является основой для дальнейшего изучения Threat Hunting.

💡 Дальнейшие шаги по развитию проекта
Интеграция с TheHive для управления инцидентами (SOAR)

Настройка оповещений в Telegram/Slack

Развертывание агентов на большее количество машин

Добавление сценариев с DDoS-атаками и их анализ

Настройка GeoIP для определения геолокации атакующих

Интеграция с VirusTotal для обогащения данных

📋 Сводная таблица ролей машин
Машина	          ОС	                 Роль	                    IP-адрес	          Собираемые логи
Жертва + сборщик	Ubuntu Server 22.04	 Генерация и сбор логов	  192.168.100.20	    Системные логи, логи Suricata (Filebeat)
Атакующий	        Kali Linux	         Симуляция атак	          192.168.100.99	    —
