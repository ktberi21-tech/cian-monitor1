# 📦 БЫСТРЫЙ СТАРТ НА GITHUB ACTIONS

## ⚡ За 10 минут

### Шаг 1: Создайте репо на GitHub (2 минуты)

1. Откройте https://github.com/new
2. Назовите **cian-monitor**
3. Выберите **Public** (чтобы видны логи)
4. Нажмите **Create repository**

### Шаг 2: Загрузьте файлы (3 минуты)

```bash
# В командной строке на компьютере:

git clone https://github.com/YOUR_USERNAME/cian-monitor.git
cd cian-monitor

# Скопируйте сюда все файлы парсера:
# - cian_parser.py
# - config.py
# - examples.py
# - requirements.txt
# - README.md
# - и т.д.

# Создайте папку для workflows
mkdir -p .github/workflows

# Затем загружайте на GitHub
git add .
git commit -m "Initial: Add Cian parser"
git push
```

### Шаг 3: Добавьте workflow файл (3 минуты)

1. На GitHub откройте файловый браузер
2. Нажмите **Add file** → **Create new file**
3. Введите путь: `.github/workflows/parse_daily.yml`
4. Скопируйте содержимое из `parse_daily.yml`
5. Нажмите **Commit new file**

### Шаг 4: Добавьте Telegram токен (2 минуты)

**Опционально, но рекомендуется!**

1. Откройте Settings репо (gear icon)
2. **Secrets and variables** → **Actions**
3. **New repository secret**
4. Добавьте:
   - `TELEGRAM_BOT_TOKEN` = ваш токен (@BotFather)
   - `TELEGRAM_CHAT_ID` = ваш ID (@userinfobot)
5. **Add secret**

### Шаг 5: Запустите тест (автоматически)

Готово! Парсер начнёт запускаться по расписанию.

---

## 🎮 УПРАВЛЕНИЕ WORKFLOW

### Посмотреть логи

1. Откройте репо на GitHub
2. Вкладка **Actions**
3. Выберите workflow
4. Нажмите на run
5. Смотрите логи в реальном времени

### Запустить вручную

1. **Actions** → выберите workflow
2. **Run workflow**
3. **Run workflow** (ещё раз)
4. Ждите результатов

### Изменить расписание

Отредактируйте `.github/workflows/parse_daily.yml`:

```yaml
on:
  schedule:
    - cron: '0 6 * * *'  # 6:00 UTC (9:00 МСК)
```

Расписания:
- `0 * * * *` — каждый час
- `0 6 * * *` — каждый день в 6:00 UTC
- `0 6 * * 1-5` — по рабочим дням в 6:00 UTC
- `*/30 * * * *` — каждые 30 минут

---

## 📂 ФАЙЛОВАЯ СТРУКТУРА

```
cian-monitor/
├── .github/
│   └── workflows/
│       └── parse_daily.yml       ← создаёте на GitHub
├── cian_parser.py
├── config.py
├── examples.py
├── requirements.txt
├── README.md
├── data/                         ← создастся автоматически
│   └── listings_2026-01-07.json
└── ...
```

---

## 🔍 РЕШЕНИЕ ПРОБЛЕМ

### ❌ Workflow не запускается

**Проверьте:**
1. Файл в правильной папке: `.github/workflows/parse_daily.yml`
2. YAML синтаксис правильный (скопируйте ещё раз)
3. Нажмите **Actions** → посмотрите статус

### ❌ Ошибка при парсинге

**Смотрите логи:**
1. **Actions** → workflow
2. Раскройте **Run parser**
3. Ищите `❌` красные ошибки
4. Скорее всего проблема в селекторах (Циан обновил сайт)

**Решение:**
```python
# Обновите селекторы в cian_parser.py
# Или добавьте прокси в config.py
CIAN_CONFIG['delay_between_pages'] = 5
```

### ❌ Telegram не отправляет

**Проверьте:**
1. Токен добавлен правильно в Secrets
2. Бот может писать в ваш чат (@BotFather → /mybots)
3. CHAT_ID правильный (@userinfobot)

---

## 💡 РАСШИРЕННЫЕ ПРИМЕРЫ

### Пример 1: Разные города в разное время

```yaml
name: Parse Multiple Cities

on:
  schedule:
    - cron: '0 6 * * *'    # Москва в 9 утра МСК
    - cron: '0 7 * * *'    # СПб в 10 утра МСК
    - cron: '0 8 * * *'    # Екб в 11 утра МСК

jobs:
  parse_moskva:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-python@v4
        with:
          python-version: '3.10'
      - run: |
          pip install -r requirements.txt
          python3 << 'EOF'
          from cian_parser import CianParser
          parser = CianParser(headless=True)
          try:
              listings = parser.search_listings({'city': 'moskva'}, max_pages=2)
              parser.save_to_json(listings, 'data/moskva.json')
          finally:
              parser.close()
          EOF
      - run: git config --local user.email "action@github.com" && git config --local user.name "GitHub Action" && git add data/ && git commit -m "🤖 Moscow" || true && git push || true
```

### Пример 2: Поиск только выгодных предложений

```yaml
name: Find Good Deals

on:
  schedule:
    - cron: '0 */2 * * *'  # Каждые 2 часа

jobs:
  deals:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-python@v4
        with:
          python-version: '3.10'
      - run: |
          pip install -r requirements.txt
          python3 << 'EOF'
          from cian_parser import CianParser
          import json
          from datetime import datetime
          
          parser = CianParser(headless=True)
          try:
              listings = parser.search_listings({'city': 'moskva'}, max_pages=1)
              
              # Фильтруем выгодные (сегодня + дешевле 150k/м²)
              good = [l for l in listings if l['days_old'] == 0 and l['price_per_sqm'] < 150_000]
              
              if good:
                  filename = f'data/deals_{datetime.now().strftime("%H%M")}.json'
                  with open(filename, 'w') as f:
                      json.dump(good, f, ensure_ascii=False, indent=2, default=str)
          finally:
              parser.close()
          EOF
      - run: git config --local user.email "action@github.com" && git config --local user.name "GitHub Action" && git add data/ && git commit -m "💰 Good deals found" || true && git push || true
```

---

## 📊 МОНИТОРИНГ ИСПОЛЬЗОВАНИЯ

**GitHub Actions бесплатные для публичных репозиториев!**

**Проверить использование:**
1. Settings → Usage
2. Смотрите действующие minutes

**Лимиты:**
- 2000 минут/месяц (для приватных)
- Для публичных — **без лимита!** ✅

---

## 🚀 ВЫ ГОТОВЫ!

Теперь у вас есть:
- ✅ Автоматический парсер на GitHub Actions
- ✅ Сохранение результатов в репо
- ✅ Расписание (каждый день, каждый час)
- ✅ Telegram уведомления
- ✅ История всех запусков

**Парсер работает 24/7 даже когда вы спите!** 🎉

---

**Версия:** 1.0  
**Дата:** 2026-01-07
