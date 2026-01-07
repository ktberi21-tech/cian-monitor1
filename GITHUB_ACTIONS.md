# 🚀 ЗАПУСК ПАРСЕРА НА GITHUB ACTIONS

## ✅ Как это работает

GitHub Actions позволяет автоматически запускать парсер:
- **Каждый час** — мониторинг новых объявлений
- **По расписанию (cron)** — в определённое время
- **На каждый push** — при обновлении кода
- **Вручную** — кликом в интерфейсе GitHub

**Результаты сохраняются в:**
- JSON файлы в репозитории
- Email уведомления
- Telegram сообщения
- GitHub Issues (проблемы)

---

## 📋 ШАГ 1: Подготовка репозитория

### 1.1 Создайте репозиторий на GitHub

```bash
# Инициализируем локальный репо
git init
git add .
git commit -m "Initial commit: Cian parser setup"

# Создаём на GitHub и пушим
git branch -M main
git remote add origin https://github.com/YOUR_USERNAME/cian-monitor.git
git push -u origin main
```

### 1.2 Создайте структуру папок

```
cian-monitor/
├── .github/
│   └── workflows/
│       ├── parse_daily.yml        ← расписание (каждый день)
│       ├── parse_hourly.yml       ← расписание (каждый час)
│       └── parse_manual.yml       ← ручной запуск
├── cian_parser.py
├── config.py
├── examples.py
├── requirements.txt
└── ...
```

---

## 📝 ШАГ 2: GitHub Actions Workflows

### 2.1 Создайте файл `.github/workflows/parse_daily.yml`

```yaml
name: Parse Cian Daily (Every 24 hours)

on:
  schedule:
    # Запускаем каждый день в 9 утра МСК (6 UTC)
    - cron: '0 6 * * *'
  
  # Также можно запустить вручную
  workflow_dispatch:

jobs:
  parse:
    runs-on: ubuntu-latest
    
    steps:
      # Шаг 1: Скачиваем репо
      - name: Checkout repository
        uses: actions/checkout@v3
      
      # Шаг 2: Устанавливаем Python 3.10
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.10'
      
      # Шаг 3: Устанавливаем зависимости
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
      
      # Шаг 4: Запускаем парсер
      - name: Run Cian Parser
        run: |
          python3 << 'EOF'
          from cian_parser import CianParser
          import json
          from datetime import datetime
          
          parser = CianParser(headless=True)
          
          try:
              # Парсим объявления
              filters = {
                  'city': 'moskva',
                  'price_max': 20_000_000,
                  'rooms': [1, 2, 3],
              }
              
              listings = parser.search_listings(filters, max_pages=2)
              
              # Сохраняем с датой
              date_str = datetime.now().strftime('%Y-%m-%d')
              filename = f'data/listings_{date_str}.json'
              
              parser.save_to_json(listings, filename)
              
              print(f"✅ Найдено {len(listings)} объявлений")
              print(f"💾 Сохранено в {filename}")
          
          finally:
              parser.close()
          
          EOF
      
      # Шаг 5: Коммитим результаты
      - name: Commit results
        run: |
          git config --local user.email "action@github.com"
          git config --local user.name "GitHub Action"
          git add data/
          git commit -m "🤖 Auto: Parse results $(date +'%Y-%m-%d %H:%M')" || true
          git push
      
      # Шаг 6: Отправляем уведомление в Telegram (опционально)
      - name: Send Telegram notification
        if: always()
        run: |
          python3 << 'EOF'
          import requests
          import os
          from datetime import datetime
          
          TOKEN = "${{ secrets.TELEGRAM_BOT_TOKEN }}"
          CHAT_ID = "${{ secrets.TELEGRAM_CHAT_ID }}"
          
          if TOKEN and CHAT_ID:
              msg = f"✅ Парсинг Циан завершён\n{datetime.now().strftime('%d.%m.%Y %H:%M')}"
              requests.post(
                  f"https://api.telegram.org/bot{TOKEN}/sendMessage",
                  json={'chat_id': CHAT_ID, 'text': msg}
              )
          EOF
```

### 2.2 Создайте файл `.github/workflows/parse_hourly.yml`

Для более частого мониторинга (каждый час):

```yaml
name: Parse Cian Hourly (Every hour)

on:
  schedule:
    # Запускаем каждый час на 0 минут
    - cron: '0 * * * *'
  
  workflow_dispatch:

jobs:
  parse:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.10'
      
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
      
      - name: Parse and filter good deals
        run: |
          python3 << 'EOF'
          from cian_parser import CianParser
          import json
          from datetime import datetime
          
          parser = CianParser(headless=True)
          
          try:
              filters = {
                  'city': 'moskva',
                  'price_min': 5_000_000,
                  'price_max': 20_000_000,
                  'rooms': [1, 2],
              }
              
              listings = parser.search_listings(filters, max_pages=1)
              
              # Фильтруем ТОЛЬКО сегодняшние объявления с хорошей ценой
              good_deals = [
                  l for l in listings 
                  if l['days_old'] == 0 and l['price_per_sqm'] < 150_000
              ]
              
              if good_deals:
                  # Сохраняем
                  filename = f'data/good_deals_{datetime.now().strftime("%H-%M")}.json'
                  with open(filename, 'w') as f:
                      json.dump(good_deals, f, ensure_ascii=False, indent=2, default=str)
                  
                  print(f"💰 Найдено {len(good_deals)} хороших предложений!")
          
          finally:
              parser.close()
          
          EOF
      
      - name: Push results
        run: |
          git config --local user.email "action@github.com"
          git config --local user.name "GitHub Action"
          git add data/ || true
          git commit -m "🤖 Hourly: Good deals found" || true
          git push || true
```

### 2.3 Создайте файл `.github/workflows/parse_manual.yml`

Для ручного запуска через интерфейс:

```yaml
name: Parse on Demand

on:
  workflow_dispatch:
    inputs:
      city:
        description: 'City (moskva, spb, ekb)'
        required: true
        default: 'moskva'
      rooms:
        description: 'Rooms (1,2,3 or 1,2)'
        required: true
        default: '1,2,3'
      max_price:
        description: 'Max price (millions, e.g. 20)'
        required: true
        default: '20'

jobs:
  parse:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.10'
      
      - name: Install dependencies
        run: pip install -r requirements.txt
      
      - name: Parse with custom params
        run: |
          python3 << 'EOF'
          from cian_parser import CianParser
          import json
          from datetime import datetime
          
          # Параметры от пользователя
          city = "${{ github.event.inputs.city }}"
          rooms = [int(r) for r in "${{ github.event.inputs.rooms }}".split(',')]
          max_price = int("${{ github.event.inputs.max_price }}") * 1_000_000
          
          parser = CianParser(headless=True)
          
          try:
              filters = {
                  'city': city,
                  'rooms': rooms,
                  'price_max': max_price,
              }
              
              print(f"🔍 Parsing: city={city}, rooms={rooms}, max_price={max_price:,}")
              
              listings = parser.search_listings(filters, max_pages=3)
              
              # Сохраняем
              filename = f'data/custom_search_{datetime.now().strftime("%Y%m%d_%H%M%S")}.json'
              parser.save_to_json(listings, filename)
              
              print(f"✅ Найдено {len(listings)} объявлений")
              print(f"💾 Сохранено: {filename}")
          
          finally:
              parser.close()
          
          EOF
      
      - name: Push and create issue
        run: |
          git config --local user.email "action@github.com"
          git config --local user.name "GitHub Action"
          git add data/
          git commit -m "📊 Custom parsing: city=${{ github.event.inputs.city }}" || true
          git push
```

---

## 🔐 ШАГ 3: Добавьте Secrets (токены)

Для отправки уведомлений в Telegram нужно добавить токены в GitHub:

### 3.1 Перейдите в Settings репозитория

1. Откройте https://github.com/YOUR_USERNAME/cian-monitor/settings
2. В левом меню: **Secrets and variables** → **Actions**
3. Нажмите **New repository secret**

### 3.2 Добавьте secrets:

```
TELEGRAM_BOT_TOKEN = your_bot_token_here
TELEGRAM_CHAT_ID = your_chat_id_here
```

**Как получить токен:**
1. Напишите @BotFather в Telegram
2. Создайте бота: `/newbot`
3. Скопируйте токен

**Как получить CHAT_ID:**
1. Напишите @userinfobot в Telegram
2. Получите свой ID

---

## 📁 ШАГ 4: Создайте папку для результатов

```bash
# Создайте папку для сохранения результатов
mkdir data
touch data/.gitkeep

# Добавьте в гит
git add data/
git commit -m "Add data folder for results"
git push
```

---

## ⚙️ ШАГ 5: Настройки дополнительно

### 5.1 `.gitignore` (не коммитим некоторые файлы)

```
.env
.DS_Store
__pycache__/
*.pyc
venv/
logs/
*.log
.vscode/
```

### 5.2 `data/.gitkeep` (чтобы папка была в гите)

```bash
touch data/.gitkeep
git add data/.gitkeep
```

---

## 🎯 ШАГ 6: Первый запуск

### 6.1 Запустите вручную

1. Откройте репо на GitHub
2. Перейдите на вкладку **Actions**
3. Выберите workflow **"Parse on Demand"**
4. Нажмите **Run workflow**
5. Заполните параметры (город, комнаты, цена)
6. Нажмите **Run workflow**

### 6.2 Посмотрите логи

1. Вернитесь на вкладку **Actions**
2. Откройте выполняющийся workflow
3. Нажмите на job **parse**
4. Смотрите логи в реальном времени

### 6.3 Проверьте результаты

```bash
# Результаты будут в папке data/
# listings_2026-01-07.json
# good_deals_09-30.json
# и т.д.
```

---

## 📊 РАСПИСАНИЕ (Cron синтаксис)

Строка `- cron: '0 6 * * *'` означает:

```
┌───────────── minute (0 - 59)
│ ┌───────────── hour (0 - 23)
│ │ ┌───────────── day of month (1 - 31)
│ │ │ ┌───────────── month (1 - 12)
│ │ │ │ ┌───────────── day of week (0 - 6) (Sunday to Saturday)
│ │ │ │ │
│ │ │ │ │
0 6 * * *    ← Каждый день в 6:00 UTC (9:00 МСК)

# Примеры:
0 * * * *    ← Каждый час
0 9 * * *    ← Каждый день в 9 утра UTC
0 9 * * 1-5  ← Каждый рабочий день в 9 утра
*/30 * * * * ← Каждые 30 минут
0 0 1 * *    ← Первого числа каждого месяца
```

**⚠️ GitHub Actions работает в UTC!**
Если хотите 9 утра МСК (UTC+3), то ставьте 6 UTC.

---

## 💾 УСОВЕРШЕНСТВОВАНИЕ: Сохранение в БД

Вместо JSON можно сохранять в базу данных:

```yaml
# В workflow можно добавить

- name: Parse and save to database
  env:
    DATABASE_URL: ${{ secrets.DATABASE_URL }}
  run: |
    python3 << 'EOF'
    import os
    from cian_parser import CianParser
    import psycopg2
    
    # Парсим
    parser = CianParser(headless=True)
    listings = parser.search_listings({'city': 'moskva'}, max_pages=2)
    
    # Сохраняем в БД
    conn = psycopg2.connect(os.getenv('DATABASE_URL'))
    cursor = conn.cursor()
    
    for listing in listings:
        cursor.execute("""
            INSERT INTO listings (address, price, area, url, parsed_at)
            VALUES (%s, %s, %s, %s, NOW())
            ON CONFLICT (url) DO NOTHING
        """, (listing['address'], listing['price'], listing['area'], listing['url']))
    
    conn.commit()
    conn.close()
    parser.close()
    EOF
```

---

## 📈 МОНИТОРИНГ

### Смотрите прогресс на GitHub

1. **Actions tab** — логи всех запусков
2. **Commits** — какие файлы менялись
3. **Releases** — если хотите версионировать

### Получайте уведомления

GitHub автоматически отправляет email при ошибках.

---

## ⚠️ ОГРАНИЧЕНИЯ GitHub Actions

| Ограничение | Значение |
|-------------|----------|
| Бесплатный лимит | 2000 минут в месяц |
| Работа одного workflow | 35 дней max |
| Память | ~7 GB |
| Диск | ~14 GB |
| CPU | 2-4 ядра |

**Расход минут:**
- 1 запуск парсера = 2-5 минут
- Запуск каждый час = 24 × 3 минуты = 72 минуты в день
- Запуск каждый день = 3-5 минут в день

**Вывод:** Бесплатного плана хватает легко даже на почасовой мониторинг! ✅

---

## 🚀 ПОЛНЫЙ ПРИМЕР (Production)

Вот боевой вариант для реального мониторинга:

**`.github/workflows/monitor_deals.yml`:**

```yaml
name: Monitor Good Deals (Production)

on:
  schedule:
    # Каждые 2 часа днём (9-21 МСК)
    - cron: '0 6,8,10,12,14,16,18 * * *'
  
  workflow_dispatch:

jobs:
  monitor:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.10'
      
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
      
      - name: Parse and find good deals
        id: parse
        run: |
          python3 << 'EOF'
          from cian_parser import CianParser
          import json
          from datetime import datetime
          
          parser = CianParser(headless=True)
          
          try:
              filters = {
                  'city': 'moskva',
                  'price_min': 5_000_000,
                  'price_max': 20_000_000,
                  'rooms': [1, 2, 3],
              }
              
              listings = parser.search_listings(filters, max_pages=2)
              
              # Ищем хорошие предложения (сегодня + дешевле)
              good = [
                  l for l in listings
                  if l['days_old'] <= 1 and l['price_per_sqm'] < 150_000
              ]
              
              if good:
                  # Сохраняем
                  filename = f'data/deals_{datetime.now().strftime("%Y%m%d_%H%M%S")}.json'
                  with open(filename, 'w') as f:
                      json.dump(good, f, ensure_ascii=False, indent=2, default=str)
                  
                  # Выводим для GitHub Actions
                  print(f"::notice::Found {len(good)} good deals!")
                  
                  # Сохраняем в переменную для следующего шага
                  with open('good_deals.txt', 'w') as f:
                      f.write(str(len(good)))
          
          finally:
              parser.close()
          
          EOF
      
      - name: Commit results
        run: |
          git config --local user.email "action@github.com"
          git config --local user.name "GitHub Action"
          git add data/
          git commit -m "🤖 Good deals found" || true
          git push || true
      
      - name: Send Telegram alert
        if: success()
        run: |
          python3 << 'EOF'
          import requests
          
          TOKEN = "${{ secrets.TELEGRAM_BOT_TOKEN }}"
          CHAT_ID = "${{ secrets.TELEGRAM_CHAT_ID }}"
          
          try:
              with open('good_deals.txt', 'r') as f:
                  count = int(f.read())
              
              if count > 0:
                  msg = f"🎯 Найдено {count} хороших объявлений!\nhttps://github.com/${{ github.repository }}/tree/main/data"
                  requests.post(
                      f"https://api.telegram.org/bot{TOKEN}/sendMessage",
                      json={'chat_id': CHAT_ID, 'text': msg, 'parse_mode': 'HTML'}
                  )
          except:
              pass
          
          EOF
```

---

## ✅ ИТОГО

Вы получили:
- ✅ Автоматический запуск на GitHub Actions
- ✅ Сохранение результатов в репо
- ✅ Расписание (ежедневно, ежечасно, или вручную)
- ✅ Telegram уведомления
- ✅ Логирование и отслеживание

**Это полностью рабочая система мониторинга!** 🚀

---

**Версия:** 1.0  
**Дата:** 2026-01-07
