# ШАГ 4: Настройка SQLite Базы Данных (через tauri-plugin-sql)

Tauri v2 предоставляет официальный плагин `tauri-plugin-sql` для работы с SQLite. База данных хранится локально в файле — идеально для нашего офлайн-плеера.

---

## 4.1 Как работает tauri-plugin-sql

```
┌─────────────────┐     invoke()     ┌──────────────────┐
│  React Frontend │  ──────────────► │  Rust Backend    │
│  (db.ts хелпер) │                  │  (tauri-plugin-  │
│                 │  ◄────────────── │   sql → SQLite)  │
└─────────────────┘    результат     └──────────────────┘
                                            │
                                            ▼
                                    ~/aurora-music-player/
                                        aurora.db
```

- Фронтенд вызывает SQL-запросы через `invoke()`
- Плагин выполняет их в Rust через `libsqlite3`
- База данных — обычный `.db` файл в папке приложения

---

## 4.2 SQL Схема — Таблица `tracks`

Создай файл `src-tauri/migrations/001_initial.sql`:

```sql
-- === Миграция 001: Начальная схема ===
-- Таблица треков — хранит всю информацию о музыкальных файлах

CREATE TABLE IF NOT EXISTS tracks (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,  -- Уникальный ID
    title           TEXT NOT NULL,                       -- Название трека
    artist          TEXT NOT NULL DEFAULT 'Unknown Artist', -- Исполнитель
    album           TEXT DEFAULT '',                     -- Название альбома
    file_path       TEXT NOT NULL UNIQUE,                -- Абсолютный путь (уникальный!)
    cover_path      TEXT DEFAULT NULL,                   -- Путь к файлу обложки
    lyrics_lrc      TEXT DEFAULT NULL,                   -- LRC текст (сырой)
    duration_seconds REAL DEFAULT 0,                     -- Длительность в секундах
    duration_formatted TEXT DEFAULT '0:00',              -- Длительность "mm:ss"
    file_size       INTEGER DEFAULT 0,                   -- Размер файла в байтах
    format          TEXT DEFAULT '',                     -- Расширение (mp3, flac, wav)
    is_lossless     INTEGER DEFAULT 0,                   -- 1 = lossless, 0 = lossy
    bitrate_kbps    INTEGER DEFAULT NULL,                -- Битрейт
    sample_rate     INTEGER DEFAULT NULL,                -- Частота дискретизации
    channels        TEXT DEFAULT NULL,                   -- Каналы (Stereo/Mono)
    play_count      INTEGER DEFAULT 0,                   -- Сколько раз проигран
    created_at      INTEGER DEFAULT (unixepoch()),       -- Дата добавления
    updated_at      INTEGER DEFAULT (unixepoch())        -- Дата обновления
);

-- Индексы для быстрого поиска
CREATE INDEX IF NOT EXISTS idx_tracks_artist ON tracks(artist);
CREATE INDEX IF NOT EXISTS idx_tracks_album ON tracks(album);
CREATE INDEX IF NOT EXISTS idx_tracks_title ON tracks(title);
```

**Почему такая структура:**
- `file_path UNIQUE` — нельзя добавить один файл дважды
- `cover_path` — путь к скопированной обложке (не оригинал!)
- `lyrics_lrc` — храним LRC как текст, парсим на фронтенде
- `is_lossless INTEGER` — SQLite нет boolean, 1=true, 0=false
- `unixepoch()` — timestamp в секундах

---

## 4.3 Инициализация БД при запуске

Добавь в `src-tauri/src/main.rs` инициализацию БД:

```rust
// === Обновлённый main.rs с инициализацией БД ===

#![cfg_attr(not(debug_assertions), windows_subsystem = "windows")]

mod commands;

use commands::scan::scan_directory_command;
use commands::metadata::read_audio_metadata_command;

// Импортируем плагин SQL
use tauri_plugin_sql::{Migration, MigrationKind};

fn main() {
    // === Миграции базы данных ===
    // При первом запуске создаст таблицу tracks
    let migrations = vec![
        Migration {
            version: 1,
            description: "create_tracks_table",
            sql: include_str!("../migrations/001_initial.sql"),
            kind: MigrationKind::Up,
        },
    ];

    tauri::Builder::default()
        // === Подключаем плагины ===
        .plugin(
            tauri_plugin_sql::Builder::default()
                .add_migrations("sqlite:aurora.db", migrations) // Файл aurora.db
                .build()
        )
        .plugin(tauri_plugin_dialog::init())
        .plugin(tauri_plugin_fs::init())
        .plugin(tauri_plugin_log::Builder::new().build())
        
        // === Регистрируем команды ===
        .invoke_handler(tauri::generate_handler![
            scan_directory_command,
            read_audio_metadata_command,
        ])
        
        .run(tauri::generate_context!())
        .expect("Ошибка запуска Tauri приложения");
}
```

> **Как это работает:** При первом запуске приложения, плагин автоматически создаст файл `aurora.db` в папке приложения и выполнит SQL из `001_initial.sql`. При последующих запусках миграция НЕ выполнится повторно (версия 1 уже применена).

---

## 4.4 TypeScript типы для трека — `src/types/track.ts`

```typescript
// === track.ts — TypeScript интерфейс трека ===
// Отражает структуру таблицы tracks в SQLite

export interface Track {
  id: number;                   // ID из БД
  title: string;                // Название
  artist: string;               // Исполнитель
  album: string;                // Альбом
  filePath: string;             // Путь к аудиофайлу
  coverPath: string | null;     // Путь к обложке
  lyricsLrc: string | null;     // LRC текст
  durationSeconds: number;      // Длительность (сек)
  durationFormatted: string;    // "mm:ss"
  fileSize: number;             // Размер (байт)
  format: string;               // mp3 / flac / wav
  isLossless: boolean;          // Lossless?
  bitrateKbps: number | null;   // Битрейт
  sampleRate: number | null;    // Частота дискретизации
  channels: string | null;      // Stereo / Mono
  playCount: number;            // Счётчик проигрываний
  createdAt: number;            // Timestamp
  updatedAt: number;            // Timestamp
}

// Для создания нового трека (без id)
export type TrackInsert = Omit<Track, 'id' | 'createdAt' | 'updatedAt' | 'playCount'>;

// Для обновления трека (все поля опциональны)
export type TrackUpdate = Partial<Omit<Track, 'id' | 'filePath' | 'createdAt'>>;
```

**`src/types/lyrics.ts`:**
```typescript
// === lyrics.ts — Типы для синхронизированного текста ===

// Одна строка LRC после парсинга
export interface LyricsLine {
  time: number;     // Время в секундах (с миллисекундами)
  text: string;     // Текст строки
}

// Состояние lyrics в плеере
export interface LyricsState {
  lines: LyricsLine[];      // Все строки
  currentIndex: number;     // Индекс текущей активной строки
  hasLyrics: boolean;       // Есть ли текст у текущего трека
}
```

**`src/types/index.ts`:**
```typescript
// === Реэкспорт всех типов ===
export * from './track';
export * from './lyrics';
```

---

## 4.5 JavaScript обёртка для БД — `src/lib/db.ts`

Этот файл — единая точка доступа к базе данных из React:

```typescript
// === db.ts — Обёртка для работы с SQLite через tauri-plugin-sql ===
// Все SQL-запросы выполняются через invoke() → Rust → SQLite

import { invoke } from '@tauri-apps/api/core';
import type { Track, TrackInsert, TrackUpdate } from '@/types';

// === Константы ===
const DB_PATH = 'sqlite:aurora.db';  // Путь к файлу БД (tauri-plugin-sql формат)

// === Вспомогательная функция: выполнить SQL-запрос ===
async function execute(sql: string, params: (string | number | null)[] = []): Promise<void> {
    await invoke('plugin:sql|execute', {
        db: DB_PATH,
        query: sql,
        values: params,
    });
}

// === Вспомогательная функция: выполнить SELECT и получить результат ===
async function select<T>(sql: string, params: (string | number | null)[] = []): Promise<T[]> {
    const result = await invoke<Array<Record<string, unknown>>>('plugin:sql|select', {
        db: DB_PATH,
        query: sql,
        values: params,
    });
    
    // Конвертируем snake_case из БД в camelCase для TypeScript
    return result.map(row => snakeToCamelCase(row)) as T[];
}

// === CRUD Операции ===

/**
 * Добавить новый трек в базу данных
 * @param track Данные трека (без id, createdAt, updatedAt)
 * @returns ID созданного трека
 */
export async function insertTrack(track: TrackInsert): Promise<number> {
    await execute(
        `INSERT INTO tracks 
         (title, artist, album, file_path, cover_path, lyrics_lrc, 
          duration_seconds, duration_formatted, file_size, format, 
          is_lossless, bitrate_kbps, sample_rate, channels)
         VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)`,
        [
            track.title,
            track.artist,
            track.album,
            track.filePath,
            track.coverPath,
            track.lyricsLrc,
            track.durationSeconds,
            track.durationFormatted,
            track.fileSize,
            track.format,
            track.isLossless ? 1 : 0,
            track.bitrateKbps,
            track.sampleRate,
            track.channels,
        ]
    );
    
    // Получаем ID последней вставленной записи
    const result = await select<{ id: number }>(
        'SELECT last_insert_rowid() as id'
    );
    return result[0]?.id ?? -1;
}

/**
 * Получить все треки из базы (с сортировкой по названию)
 */
export async function getAllTracks(): Promise<Track[]> {
    return select<Track>(
        'SELECT * FROM tracks ORDER BY title COLLATE NOCASE ASC'
    );
}

/**
 * Получить трек по ID
 */
export async function getTrackById(id: number): Promise<Track | null> {
    const results = await select<Track>(
        'SELECT * FROM tracks WHERE id = ?',
        [id]
    );
    return results[0] ?? null;
}

/**
 * Обновить метаданные трека (редактирование)
 */
export async function updateTrack(id: number, update: TrackUpdate): Promise<void> {
    // Строим динамический UPDATE (только переданные поля)
    const fields: string[] = [];
    const values: (string | number | null)[] = [];
    
    if (update.title !== undefined) { fields.push('title = ?'); values.push(update.title); }
    if (update.artist !== undefined) { fields.push('artist = ?'); values.push(update.artist); }
    if (update.album !== undefined) { fields.push('album = ?'); values.push(update.album); }
    if (update.coverPath !== undefined) { fields.push('cover_path = ?'); values.push(update.coverPath); }
    if (update.lyricsLrc !== undefined) { fields.push('lyrics_lrc = ?'); values.push(update.lyricsLrc); }
    
    if (fields.length === 0) return;
    
    // Добавляем updated_at
    fields.push('updated_at = unixepoch()');
    values.push(id);
    
    await execute(
        `UPDATE tracks SET ${fields.join(', ')} WHERE id = ?`,
        values
    );
}

/**
 * Удалить трек из базы (файл НЕ удаляется!)
 */
export async function deleteTrack(id: number): Promise<void> {
    await execute('DELETE FROM tracks WHERE id = ?', [id]);
}

/**
 * Увеличить счётчик проигрываний
 */
export async function incrementPlayCount(id: number): Promise<void> {
    await execute(
        'UPDATE tracks SET play_count = play_count + 1 WHERE id = ?',
        [id]
    );
}

/**
 * Поиск треков по названию или исполнителю
 */
export async function searchTracks(query: string): Promise<Track[]> {
    const searchPattern = `%${query}%`;
    return select<Track>(
        `SELECT * FROM tracks 
         WHERE title LIKE ? OR artist LIKE ? OR album LIKE ?
         ORDER BY title COLLATE NOCASE ASC`,
        [searchPattern, searchPattern, searchPattern]
    );
}

/**
 * Проверить, существует ли трек с таким путём
 */
export async function trackExists(filePath: string): Promise<boolean> {
    const results = await select<{ count: number }>(
        'SELECT COUNT(*) as count FROM tracks WHERE file_path = ?',
        [filePath]
    );
    return (results[0]?.count ?? 0) > 0;
}

// === Утилита: Конвертация snake_case → camelCase ===
function snakeToCamelCase(obj: Record<string, unknown>): Record<string, unknown> {
    const result: Record<string, unknown> = {};
    for (const [key, value] of Object.entries(obj)) {
        // Конвертируем is_lossless → isLossless, file_path → filePath и т.д.
        const camelKey = key.replace(/_([a-z])/g, (_, letter) => letter.toUpperCase());
        result[camelKey] = value;
    }
    return result;
}
```

---

## 4.6 Импорт треков — сканирование + сохранение в БД

**`src/lib/importTracks.ts`** — полный цикл импорта:

```typescript
// === importTracks.ts — Импорт треков из папки в БД ===
// 1. Открывает диалог выбора папки
// 2. Сканирует через Rust
// 3. Читает метаданные каждого файла
// 4. Сохраняет в SQLite

import { invoke } from '@tauri-apps/api/core';
import { open } from '@tauri-apps/plugin-dialog';
import type { TrackInsert } from '@/types';
import { insertTrack, trackExists } from './db';

export interface ImportResult {
    imported: number;      // Сколько добавлено
    skipped: number;       // Сколько пропущено (уже есть)
    errors: string[];      // Ошибки
}

/**
 * Открывает диалог выбора папки и импортирует все аудиофайлы
 */
export async function importFolder(): Promise<ImportResult> {
    const result: ImportResult = { imported: 0, skipped: 0, errors: [] };
    
    // 1. Открываем диалог выбора папки (Tauri v2 API)
    const folderPath = await open({
        directory: true,           // Выбираем папку, не файл
        multiple: false,           // Одна папка
        title: 'Выберите папку с музыкой',
    });
    
    if (!folderPath || typeof folderPath !== 'string') {
        return result;  // Пользователь отменил
    }
    
    // 2. Сканируем папку через Rust
    const scanResult = await invoke<{
        files: Array<{ path: string; filename: string; extension: string; size: number }>;
        totalFound: number;
        errors: string[];
    }>('scan_directory_command', { directoryPath: folderPath });
    
    // 3. Обрабатываем каждый файл
    for (const file of scanResult.files) {
        try {
            // Проверяем, нет ли уже такого файла в БД
            if (await trackExists(file.path)) {
                result.skipped++;
                continue;
            }
            
            // Читаем метаданные через Rust (Symphonia)
            const metadata = await invoke<{
                title: string;
                artist: string;
                album: string;
                durationSeconds: number;
                durationFormatted: string;
                bitrateKbps?: number;
                sampleRate?: number;
                channels?: string;
                format: string;
                fileSize: number;
                isLossless: boolean;
            }>('read_audio_metadata_command', { filePath: file.path });
            
            // Формируем объект для вставки
            const track: TrackInsert = {
                title: metadata.title || file.filename,
                artist: metadata.artist || 'Unknown Artist',
                album: metadata.album || '',
                filePath: file.path,
                coverPath: null,
                lyricsLrc: null,
                durationSeconds: metadata.durationSeconds,
                durationFormatted: metadata.durationFormatted,
                fileSize: file.size,
                format: metadata.format,
                isLossless: metadata.isLossless,
                bitrateKbps: metadata.bitrateKbps ?? null,
                sampleRate: metadata.sampleRate ?? null,
                channels: metadata.channels ?? null,
            };
            
            // Сохраняем в БД
            await insertTrack(track);
            result.imported++;
            
        } catch (error) {
            result.errors.push(`Ошибка "${file.filename}": ${error}`);
        }
    }
    
    return result;
}

/**
 * Drag & Drop импорт — принимает массив путей из события drop
 */
export async function importDroppedPaths(paths: string[]): Promise<ImportResult> {
    const result: ImportResult = { imported: 0, skipped: 0, errors: [] };
    
    for (const path of paths) {
        try {
            if (await trackExists(path)) {
                result.skipped++;
                continue;
            }
            
            const metadata = await invoke<{
                title: string; artist: string; album: string;
                durationSeconds: number; durationFormatted: string;
                bitrateKbps?: number; sampleRate?: number;
                channels?: string; format: string;
                fileSize: number; isLossless: boolean;
            }>('read_audio_metadata_command', { filePath: path });
            
            await insertTrack({
                title: metadata.title || path.split('/').pop() || 'Unknown',
                artist: metadata.artist || 'Unknown Artist',
                album: metadata.album || '',
                filePath: path,
                coverPath: null,
                lyricsLrc: null,
                durationSeconds: metadata.durationSeconds,
                durationFormatted: metadata.durationFormatted,
                fileSize: metadata.fileSize,
                format: metadata.format,
                isLossless: metadata.isLossless,
                bitrateKbps: metadata.bitrateKbps ?? null,
                sampleRate: metadata.sampleRate ?? null,
                channels: metadata.channels ?? null,
            });
            result.imported++;
            
        } catch (error) {
            result.errors.push(`Ошибка "${path}": ${error}`);
        }
    }
    
    return result;
}
```

---

## 4.7 Проверка работы БД

Добавь тестовый вызов в `App.tsx` (временно, для проверки):

```tsx
// Временный тест в App.tsx
import { useEffect } from 'react';
import { getAllTracks } from '@/lib/db';

useEffect(() => {
    // Проверяем подключение к БД
    getAllTracks().then(tracks => {
        console.log('Треков в базе:', tracks.length);
    }).catch(err => {
        console.error('Ошибка БД:', err);
    });
}, []);
```

Запусти `cargo tauri dev` — если в консоли нет ошибок SQLite, БД работает!

---

## ✅ Итог ШАГА 4

| Компонент | Описание |
|-----------|----------|
| `001_initial.sql` | Схема таблицы tracks с 16 полями + индексы |
| `main.rs` (обновлённый) | Инициализация БД + миграции при старте |
| `src/types/track.ts` | TypeScript интерфейс Track |
| `src/types/lyrics.ts` | Типы для LRC строк |
| `src/lib/db.ts` | CRUD: insert, select, update, delete, search |
| `src/lib/importTracks.ts` | Полный цикл: диалог → сканирование → метаданные → БД |

---

**Напиши "далее" — перейдём к ШАГУ 5: Фронтенд UI (плеер с glassmorphism, список треков, модалка редактирования).**
