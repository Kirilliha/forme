# ШАГ 7: Сборка и Дистрибуция (.exe, .dmg, иконки)

Финальный шаг! Здесь мы собираем приложение в нативные файлы для Windows и macOS, меняем иконку и готовим к передаче другу.

---

## 7.1 Предварительная проверка

Перед сборкой убедись, что всё работает:

```bash
# Перейди в папку проекта
cd ~/aurora-music-player

# 1. Проверь что frontend собирается
npm run build

# 2. Проверь что Rust компилируется
cd src-tauri
cargo check
cd ..

# 3. Запусти в dev-режиме и проверь функционал
cargo tauri dev
```

**Чек-лист перед сборкой:**
- [ ] Приложение запускается без ошибок
- [ ] Импорт папки работает (треки появляются в списке)
- [ ] Play/Pause работает
- [ ] Переключение треков (Next/Prev) работает
- [ ] Прогресс-бар двигается
- [ ] Громкость регулируется
- [ ] Редактирование метаданных сохраняется
- [ ] LRC текст синхронизируется и скроллится

---

## 7.2 Смена иконки приложения

### Подготовь изображение

1. Создай или найди квадратное изображение **1024×1024 пикселей** (PNG)
2. Желательно с прозрачным фоном (но не обязательно)
3. Положи его в папку проекта: `~/aurora-music-player/icon-source.png`

### Установка tauri-icon

```bash
# Установи утилиту для генерации иконок (один раз)
cargo install tauri-icon
```

### Генерация всех размеров иконок

```bash
# Находясь в ~/aurora-music-player:
tauri-icon icon-source.png --output src-tauri/icons
```

Эта команда автоматически создаст все необходимые файлы:

| Файл | Размер | Назначение |
|------|--------|------------|
| `32x32.png` | 32×32 | Мелкие иконки в UI |
| `128x128.png` | 128×128 | Средние иконки |
| `128x128@2x.png` | 256×256 | Retina дисплеи |
| `icon.icns` | мульти | macOS (содержит все размеры) |
| `icon.ico` | мульти | Windows (содержит 16, 32, 48, 256) |

> **Важно:** Не меняй имена файлов! Tauri ищет именно эти имена в `src-tauri/icons/`.

### Проверь `src-tauri/tauri.conf.json`

Убедись что секция `bundle.icon` указывает на правильные файлы (должно быть по умолчанию):

```json
{
  "bundle": {
    "icon": [
      "icons/32x32.png",
      "icons/128x128.png",
      "icons/128x128@2x.png",
      "icons/icon.icns",
      "icons/icon.ico"
    ]
  }
}
```

---

## 7.3 Сборка приложения

### Команда сборки (универсальная)

```bash
# В корне проекта (~/aurora-music-player)
cargo tauri build
```

Что происходит при сборке:
1. **Frontend build** — Vite собирает React в `dist/` (HTML, CSS, JS)
2. **Rust build** — Cargo компилирует Rust код
3. **Bundler** — Tauri объединяет frontend + backend в одно приложение
4. **Упаковка** — Создаётся установщик для текущей ОС

---

## 7.4 Результаты сборки

### Windows

После сборки на Windows:

```
src-tauri/target/release/
├── Aurora Music Player.exe          # ← Само приложение (можно запустить напрямую)
└── bundle/
    └── nsis/
        └── Aurora Music Player-setup.exe   # ← Установщик (рекомендуется для друга)
```

**Для передачи другу:**
- **Вариант A (портативный):** Передай папку `release/` — друг запустит `.exe`
- **Вариант B (установщик):** Передай `*-setup.exe` — друг установит как обычную программу

### macOS

После сборки на macOS:

```
src-tauri/target/release/
├── bundle/
│   ├── dmg/
│   │   └── Aurora Music Player_1.0.0_x64.dmg   # ← Установщик (рекомендуется)
│   └── macos/
│       └── Aurora Music Player.app               # ← .app bundle
```

**Для передачи другу:**
- **Вариант A (.dmg):** Передай `.dmg` — друг откроет и перетащит app в Applications
- **Вариант B (.app):** Передай `.app` папку (но может быть проблема с "неизвестным разработчиком")

---

## 7.5 Кроссплатформенная сборка

### Собрать под Windows на Windows

```bash
# Просто запусти (результат: .exe + .msi)
cargo tauri build

# Только .exe (без MSI установщика):
cargo tauri build --bundles nsis
```

### Собрать под macOS на macOS

```bash
# Универсальный бинарник (Intel + Apple Silicon):
cargo tauri build --target universal-apple-darwin

# Только Intel (x64):
cargo tauri build --target x86_64-apple-darwin

# Только Apple Silicon (M1/M2):
cargo tauri build --target aarch64-apple-darwin
```

### Сборка под другую ОС (кросскомпиляция)

Tauri **не поддерживает кросскомпиляцию** из коробки. Для сборки под:
- **Windows** → нужен Windows (или GitHub Actions / CI)
- **macOS** → нужен Mac (из-за code signing)

### GitHub Actions (Автоматическая сборка)

Создай файл `.github/workflows/release.yml` для автоматической сборки на всех платформах:

```yaml
name: Release
on:
  push:
    tags: ['v*']
jobs:
  build:
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - uses: dtolnay/rust-action@stable
      - run: npm install
      - run: cargo tauri build
      - uses: actions/upload-artifact@v4
        with:
          name: aurora-${{ matrix.os }}
          path: src-tauri/target/release/bundle/**
```

---

## 7.6 Оптимизация размера

### Уменьши размер `.exe`:

**`src-tauri/Cargo.toml`** — добавь в `[profile.release]`:

```toml
[profile.release]
# Оптимизация размера (вместо скорости)
opt-level = "s"           # Оптимизация под размер
lto = true                # Link Time Optimization (медленнее сборка, меньше бинарник)
codegen-units = 1         # Единый codegen unit (лучше оптимизация)
panic = "abort"           # Меньше кода обработки паники
strip = true              # Удалить debug символы
```

### Уменьши размер frontend:

**`vite.config.ts`** — уже должно быть по умолчанию:

```typescript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
    plugins: [react()],
    build: {
        minify: 'terser',        // Минификация
        sourcemap: false,        // Не генерировать source maps
        rollupOptions: {
            output: {
                manualChunks: {  // Разделение на чанки
                    vendor: ['react', 'react-dom'],
                    howler: ['howler'],
                    framer: ['framer-motion'],
                },
            },
        },
    },
});
```

---

## 7.7 Подпись кода (Code Signing) — Для продакшена

Если передаёшь другу — подпись не обязательна. Для распространения:

### Windows (самоподписанный сертификат):
```powershell
# Создай сертификат
New-SelfSignedCertificate -DnsName "Aurora Music" -CertStoreLocation cert:\CurrentUser\My

# Подпиши .exe
signtool sign /fd SHA256 /a "Aurora Music Player.exe"
```

### macOS (требуется Apple Developer ID):
```bash
# Экспорт сертификата
security find-identity -v -p codesigning

# Подпись
codesign --force --sign "Developer ID" Aurora Music Player.app

# Нотаризация (для запуска без предупреждений)
xcrun notarytool submit Aurora Music Player.dmg --wait
```

---

## 7.8 Итоговая структура проекта

После прохождения всех 7 шагов:

```
~/aurora-music-player/
├── src/                              # Frontend (React + TS)
│   ├── components/
│   │   ├── player/PlayerBar.tsx      # Панель плеера
│   │   ├── tracklist/                # Список треков
│   │   ├── lyrics/LyricsViewer.tsx   # Синхронизированный текст
│   │   ├── edit/EditTrackModal.tsx   # Редактирование
│   │   └── layout/                   # Layout компоненты
│   ├── hooks/
│   │   ├── useAudioPlayer.ts         # Howler.js хук
│   │   ├── useLyrics.ts              # Синхронизация lyrics
│   │   └── useDragDrop.ts            # Drag & Drop
│   ├── lib/
│   │   ├── db.ts                     # SQLite CRUD
│   │   ├── lrcParser.ts              # Парсер LRC
│   │   ├── importTracks.ts           # Импорт треков
│   │   └── utils.ts                  # Утилиты
│   ├── types/                        # TypeScript типы
│   ├── providers/PlayerProvider.tsx  # Глобальное состояние
│   ├── App.tsx                       # Корневой компонент
│   ├── main.tsx                      # Точка входа
│   └── index.css                     # Стили + тема Aurora
│
├── src-tauri/                        # Backend (Rust)
│   ├── src/
│   │   ├── main.rs                   # Точка входа + миграции
│   │   └── commands/
│   │       ├── scan.rs               # Сканирование директорий
│   │       └── metadata.rs           # Чтение аудио метаданных
│   ├── migrations/
│   │   └── 001_initial.sql           # Схема БД
│   ├── capabilities/
│   │   └── default.json              # Разрешения
│   ├── icons/                        # Иконки приложения
│   ├── Cargo.toml                    # Rust зависимости
│   └── tauri.conf.json               # Конфигурация
│
├── package.json                      # npm зависимости
├── vite.config.ts                    # Vite конфиг
├── tailwind.config.js                # Тема Aurora
└── tsconfig.json                     # TypeScript
```

---

## 7.9 Чек-лист запуска

### Первый запуск на новой машине:

```bash
# 1. Клонируй/скопируй проект
cd aurora-music-player

# 2. Установи frontend зависимости
npm install

# 3. Запусти dev-сервер
cargo tauri dev

# 4. Для сборки:
cargo tauri build
```

### Распространение (другу):

1. **Собери** приложение: `cargo tauri build`
2. **Найди файл:**
   - Windows: `src-tauri/target/release/bundle/nsis/*.exe`
   - macOS: `src-tauri/target/release/bundle/dmg/*.dmg`
3. **Передай файл** другу (Telegram, облако, флешка)
4. **Друг устанавливает** как обычную программу

---

## ✅ ПОЛНЫЙ ИТОГ ВСЕХ 7 ШАГОВ

| Шаг | Что сделали |
|-----|-------------|
| **1** | Установили Rust, Node.js, Tauri CLI, все npm-пакеты |
| **2** | Создали структуру папок (src/, src-tauri/) |
| **3** | Написали Rust-бэкенд: сканирование папок, чтение метаданных через Symphonia |
| **4** | Настроили SQLite: таблица tracks, CRUD, импорт, поиск |
| **5** | Создали UI: glassmorphism-панель, список треков, редактор, модалки |
| **6** | Реализовали аудио: Howler.js, LRC-парсер, синхронизация, автоскролл |
| **7** | Собрали .exe/.dmg, настроили иконку, подготовили к дистрибуции |

---

**Твой Aurora Music Player готов!** Если возникнут вопросы по любому шагу — задавай, разберёмся вместе.
