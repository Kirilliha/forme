# ШАГ 2: Структура проекта — Дерево папок и файлов

После инициализации Tauri у тебя базовая структура. Нам нужно создать дополнительные папки и файлы под наши компоненты. Вот полная структура проекта Aurora Music Player:

---

## Полное дерево файлов

```
~/aurora-music-player/                    # Корень проекта
│
├── src/                                  # === FRONTEND (React + TypeScript) ===
│   │
│   ├── main.tsx                          # Точка входа React (рендерит App)
│   ├── App.tsx                           # Корневой компонент, маршрутизация
│   ├── index.css                         # Глобальные стили Tailwind + тема Aurora
│   ├── vite-env.d.ts                     # Типы для Vite
│   │
│   ├── types/                            # TypeScript интерфейсы
│   │   ├── track.ts                      # Интерфейс Track (трек)
│   │   ├── lyrics.ts                     # Интерфейс LyricsLine (строка текста)
│   │   └── index.ts                      # Реэкспорт всех типов
│   │
│   ├── hooks/                            # Кастомные React-хуки
│   │   ├── useAudioPlayer.ts             # Главный хук: play, pause, progress, Howler
│   │   ├── useLyrics.ts                  # Хук для синхронизации lyrics с аудио
│   │   ├── useTracks.ts                  # Хук для работы с БД треков (CRUD)
│   │   └── useDragDrop.ts                # Хук для Drag & Drop файлов/папок
│   │
│   ├── components/                       # Переиспользуемые компоненты
│   │   ├── ui/                           # shadcn/ui компоненты (автогенерация)
│   │   │   ├── button.tsx
│   │   │   ├── slider.tsx
│   │   │   ├── dialog.tsx
│   │   │   ├── scroll-area.tsx
│   │   │   ├── input.tsx
│   │   │   └── label.tsx
│   │   │
│   │   ├── player/                       # Компоненты плеера
│   │   │   ├── PlayerBar.tsx             # Нижняя панель плеера (обложка, кнопки, прогресс)
│   │   │   ├── TrackInfo.tsx             # Информация о треке (название, исполнитель)
│   │   │   ├── Controls.tsx              # Кнопки управления (play, pause, next, prev)
│   │   │   ├── ProgressBar.tsx           # Полоса прогресса + текущее время
│   │   │   └── VolumeControl.tsx         # Регулятор громкости
│   │   │
│   │   ├── tracklist/                    # Список треков
│   │   │   ├── TrackList.tsx             # Основной список всех треков
│   │   │   ├── TrackItem.tsx             # Одна строка трека в списке
│   │   │   └── TrackListHeader.tsx       # Заголовок списка (колонки)
│   │   │
│   │   ├── lyrics/                       # Компоненты lyrics
│   │   │   ├── LyricsViewer.tsx          # Основной компонент отображения текста
│   │   │   ├── LyricsLine.tsx            # Одна строка lyrics (с анимацией)
│   │   │   └── LyricsParser.ts           # Утилита парсинга LRC-формата
│   │   │
│   │   ├── edit/                         # Редактирование метаданных
│   │   │   ├── EditTrackModal.tsx        # Модальное окно редактирования трека
│   │   │   ├── CoverUploader.tsx         # Загрузка/смена обложки
│   │   │   └── MetadataForm.tsx          # Форма ввода названия/исполнителя
│   │   │
│   │   ├── layout/                       # Компоненты макета
│   │   │   ├── Sidebar.tsx               # Боковая панель (навигация)
│   │   │   ├── MainContent.tsx           # Основная область контента
│   │   │   └── GlassPanel.tsx            # Обертка с glassmorphism эффектом
│   │   │
│   │   └── common/                       # Общие компоненты
│   │       ├── CoverArt.tsx              # Отображение обложки (с fallback)
│   │       ├── IconButton.tsx            # Кнопка с иконкой (lucide)
│   │       └── TimeDisplay.tsx           # Отображение времени (mm:ss)
│   │
│   ├── lib/                              # Утилиты и хелперы
│   │   ├── utils.ts                      # cn() и другие утилиты (от shadcn)
│   │   ├── db.ts                         # Обёртка для работы с SQLite через Tauri plugin
│   │   ├── audio.ts                      # Утилиты для аудио (формат времени и т.д.)
│   │   └── lrcParser.ts                  # Парсер LRC-файлов
│   │
│   └── providers/                        # React Context / Providers
│       └── PlayerProvider.tsx            # Контекст плеера (глобальное состояние)
│
│
├── src-tauri/                            # === BACKEND (Rust) ===
│   │
│   ├── src/
│   │   ├── main.rs                       # Точка входа Rust, регистрация команд
│   │   ├── lib.rs                        # Модульные экспорты (если нужно)
│   │   └── commands/                     # Tauri Commands (функции бэкенда)
│   │       ├── scan.rs                   # Сканирование директории на аудиофайлы
│   │       ├── metadata.rs               # Чтение/запись метаданных аудио
│   │       └── fs_extra.rs               # Дополнительные файловые операции
│   │
│   ├── Cargo.toml                        # Зависимости Rust (tauri, symphonia и т.д.)
│   ├── build.rs                          # Скрипт сборки Tauri
│   ├── tauri.conf.json                   # Конфигурация Tauri (окно, иконки, права)
│   │
│   ├── capabilities/
│   │   └── default.json                  # Разрешения безопасности (fs, dialog, sql)
│   │
│   └── icons/                            # Иконки приложения
│       ├── 32x32.png
│       ├── 128x128.png
│       ├── 128x128@2x.png
│       ├── icon.icns                     # macOS иконка
│       └── icon.ico                      # Windows иконка
│
│
├── public/                               # Статические файлы
│   └── default-cover.png                 # Стандартная обложка (fallback)
│
│
├── index.html                            # HTML-шаблон (Vite)
├── package.json                          # npm зависимости
├── vite.config.ts                        # Конфигурация Vite
├── tailwind.config.js                    # Конфигурация Tailwind
├── tsconfig.json                         # TypeScript (основной)
├── tsconfig.app.json                     # TypeScript (приложение)
├── tsconfig.node.json                    # TypeScript (Vite config)
├── postcss.config.js                     # PostCSS (Tailwind)
├── components.json                       # Конфиг shadcn/ui
└── .gitignore                            # Игнорируемые файлы Git
```

---

## Создание всех папок (одной командой)

Выполни эту команду в терминале из корня проекта (`~/aurora-music-player`):

```bash
# Создаём все папки для frontend
mkdir -p src/types src/hooks src/lib src/providers
mkdir -p src/components/player src/components/tracklist
mkdir -p src/components/lyrics src/components/edit
mkdir -p src/components/layout src/components/common

# Создаём папки для backend (commands уже созданы — инициализацией Tauri)
mkdir -p src-tauri/src/commands
mkdir -p src-tauri/capabilities
mkdir -p public
```

---

## Что хранится в каждой папке

| Папка | Назначение |
|-------|-----------|
| `src/types/` | Все TypeScript интерфейсы — единый источник типов для треков, lyrics, состояний |
| `src/hooks/` | Кастомные React-хуки — вся бизнес-логика (аудио, lyrics, Drag&Drop) |
| `src/components/` | UI-компоненты, разбитые по функциональным областям |
| `src/lib/` | Утилиты — чистые функции без React-зависимостей (парсеры, форматеры) |
| `src/providers/` | React Context для глобального состояния (текущий трек, плейлист) |
| `src-tauri/src/commands/` | Rust-функции, вызываемые из фронтенда через `invoke` |
| `public/` | Статика — fallback обложка и другие ресурсы |

---

## Принципы организации кода

1. **Разделение ответственности:**
   - `hooks/` — состояние и побочные эффекты
   - `components/` — отрисовка UI
   - `lib/` — чистые функции (без React)
   - `types/` — только TypeScript интерфейсы

2. **Каждый компонент — в своей папке:**
   - `PlayerBar/` содержит `PlayerBar.tsx` + вспомогательные компоненты
   - Все дочерние компоненты импортируются только родителем

3. **Единый источник типов:**
   - Все интерфейсы в `types/`, никаких `interface` inline в компонентах
   - Реэкспорт через `types/index.ts`

---

## ✅ Что делаем на этом шаге

1. Создать все папки (команда `mkdir` выше) ✅
2. Понять, куда класть каждый файл ✅
3. Перейти к написанию кода (Шаг 3) ✅

---

**Напиши "далее" — перейдём к ШАГУ 3: Rust-бэкенд с Tauri Commands для сканирования аудиофайлов.**
