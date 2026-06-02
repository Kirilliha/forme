# Aurora Music Player — Пошаговый Туториал (Tauri v2 + React + TS)

## ШАГ 1: Подготовка окружения

В этом шаге мы установим все необходимые инструменты: Rust, Node.js, Tauri CLI и создадим скелет проекта.

---

### 1.1 Установка Rust (если не установлен)

**Windows (PowerShell):**
```powershell
# Скачай и запусти установщик с официального сайта:
# https://www.rust-lang.org/tools/install
# Или через winget:
winget install Rustlang.Rustup

# После установки — перезапусти терминал и проверь:
rustc --version    # Должно показать версию, например: rustc 1.78.0
cargo --version    # Должно показать версию cargo
```

**macOS / Linux:**
```bash
# Установка через rustup (официальный менеджер версий Rust)
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# Выбери опцию 1 (Default installation) при запросе

# После установки — перезагрузи переменные окружения:
source $HOME/.cargo/env

# Проверь установку:
rustc --version    # rustc 1.78.0 (или новее)
cargo --version    # cargo 1.78.0 (или новее)
```

> **Важно:** Tauri v2 требует Rust **1.75.0** или новее. Рекомендуется стабильная версия (stable channel).

---

### 1.2 Установка Node.js (если не установлен)

**Windows:**
```powershell
# Скачай LTS-версию с https://nodejs.org/ (рекомендуется 20.x LTS)
# Или через winget:
winget install OpenJS.NodeJS.LTS

# Проверь:
node --version     # v20.x.x или выше
npm --version      # 10.x.x или выше
```

**macOS:**
```bash
# Через Homebrew (рекомендуется):
brew install node@20

# Или скачай установщик с https://nodejs.org/

# Проверь:
node --version     # v20.x.x или выше
npm --version      # 10.x.x или выше
```

---

### 1.3 Установка Tauri CLI

Tauri CLI — это инструмент командной строки для создания, разработки и сборки Tauri-приложений.

```bash
# Устанавливаем глобально через cargo (рекомендуется для v2)
cargo install tauri-cli@^2.0.0

# Проверь установку:
cargo tauri --version    # Должно показать версию 2.x.x
```

---

### 1.4 Установка системных зависимостей (ОБЯЗАТЕЛЬНО!)

**Windows:**
```powershell
# Установи Microsoft C++ Build Tools:
# 1. Скачай: https://visualstudio.microsoft.com/visual-cpp-build-tools/
# 2. При установке выбери:
#    - "Desktop development with C++" 
#    - Или минимум: MSVC v143 + Windows 11 SDK

# Также нужен WebView2 Runtime (обычно уже есть в Windows 11/10):
# https://developer.microsoft.com/en-us/microsoft-edge/webview2/
```

**macOS:**
```bash
# Установи Xcode Command Line Tools:
xcode-select --install

# Если уже установлены — обнови:
xcode-select --reset
```

**Linux (Ubuntu/Debian):**
```bash
sudo apt update
sudo apt install libwebkit2gtk-4.1-dev build-essential curl wget file libssl-dev libayatana-appindicator3-dev librsvg2-dev
```

---

### 1.5 Создание проекта Tauri + React + TypeScript

```bash
# Создай папку для проекта и перейди в неё
mkdir ~/aurora-music-player
cd ~/aurora-music-player

# Инициализируй frontend часть через Vite (выбери React + TypeScript)
npm create vite@latest . -- --template react-ts

# Ответь на вопросы:
# ? Package name: aurora-music-player
# ? Select a framework: React
# ? Select a variant: TypeScript
```

После выполнения этой команды у тебя будет базовая структура Vite-проекта:
```
~/aurora-music-player/
├── src/
│   ├── App.tsx
│   ├── main.tsx
│   └── vite-env.d.ts
├── index.html
├── package.json
├── tsconfig.json
├── tsconfig.app.json
├── tsconfig.node.json
└── vite.config.ts
```

---

### 1.6 Инициализация Tauri в проекте

```bash
# Находясь в ~/aurora-music-player, инициализируй Tauri:
cargo tauri init

# Ответь на вопросы:
# ? What is your app name? Aurora Music Player
# ? What should the window title be? Aurora Music Player
# ? Where are your web assets located relative to the <current dir>/src-tauri? ../dist
# ? What is the URL of your dev server? http://localhost:5173
# ? What is your frontend dev command? npm run dev
# ? What is your frontend build command? npm run build
```

После этой команды появится папка `src-tauri/` с Rust-кодом:
```
~/aurora-music-player/
├── src/                    # React frontend
├── src-tauri/              # Rust backend
│   ├── src/
│   │   └── main.rs         # Точка входа Rust
│   ├── Cargo.toml          # Зависимости Rust
│   ├── tauri.conf.json     # Конфигурация Tauri
│   └── icons/              # Иконки приложения
├── index.html
├── package.json
└── ...
```

---

### 1.7 Установка npm-зависимостей (Frontend)

```bash
# Установи базовые зависимости (уже должны быть от Vite)
npm install

# Установи Tailwind CSS + PostCSS + Autoprefixer
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p

# Установи shadcn/ui CLI и инициализируй
npm install -D @shadcn/ui
npx shadcn-ui@latest init

# При инициализации shadcn/ui ответь:
# ? Would you like to use TypeScript? yes
# ? Which style would you like to use? Default
# ? Which base color would you like to use? Slate (или Neutral)
# ? Where is your global CSS file? src/index.css
# ? Do you want to use CSS variables? yes
# ? Where is your tailwind.config.js? tailwind.config.js
# ? Configure the import alias? yes
# ? What is the import alias? @/

# Установи необходимые shadcn/ui компоненты
npx shadcn-ui@latest add button
npx shadcn-ui@latest add slider
npx shadcn-ui@latest add dialog
npx shadcn-ui@latest add scroll-area
npx shadcn-ui@latest add input
npx shadcn-ui@latest add label

# Установи остальные npm-пакеты
npm install howler framer-motion lucide-react
npm install -D @types/howler

# Установи Tauri API и плагины
npm install @tauri-apps/api @tauri-apps/plugin-dialog @tauri-apps/plugin-fs @tauri-apps/plugin-sql
```

**Итоговый список зависимостей в `package.json`:**
```json
{
  "name": "aurora-music-player",
  "private": true,
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "preview": "vite preview",
    "tauri": "tauri",
    "tauri:dev": "tauri dev",
    "tauri:build": "tauri build"
  },
  "dependencies": {
    "@tauri-apps/api": "^2.0.0",
    "@tauri-apps/plugin-dialog": "^2.0.0",
    "@tauri-apps/plugin-fs": "^2.0.0",
    "@tauri-apps/plugin-sql": "^2.0.0",
    "howler": "^2.2.4",
    "framer-motion": "^11.0.0",
    "lucide-react": "^0.400.0",
    "react": "^18.3.0",
    "react-dom": "^18.3.0",
    "class-variance-authority": "^0.7.0",
    "clsx": "^2.1.0",
    "tailwind-merge": "^2.3.0"
  },
  "devDependencies": {
    "@types/howler": "^2.2.11",
    "@types/react": "^18.3.0",
    "@types/react-dom": "^18.3.0",
    "@vitejs/plugin-react": "^4.3.0",
    "autoprefixer": "^10.4.19",
    "postcss": "^8.4.38",
    "tailwindcss": "^3.4.0",
    "@shadcn/ui": "latest",
    "typescript": "^5.4.0",
    "vite": "^5.3.0"
  }
}
```

---

### 1.8 Настройка Tailwind CSS

**`tailwind.config.js`** — добавь пути к файлам и кастомные цвета:
```javascript
/** @type {import('tailwindcss').Config} */
export default {
  darkMode: ["class"],
  content: [
    "./index.html",
    "./src/**/*.{ts,tsx,js,jsx}",
  ],
  theme: {
    extend: {
      colors: {
        // Aurora тема — тёмные оттенки с фиолетово-синими акцентами
        aurora: {
          bg: "#0a0a0f",           // Основной фон (почти чёрный)
          surface: "#12121a",       // Поверхность (чуть светлее)
          glass: "rgba(18, 18, 26, 0.75)",  // Стеклянный эффект
          accent: "#7c3aed",        // Фиолетовый акцент
          "accent-light": "#a78bfa", // Светло-фиолетовый
          text: "#e2e8f0",          // Основной текст
          "text-muted": "#94a3b8",  // Приглушённый текст
        },
      },
      backdropBlur: {
        xs: "2px",
      },
      animation: {
        "pulse-slow": "pulse 3s cubic-bezier(0.4, 0, 0.6, 1) infinite",
        "float": "float 6s ease-in-out infinite",
      },
      keyframes: {
        float: {
          "0%, 100%": { transform: "translateY(0)" },
          "50%": { transform: "translateY(-10px)" },
        },
      },
    },
  },
  plugins: [],
}
```

**`src/index.css`** — глобальные стили с поддержкой тёмной темы:
```css
@tailwind base;
@tailwind components;
@tailwind utilities;

/* ===== Aurora Music Player — Global Styles ===== */

@layer base {
  :root {
    --background: 240 10% 4%;
    --foreground: 210 40% 93%;
    --card: 240 15% 8%;
    --card-foreground: 210 40% 93%;
    --popover: 240 15% 8%;
    --popover-foreground: 210 40% 93%;
    --primary: 262 83% 58%;
    --primary-foreground: 0 0% 100%;
    --secondary: 240 12% 15%;
    --secondary-foreground: 210 40% 93%;
    --muted: 240 12% 20%;
    --muted-foreground: 215 20% 55%;
    --accent: 262 83% 58%;
    --accent-foreground: 0 0% 100%;
    --destructive: 0 62% 50%;
    --destructive-foreground: 0 0% 100%;
    --border: 240 12% 20%;
    --input: 240 12% 20%;
    --ring: 262 83% 58%;
    --radius: 0.75rem;
  }

  * {
    @apply border-border;
  }

  html, body, #root {
    @apply h-full w-full m-0 p-0;
    background-color: #0a0a0f;
    color: #e2e8f0;
    overflow: hidden;
  }

  /* Кастомный скроллбар */
  ::-webkit-scrollbar {
    width: 6px;
    height: 6px;
  }
  ::-webkit-scrollbar-track {
    background: transparent;
  }
  ::-webkit-scrollbar-thumb {
    background: rgba(124, 58, 237, 0.3);
    border-radius: 3px;
  }
  ::-webkit-scrollbar-thumb:hover {
    background: rgba(124, 58, 237, 0.5);
  }
}

@layer components {
  /* Класс для glassmorphism эффекта */
  .glass-panel {
    @apply backdrop-blur-xl bg-aurora-glass border border-white/10;
    box-shadow: 
      0 8px 32px rgba(0, 0, 0, 0.3),
      inset 0 1px 0 rgba(255, 255, 255, 0.05);
  }

  .glass-panel-strong {
    @apply backdrop-blur-2xl bg-black/60 border border-white/10;
    box-shadow: 
      0 8px 32px rgba(0, 0, 0, 0.4),
      inset 0 1px 0 rgba(255, 255, 255, 0.08);
  }

  /* Анимация для активной строки lyrics */
  .lyrics-active {
    @apply text-aurora-accent-light font-semibold;
    text-shadow: 0 0 20px rgba(167, 139, 250, 0.5);
    transform: scale(1.05);
    transition: all 0.3s ease;
  }
}
```

---

### 1.9 Настройка tsconfig.json

Убедись, что `tsconfig.json` содержит правильные настройки:
```json
{
  "compilerOptions": {
    "target": "ES2020",
    "useDefineForClassFields": true,
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "skipLibCheck": true,
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "jsx": "react-jsx",
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["src"],
  "references": [{ "path": "./tsconfig.node.json" }]
}
```

---

### 1.10 Настройка Tauri (`src-tauri/tauri.conf.json`)

Это главный конфигурационный файл Tauri. В нём мы разрешаем доступ к файловой системе и диалоговым окнам:

```json
{
  "$schema": "https://schema.tauri.app/config/2",
  "productName": "Aurora Music Player",
  "version": "1.0.0",
  "identifier": "com.aurora.musicplayer",
  "build": {
    "beforeDevCommand": "npm run dev",
    "beforeBuildCommand": "npm run build",
    "devUrl": "http://localhost:5173",
    "frontendDist": "../dist"
  },
  "app": {
    "windows": [
      {
        "title": "Aurora Music Player",
        "width": 1200,
        "height": 800,
        "minWidth": 900,
        "minHeight": 600,
        "decorations": true,
        "transparent": false,
        "resizable": true,
        "fullscreen": false,
        "center": true
      }
    ],
    "security": {
      "csp": null,
      "capabilities": []
    }
  },
  "bundle": {
    "active": true,
    "targets": "all",
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

### 1.11 Настройка capabilities (Разрешения безопасности)

Tauri v2 использует систему capabilities для разрешений. Создай файл:

**`src-tauri/capabilities/default.json`**:
```json
{
  "$schema": "../gen/schemas/desktop-schema.json",
  "identifier": "default",
  "description": "Default capabilities for Aurora Music Player",
  "windows": ["main"],
  "permissions": [
    "core:default",
    "dialog:default",
    "dialog:allow-open",
    "fs:default",
    "fs:allow-read-file",
    "fs:allow-read-dir",
    "fs:allow-app-read",
    "fs:allow-app-write",
    "fs:allow-app-meta",
    "sql:default",
    "sql:allow-execute",
    "sql:allow-select"
  ]
}
```

---

### 1.12 Первый запуск (Проверка)

После всех настроек запусти приложение в режиме разработки:

```bash
# В терминале, из папки проекта:
cargo tauri dev
```

Если всё настроено правильно — откроется окно приложения с заголовком "Aurora Music Player" и белым экраном (пока нет UI).

> **Важно:** При первом запуске Tauri скачает и скомпилирует все Rust-зависимости — это может занять **5–15 минут**. Последующие запуски будут быстрыми.

---

## ✅ Что мы сделали в ШАГЕ 1

| Компонент | Статус |
|-----------|--------|
| Rust (rustc + cargo) | ✅ Установлен |
| Node.js + npm | ✅ Установлен |
| Tauri CLI | ✅ Установлен |
| Проект Vite + React + TS | ✅ Создан |
| Tauri инициализирован | ✅ `src-tauri/` создана |
| Tailwind CSS | ✅ Настроен |
| shadcn/ui компоненты | ✅ Установлены |
| Howler.js, Framer Motion | ✅ Установлены |
| Tauri плагины (dialog, fs, sql) | ✅ Установлены |
| Тема Aurora (тёмная + glassmorphism) | ✅ Настроена |
| Первый запуск | ✅ Проверен |

---

**Напиши "далее" — и я продолжу с ШАГА 2: Структура проекта и дерево файлов.**
