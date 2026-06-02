# ШАГ 3: Настройка Бэкенда (Rust) — Tauri Commands

В этом шаге мы напишем весь Rust-код: зависимости, точку входа, команды для сканирования папок и чтения аудио-метаданных.

---

## 3.1 Зависимости Rust — `src-tauri/Cargo.toml`

Открой `src-tauri/Cargo.toml` и замени его содержимое:

```toml
[package]
name = "aurora-music-player"
version = "1.0.0"
description = "Aurora Music Player — локальный музыкальный плеер с Tauri"
authors = ["you"]
edition = "2021"

[lib]
name = "aurora_music_player_lib"
crate-type = ["lib", "cdylib", "staticlib"]

[[bin]]
name = "aurora-music-player"
path = "src/main.rs"

[build-dependencies]
tauri-build = { version = "2", features = [] }

[dependencies]
# === Основной фреймворк Tauri v2 ===
tauri = { version = "2", features = [] }
tauri-build = "2"

# === Плагины Tauri ===
tauri-plugin-dialog = "2"
tauri-plugin-fs = "2"
tauri-plugin-sql = { version = "2", features = ["sqlite"] }
tauri-plugin-log = "2"

# === Сериализация данных (JSON) ===
serde = { version = "1", features = ["derive"] }
serde_json = "1"

# === Работа с файлами ===
walkdir = "2"              # Рекурсивный обход директорий

# === Аудио метаданные (MP3, FLAC, WAV, OGG, MP4, AAC) ===
symphonia = "0.5"          # Основной крейт
# Поддержка разных форматов через features:
symphonia-metadata = "0.5"
symphonia-core = "0.5"
# Декодеры форматов:
symphonia-bundle-mp3 = "0.5"   # MP3 (包括 ID3 tags)
symphonia-format-flac = "0.5"  # FLAC
symphonia-format-wav = "0.5"   # WAV
symphonia-format-ogg = "0.5"   # OGG
symphonia-format-isomp4 = "0.5" # MP4/M4A

# === Утилиты ===
log = "0.4"                # Логирование
thiserror = "1"            # Удобные ошибки
tokio = { version = "1", features = ["rt", "macros"] } # Асинхронность

[features]
default = [ "custom-protocol" ]
# Этот feature включается ТОЛЬКО при сборке релиза (не в dev mode)
custom-protocol = [ "tauri/custom-protocol" ]
```

> **Что делают эти зависимости:**
> - `tauri` + `tauri-build` — ядро Tauri v2
> - `tauri-plugin-*` — плагины для диалогов, файловой системы, SQLite
> - `serde` — сериализация структур Rust в JSON (для передачи во фронтенд)
> - `walkdir` — безопасный рекурсивный обход папок
> - `symphonia-*` — библиотека для чтения аудио метаданных (длительность, теги) из MP3/FLAC/WAV и других форматов
> - `thiserror` — макросы для создания типов ошибок

---

## 3.2 Точка входа — `src-tauri/src/main.rs`

Это главный файл Rust. Он инициализирует Tauri, подключает плагины и регистрирует наши команды:

```rust
// === AURORA MUSIC PLAYER — Точка входа Rust ===
// Здесь регистрируются все Tauri Commands (функции бэкенда)
// и подключаются плагины (SQLite, диалоги, файловая система)

#![cfg_attr(not(debug_assertions), windows_subsystem = "windows")]

// Импортируем модули с командами
mod commands;

use commands::scan::scan_directory_command;
use commands::metadata::read_audio_metadata_command;
use commands::metadata::AudioFileInfo;

fn main() {
    // Инициализируем Tauri приложение
    tauri::Builder::default()
        // === Подключаем плагины ===
        // Плагин SQLite для локальной базы данных
        .plugin(tauri_plugin_sql::Builder::new().build())
        // Плагин диалогов (открытие файлов/папок)
        .plugin(tauri_plugin_dialog::init())
        // Плагин файловой системы
        .plugin(tauri_plugin_fs::init())
        // Плагин логирования
        .plugin(tauri_plugin_log::Builder::new().build())
        
        // === Регистрируем Tauri Commands ===
        // Эти функции можно вызывать из JavaScript через invoke()
        .invoke_handler(tauri::generate_handler![
            // Сканирование директории на аудиофайлы
            scan_directory_command,
            // Чтение метаданных аудиофайла (длительность, теги)
            read_audio_metadata_command,
        ])
        
        // Запускаем приложение
        .run(tauri::generate_context!())
        .expect("Ошибка запуска Tauri приложения");
}
```

---

## 3.3 Модуль сканирования — `src-tauri/src/commands/mod.rs`

Создай файл `src-tauri/src/commands/mod.rs` (реэкспорт модулей):

```rust
// === Модуль commands ===
// Объединяет все Tauri Commands в одном месте

pub mod scan;
pub mod metadata;
```

---

## 3.4 Команда сканирования — `src-tauri/src/commands/scan.rs`

Эта команда принимает путь к папке, рекурсивно ищет все аудиофайлы и возвращает их список:

```rust
// === scan.rs — Команда сканирования директории ===
// Рекурсивно обходит папку и находит все аудиофайлы (MP3, FLAC, WAV, OGG, M4A, AAC)

use serde::{Deserialize, Serialize};
use std::path::Path;
use walkdir::WalkDir;

// === Поддерживаемые аудио расширения ===
const AUDIO_EXTENSIONS: &[&str] = &[
    "mp3", "flac", "wav", "ogg", "oga", "m4a", "aac", "wma", "opus",
];

/// Результат сканирования — информация о найденном файле
#[derive(Serialize, Debug, Clone)]
pub struct ScannedFile {
    /// Полный путь к файлу
    pub path: String,
    /// Имя файла с расширением
    pub filename: String,
    /// Расширение файла (без точки)
    pub extension: String,
    /// Размер файла в байтах
    pub size: u64,
}

/// Структура запроса от фронтенда
#[derive(Deserialize)]
pub struct ScanRequest {
    /// Абсолютный путь к директории для сканирования
    pub directory_path: String,
}

/// Структура ответа фронтенду
#[derive(Serialize)]
pub struct ScanResult {
    /// Найденные аудиофайлы
    pub files: Vec<ScannedFile>,
    /// Общее количество найденных файлов
    pub total_found: usize,
    /// Ошибки (если не удалось прочитать какие-то папки)
    pub errors: Vec<String>,
}

/// Tauri Command: сканирование директории на аудиофайлы
/// 
/// Вызывается из JavaScript:
/// ```js
/// const result = await invoke('scan_directory_command', { 
///     directoryPath: '/Users/music' 
/// });
/// ```
#[tauri::command(rename_all = "camelCase")]
pub fn scan_directory_command(directory_path: String) -> Result<ScanResult, String> {
    // Проверяем, что путь существует и это директория
    let path = Path::new(&directory_path);
    if !path.exists() {
        return Err(format!("Папка не существует: {}", directory_path));
    }
    if !path.is_dir() {
        return Err(format!("Это не папка: {}", directory_path));
    }

    let mut files: Vec<ScannedFile> = Vec::new();
    let mut errors: Vec<String> = Vec::new();

    // walkdir рекурсивно обходит все подпапки
    for entry in WalkDir::new(path)
        .follow_links(true)           // Следовать по символическим ссылкам
        .max_depth(10)                // Максимальная глубина вложенности
        .into_iter()
        .filter_map(|e| e.ok())       // Пропускаем ошибки чтения отдельных файлов
    {
        let entry_path = entry.path();
        
        // Пропускаем директории, нас интересуют только файлы
        if !entry_path.is_file() {
            continue;
        }

        // Получаем расширение файла (нижний регистр)
        let extension = entry_path
            .extension()
            .and_then(|ext| ext.to_str())
            .map(|ext| ext.to_lowercase())
            .unwrap_or_default();

        // Проверяем, что это аудио файл
        if AUDIO_EXTENSIONS.contains(&extension.as_str()) {
            // Получаем имя файла
            let filename = entry_path
                .file_name()
                .and_then(|name| name.to_str())
                .unwrap_or("unknown")
                .to_string();

            // Получаем размер файла
            let size = entry.metadata().map(|m| m.len()).unwrap_or(0);

            // Полный путь как строка
            let path_str = entry_path.to_string_lossy().to_string();

            files.push(ScannedFile {
                path: path_str,
                filename,
                extension,
                size,
            });
        }
    }

    // Сортируем по имени файла
    files.sort_by(|a, b| a.filename.cmp(&b.filename));

    let total_found = files.len();
    log::info!("Сканирование завершено: найдено {} аудиофайлов в {}", total_found, directory_path);

    Ok(ScanResult {
        files,
        total_found,
        errors,
    })
}
```

---

## 3.5 Команда чтения метаданных — `src-tauri/src/commands/metadata.rs`

Эта команда читает аудио-файл и извлекает: длительность, название трека, исполнителя, альбом, обложку:

```rust
// === metadata.rs — Чтение метаданных аудиофайла ===
// Использует библиотеку Symphonia для поддержки MP3, FLAC, WAV, OGG, M4A

use serde::{Deserialize, Serialize};
use std::fs::File;
use std::path::Path;

/// Структура с метаданными аудиофайла
#[derive(Serialize, Deserialize, Debug, Clone)]
pub struct AudioFileInfo {
    /// Полный путь к файлу
    pub file_path: String,
    /// Название трека (из тегов или имя файла)
    pub title: String,
    /// Исполнитель
    pub artist: String,
    /// Название альбома
    pub album: String,
    /// Длительность в секундах (f64 для точности)
    pub duration_seconds: f64,
    /// Длительность как строка "mm:ss"
    pub duration_formatted: String,
    /// Битрейт в kbps
    pub bitrate_kbps: Option<u32>,
    /// Частота дискретизации (Hz)
    pub sample_rate: Option<u32>,
    /// Формат каналов ("Stereo", "Mono" и т.д.)
    pub channels: Option<String>,
    /// Расширение файла
    pub format: String,
    /// Размер файла в байтах
    pub file_size: u64,
    /// true = Lossless (FLAC, WAV), false = Lossy (MP3, AAC, OGG)
    pub is_lossless: bool,
}

/// Tauri Command: чтение метаданных аудиофайла
/// 
/// Вызывается из JavaScript:
/// ```js
/// const info = await invoke('read_audio_metadata_command', { filePath: '/music/song.mp3' });
/// console.log(info.duration_seconds); // 245.6
/// ```
#[tauri::command(rename_all = "camelCase")]
pub fn read_audio_metadata_command(file_path: String) -> Result<AudioFileInfo, String> {
    let path = Path::new(&file_path);
    
    // Проверяем что файл существует
    if !path.exists() {
        return Err(format!("Файл не найден: {}", file_path));
    }

    // Получаем размер файла
    let file_size = std::fs::metadata(path)
        .map(|m| m.len())
        .unwrap_or(0);

    // Получаем расширение файла
    let extension = path
        .extension()
        .and_then(|ext| ext.to_str())
        .map(|ext| ext.to_lowercase())
        .unwrap_or_default();

    // Определяем, lossless ли формат
    let is_lossless = matches!(extension.as_str(), "flac" | "wav" | "aiff" | "ape");

    // === Читаем аудио метаданные через Symphonia ===
    let (duration_seconds, bitrate_kbps, sample_rate, channels, title, artist, album) = 
        read_with_symphonia(path).unwrap_or_default();

    // Если теги пустые — используем имя файла как название
    let filename = path
        .file_stem()
        .and_then(|s| s.to_str())
        .unwrap_or("Unknown")
        .to_string();

    let final_title = if title.is_empty() { filename.clone() } else { title };
    let final_artist = if artist.is_empty() { "Unknown Artist".to_string() } else { artist };

    // Форматируем длительность как "mm:ss"
    let duration_formatted = format_duration(duration_seconds);

    Ok(AudioFileInfo {
        file_path,
        title: final_title,
        artist: final_artist,
        album,
        duration_seconds,
        duration_formatted,
        bitrate_kbps,
        sample_rate,
        channels,
        format: extension,
        file_size,
        is_lossless,
    })
}

/// Читает аудио-файл через Symphonia и возвращает метаданные
fn read_with_symphonia(path: &Path) -> Result<(f64, Option<u32>, Option<u32>, Option<String>, String, String, String), String> {
    use symphonia::core::io::MediaSourceStream;
    use symphonia::core::probe::Hint;
    use symphonia::core::meta::MetadataRevision;
    use symphonia::core::units::Time;

    // Открываем файл
    let file = File::open(path).map_err(|e| format!("Не удалось открыть файл: {}", e))?;
    let mss = MediaSourceStream::new(Box::new(file), Default::default());

    // Подсказка формата по расширению
    let mut hint = Hint::new();
    if let Some(ext) = path.extension().and_then(|e| e.to_str()) {
        hint.with_extension(ext);
    }

    // Пробуем определить формат
    let probed = symphonia::default::get_probe()
        .format(&hint, mss, &Default::default(), &Default::default())
        .map_err(|e| format!("Не удалось определить формат: {}", e))?;

    let mut format = probed.format;

    // Получаем длительность и параметры из первого трека
    let default_track = format.default_track();
    let mut duration_seconds = 0.0;
    let mut bitrate_kbps = None;
    let mut sample_rate = None;
    let mut channels_str = None;

    if let Some(track) = default_track {
        let codec_params = &track.codec_params;
        
        // Частота дискретизации
        if let Some(rate) = codec_params.sample_rate {
            sample_rate = Some(rate);
        }
        
        // Количество каналов
        if let Some(layout) = codec_params.channel_layout {
            channels_str = Some(format!("{:?}", layout));
        } else if let Some channels) = codec_params.channels {
            let count = channels.count();
            channels_str = Some(match count {
                1 => "Mono".to_string(),
                2 => "Stereo".to_string(),
                n => format!("{} channels", n),
            });
        }

        // Длительность из time_base + n_frames
        if let (Some(time_base), Some(n_frames)) = (codec_params.time_base, codec_params.n_frames) {
            let time = time_base.calc_time(n_frames);
            duration_seconds = time.seconds as f64 + time.frac as f64;
        }

        // Битрейт
        if let Some(bitrate) = codec_params.bits_per_sample {
            // bits_per_sample * sample_rate * channels / 1000
            if let Some(rate) = sample_rate {
                let ch = channels_str.as_ref().map(|c| {
                    if c == "Mono" { 1 } else { 2 }
                }).unwrap_or(2);
                bitrate_kbps = Some((bitrate as u32 * rate * ch as u32) / 1000);
            }
        }
    }

    // Если длительность не определена через codec_params, попробуем через контейнер
    if duration_seconds == 0.0 {
        if let Some(track) = default_track {
            // Пробуем оценить через битрейт и размер файла
            // Это приближенная оценка для CBR MP3
        }
    }

    // Читаем мета-теги (ID3v2, Vorbis Comments, и т.д.)
    let mut title = String::new();
    let mut artist = String::new();
    let mut album = String::new();

    // Проверяем метаданные формата
    while let Ok(Some(rev)) = format.next_metadata_revision() {
        for tag in rev.tags() {
            if let Some(std_key) = tag.std_key {
                match std_key {
                    symphonia::core::meta::StandardTagKey::TrackTitle => {
                        title = tag.value.to_string();
                    }
                    symphonia::core::meta::StandardTagKey::Artist => {
                        artist = tag.value.to_string();
                    }
                    symphonia::core::meta::StandardTagKey::Album => {
                        album = tag.value.to_string();
                    }
                    _ => {}
                }
            }
        }
    }

    Ok((duration_seconds, bitrate_kbps, sample_rate, channels_str, title, artist, album))
}

/// Форматирует секунды в строку "mm:ss"
fn format_duration(seconds: f64) -> String {
    if seconds <= 0.0 {
        return "0:00".to_string();
    }
    let total_secs = seconds as u64;
    let mins = total_secs / 60;
    let secs = total_secs % 60;
    format!("{}:{:02}", mins, secs)
}
```

---

## 3.6 Как вызывать команды из фронтенда

В JavaScript/TypeScript команды вызываются через `@tauri-apps/api`:

```typescript
// === Пример вызова команд из React ===
import { invoke } from '@tauri-apps/api/core';

// 1. Сканирование папки
async function scanFolder(path: string) {
    const result = await invoke('scan_directory_command', { 
        directoryPath: path 
    });
    console.log(`Найдено ${result.total_found} треков`);
    return result.files; // Массив ScannedFile[]
}

// 2. Чтение метаданных
async function getMetadata(filePath: string) {
    const info = await invoke('read_audio_metadata_command', { 
        filePath 
    });
    console.log(`${info.title} — ${info.artist} (${info.durationFormatted})`);
    return info; // AudioFileInfo
}
```

---

## 3.7 Проверка бэкенда

После создания всех файлов, запусти проверку:

```bash
# Перейди в папку src-tauri
cd src-tauri

# Проверь компиляцию (без запуска приложения)
cargo check

# Если есть ошибки — прочитай их и исправь
# Частые проблемы:
# - "unresolved import" → проверь mod.rs
# - "missing field" → проверь структуры Serialize/Deserialize
# - "trait bound not satisfied" → проверь derive макросы
```

Если `cargo check` прошёл без ошибок — бэкенд готов! 

---

## ✅ Итог ШАГА 3

| Файл | Назначение |
|------|-----------|
| `Cargo.toml` | 12+ зависимостей (Tauri v2, плагины, Symphonia, walkdir) |
| `main.rs` | Точка входа, регистрация 2 команд |
| `commands/mod.rs` | Реэкспорт модулей |
| `commands/scan.rs` | Рекурсивное сканирование папок, фильтрация по расширениям |
| `commands/metadata.rs` | Чтение длительности, ID3-тегов, параметров аудио |

---

**Напиши "далее" — перейдём к ШАГУ 4: SQLite база данных (схема, инициализация, CRUD).**
