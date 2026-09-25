# Osint-station
Система в автоматическом режиме осуществляет мониторинг утечек, выявление фишинговой инфраструктуры (тайпосквоттинг) и поиск мошеннических ресурсов в мессенджерах и скрытом сегменте интернета (Даркнет).
Комплексная OSINT-Станция: Мониторинг внешних угроз бренда и защита цифрового периметра

> Краткое описание архитектуры
> Проект представляет собой развернутый на удаленном сервере **Ubuntu 24.04 LTS** аппаратно-программный комплекс внешней разведки (Threat Intelligence). Система в автоматическом режиме осуществляет мониторинг утечек, выявление фишинговой инфраструктуры (тайпосквоттинг) и поиск мошеннических ресурсов в мессенджерах и скрытом сегменте интернета (Даркнет).

---

## 🛠️ Раздел 0: Решение инфраструктурных проблем (Troubleshooting)

В процессе развертывания комплекса на VPS хостинга были выявлены и устранены критические сетевые аномалии операционной системы.

### 1. Сбой DNS-резолвера (Сеть Linux)
**Проблема:** Ошибка `Name or service not known` при попытке утилит `curl` / `wget` достучаться до внешних API. Система автоматически затирала конфигурационный файл.
**Решение:** Принудительное назначение стабильного публичного DNS-сервера Google:
```bash
echo "nameserver 8.8.8.8" > /etc/resolv.conf
# Верификация работоспособности сети
ping -c 3 google.com
```

### 2. Искажение спецсимволов буфером консоли Bash
**Проблема:** При вставке многострочного Python-кода терминал обрезал символы слэша (`/`), переноса строки (`\n`) и триггерил ошибку истории команд (`-bash: !: event not found`).
**Решение:** Переход на чистую потоковую запись через изолированный интерпретатор Python с математической заменой опасных символов через таблицу ASCII (`chr(47)` — слэш, `chr(10)` — перевод строки).

---

## 🌐 Раздел 1: Модуль Веб-Периметра (Инвентаризация доменов)

Модуль осуществляет генерацию потенциально опасных созвучных доменов (атаки класса Homoglyph, Bitsquatting, Insertion) для выявления фишинговых сайтов-клонов, крадущих трафик и депозиты пользователей.

### Технический стек и команды
Используется CLI-утилита `dnstwist`, опрашивающая глобальные реестры и Certificate Transparency logs.

```bash
# Установка пакета
sudo apt update && sudo apt install dnstwist -y

# Запуск сканирования периметра с сохранением в JSON
dnstwist --registered target_domain.com --format json > /root/domain_scan_results.json
```

### 🔍 Методология фильтрации ложных срабатываний (False Positives)
В ходе аудита гемблинг-платформ важно разделять официальные зеркала SEO-отдела от реальных угроз:
* **Легитимный контур:** Домены, находящиеся за официальной прокси-защитой (например, Cloudflare), имеющие корпоративные IP-адреса компании и настроенный редирект (301/302).
* **Враждебный контур:** Домены на сторонних NS-серверах (Namecheap, Spaceship) с чужими IP-адресами. Особый приоритет — наличие **MX-записей (почтовых серверов)**, что указывает на подготовку к рассылке фишинга от имени бренда.

---

## 💬 Раздел 2: Модуль Мониторинга Мессенджеров (Telegram OSINT)

Поскольку стандартные боты заблокированы политиками Telegram на глобальный поиск, модуль реализован в виде **UserBot-скрипта** на фреймворке `Telethon` (Python), имитирующего действия реального пользователя.

### 🐍 Исходный код мультиязычного парсера (`/root/tg_osint.py`)

```Python import asyncio
import csv
from telethon import TelegramClient
from telethon.tl.functions.contacts import SearchRequest
from telethon.tl.functions.channels import GetFullChannelRequest

Получение ключей авторизации из переменных окружения (безопасный метод)
import os  
API_ID = os.environ.get("TG_API_ID")  
API_HASH = os.environ.get("TG_API_HASH")

# Массив ключевых слов для циклической проверки периметра
KEYWORDS = ["Brand Casino", "Brand Money", "Бренд Казино", "Бренд Зеркало", "Бренд Поддержка"] 

async def main():
    async with TelegramClient('osint_session', API_ID, API_HASH) as client:
        print("[*] Запуск циклического сканирования Telegram API...")
        found_chats = {}
        
        for query in KEYWORDS:
            try:
                result = await client(SearchRequest(q=query, limit=50))
                for chat in result.chats:
                    if chat.id not in found_chats: 
                        found_chats[chat.id] = chat
                await asyncio.sleep(2)  # Защита от FloodWait/RateLimiting
            except Exception as e:
                print(f"[!] Ошибка запроса по ключу '{query}': {e}")
        
        # Экспорт результатов в CSV-отчет
        report_path = '/root/tg_report.csv'
        with open(report_path, 'w', newline='', encoding='utf-8') as f:
            writer = csv.writer(f)
            writer.writerow(['Type', 'Title', 'Username', 'ID', 'Participants', 'About'])
            
            for chat_id, chat in found_chats.items():
                title = chat.title
                username = f"@{chat.username}" if chat.username else "Private"
                members, about = "N/A", "N/A"
                chat_type = "Channel" if hasattr(chat, 'broadcast') else "Group/Chat"
                
                try:
                    full_info = await client(GetFullChannelRequest(channel=chat))
                    members = full_info.full_chat.participants_count
                    about = full_info.full_chat.about
                except: pass
                
                writer.writerow([chat_type, title, username, chat_id, members, about])
                print(f"[{chat_type}] {title} | {username} | Участников: {members}")
                
        print(f"\n[+] Сбор завершен. Полный отчет сохранен в {report_path}")

asyncio.run(main())
```

### 🚨 Матрица классификации угроз мессенджеров


| Класс риска | Тип ресурса | Индикатор угроз | Сценарий атаки / Ущерб | Действие |
| :--- | :--- | :--- | :--- | :--- |
| 🔴 **КРИТИЧЕСКИЙ** | Фишинговый канал | Накрученная аудитория (>1000), маскировка под «Официальное зеркало» | Публикация поддельных ссылок, увод депозитов, кража платежных данных игроков. | Немедленный Abuse-takedown |
| 🟡 **СРЕДНИЙ** | Аффилейт-паразит | Использование логотипов компании для сбора трафика | Перехват релевантных клиентов из глобального поиска, слив на чужие реферальные ссылки. | Жалоба на нарушение прав ТМ |
| 🟢 **НИЗКИЙ** | Киберсквоттинг | Пустые приватные каналы с 1 участником | Превентивный захват красивых юзернеймов под будущие фейк-чаты техподдержки. | Внесение в черный список мониторинга |

---

## 🕶️ Раздел 3: Модуль Разведки Даркнета (Теневой контур)

Модуль предназначен для сканирования скрытых хакерских форумов и агрегаторов утечек на предмет продажи доступов к админкам компании (Initial Access Brokerage) или сливов баз данных клиентов (PII).

### Настройка Tor-шлюза (CLI)
Для оптимизации RAM (1.5 ГБ) Tor работает исключительно в фоновом режиме в виде системного демона SOCKS5-прокси (порт 9050):
```bash
sudo apt install tor -y
sudo systemctl enable tor --now
```

### 🐍 Исходный код парсера баз данных Даркнета (`/root/ahmia_search.py`)
Скрипт оптимизирован для работы через стабильное API поисковой системы Ahmia, индексирующей теневые сайты сети `.onion`. В коде полностью исключены опасные символы переноса строки и слэши.

```python
import requests, re, time, urllib.parse
KEYWORDS = ["Dragon Casino", "Dragon Money", "Драгон Казино", "drgn59"]
print("[*] Запуск стабильного поиска по базам Даркнета (API Ahmia)...")
print("-" * 60)
for kw in KEYWORDS:
    print(f"[*] Ищу упоминания для: '{kw}'...")
    try:
        url = "https:" + "//" + "ahmia.fi" + "/search/?q=" + urllib.parse.quote(kw)
        res = requests.get(url, headers={"User-Agent": "Mozilla/5.0"}, timeout=15)
        if res.status_code == 200:
            links = list(set(re.findall(r"[a-z2-7]{16,56}\.onion", res.text)))
            if links:
                print(f"[+] УСПЕХ! Найдено скрытых сайтов: {len(links)}")
                out_file = "/root/onion_" + str(kw.replace(" ", "_")) + ".txt"
                with open(out_file, "w") as file:
                    for l in links: file.write("http://" + str(l) + chr(10))
            else: print("[-] No matches found in Darknet base.")
        else: print("[!] Server error, code: " + str(res.status_code))
    except Exception as e: print("[!] Network error: " + str(e))
    time.sleep(2)
print("[+] Darknet scanning completed")
```

> Особенности интерпретации результатов (Negative Result)
> Если в текстовые файлы отчетов попадает исключительно адрес `juhanurmihxlp77nkq...onion` — это легитимный адрес самого поисковика Ahmia, находящийся в коде веб-страницы. Отсутствие других скрытых ссылок означает **отрицательный результат утечек (периметр чист)**, что является ключевым показателем безопасности бренда на дату проверки.
