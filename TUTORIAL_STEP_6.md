# ШАГ 6: Логика Аудио и Синхронизированный Текст (Lyrics)

Самый важный шаг! Здесь мы создаём:
- Хук `useAudioPlayer` на базе Howler.js — воспроизведение, прогресс, переключение
- Парсер LRC — конвертация строк `[mm:ss.xx]` в массив объектов
- Хук `useLyrics` — синхронизация текста с аудио (каждые 100мс)
- `LyricsViewer` — компонент с автоскроллом и подсветкой

---

## 6.1 Парсер LRC — `src/lib/lrcParser.ts`

```typescript
// === lrcParser.ts — Парсер формата LRC ===
// Конвертирует строки формата [mm:ss.xx]Текст → массив { time, text }
//
// Поддерживает:
// - [mm:ss.xx] — стандартный формат
// - [mm:ss] — без миллисекунд
// - Множественные таймстампы на одну строку: [00:01.00][00:03.00]Текст
// - Offset correction: [offset:+/-мс]

import type { LyricsLine } from '@/types';

/**
 * Парсит LRC-строку в массив LyricsLine
 * @param lrcContent — сырой текст LRC файла
 * @returns Отсортированный массив строк с таймстампами
 */
export function parseLrc(lrcContent: string): LyricsLine[] {
    if (!lrcContent || !lrcContent.trim()) {
        return [];
    }

    const lines: LyricsLine[] = [];
    let offset = 0; // Коррекция в миллисекундах (из [offset:...])

    // Разбиваем на строки
    const rawLines = lrcContent.split('\n');

    for (const line of rawLines) {
        const trimmed = line.trim();
        if (!trimmed) continue;

        // === Парсим [offset:+/-мс] ===
        if (trimmed.toLowerCase().startsWith('[offset:')) {
            const offsetMatch = trimmed.match(/\[offset:([+-]?\d+)\]/i);
            if (offsetMatch) {
                offset = parseInt(offsetMatch[1], 10);
            }
            continue;
        }

        // === Парсим таймстампы [mm:ss.xx] или [mm:ss] ===
        // Регулярка находит ВСЕ таймстампы в строке
        const timeRegex = /\[(\d{1,2}):(\d{2})\.?(\d{0,3})\]/g;
        const timestamps: number[] = [];
        let match: RegExpExecArray | null;

        while ((match = timeRegex.exec(trimmed)) !== null) {
            const minutes = parseInt(match[1], 10);
            const seconds = parseInt(match[2], 10);
            // Миллисекунды: может быть 0-3 цифры (10 → 100мс, 100 → 100мс)
            const msStr = match[3] || '0';
            const ms = parseInt(msStr.padEnd(3, '0'), 10);

            const totalSeconds = minutes * 60 + seconds + ms / 1000;
            timestamps.push(totalSeconds + offset / 1000);
        }

        // Если нет таймстампов — пропускаем (ID3 теги, комментарии и т.д.)
        if (timestamps.length === 0) continue;

        // === Извлекаем текст (всё после последнего таймстампа) ===
        const lastBracketEnd = trimmed.lastIndexOf(']') + 1;
        const text = trimmed.slice(lastBracketEnd).trim();

        // === Создаём запись для КАЖДОГО таймстампа ===
        for (const time of timestamps) {
            lines.push({ time, text });
        }
    }

    // Сортируем по времени
    lines.sort((a, b) => a.time - b.time);

    // Удаляем дубликаты (одинаковое время + текст)
    const unique: LyricsLine[] = [];
    for (const line of lines) {
        const last = unique[unique.length - 1];
        if (!last || last.time !== line.time || last.text !== line.text) {
            unique.push(line);
        }
    }

    return unique;
}

/**
 * Находит индекс текущей строки lyrics по времени аудио
 * @param lines — распарсенные строки lyrics
 * @param currentTime — текущее время аудио в секундах
 * @returns Индекс активной строки (-1 если ещё не началось)
 */
export function findCurrentLyricIndex(lines: LyricsLine[], currentTime: number): number {
    if (!lines.length || currentTime < 0) return -1;

    // Бинарный поиск для эффективности
    let left = 0;
    let right = lines.length - 1;
    let result = -1;

    while (left <= right) {
        const mid = Math.floor((left + right) / 2);
        if (lines[mid].time <= currentTime) {
            result = mid;
            left = mid + 1;
        } else {
            right = mid - 1;
        }
    }

    return result;
}

/**
 * Конвертирует массив lyrics обратно в LRC формат
 * (для сохранения после редактирования)
 */
export function lyricsToLrc(lines: LyricsLine[]): string {
    return lines
        .map(line => {
            const minutes = Math.floor(line.time / 60);
            const seconds = Math.floor(line.time % 60);
            const ms = Math.round((line.time % 1) * 100);
            return `[${minutes.toString().padStart(2, '0')}:${seconds.toString().padStart(2, '0')}.${ms.toString().padStart(2, '0')}]${line.text}`;
        })
        .join('\n');
}
```

---

## 6.2 Главный аудио-хук — `src/hooks/useAudioPlayer.ts`

```typescript
// === useAudioPlayer.ts — Главный хук управления аудио ===
// На базе Howler.js: воспроизведение, пауза, seek, громкость, прогресс
// Автоматически переключает треки, обновляет прогресс через requestAnimationFrame

import { useEffect, useRef, useCallback } from 'react';
import { Howl, Howler } from 'howler';
import { usePlayer } from '@/providers/PlayerProvider';
import { parseLrc } from '@/lib/lrcParser';

export function useAudioPlayer() {
    const { state, actions, stateRef } = usePlayer();
    const howlRef = useRef<Howl | null>(null);
    const rafRef = useRef<number | null>(null);
    const lastTrackIdRef = useRef<number | null>(null);

    // === Загрузка и воспроизведение трека ===
    useEffect(() => {
        const track = state.currentTrack;
        
        // Если трек не выбран — ничего не делаем
        if (!track) return;
        
        // Если трек не изменился — не перезагружаем
        if (lastTrackIdRef.current === track.id && howlRef.current) {
            return;
        }
        
        lastTrackIdRef.current = track.id;

        // Останавливаем предыдущий трек
        if (howlRef.current) {
            howlRef.current.stop();
            howlRef.current.unload();
        }

        // Отменяем предыдущий RAF
        if (rafRef.current) {
            cancelAnimationFrame(rafRef.current);
        }

        // === Парсим LRC если есть ===
        if (track.lyricsLrc) {
            const parsed = parseLrc(track.lyricsLrc);
            actions.setLyrics(parsed);
        } else {
            actions.setLyrics([]);
            actions.setCurrentLyricIndex(-1);
        }

        // === Создаём новый Howl ===
        const sound = new Howl({
            src: [track.filePath],      // Путь к файлу (Tauri позволяет читать локальные пути)
            html5: true,                // HTML5 Audio API (поддержка потоковой загрузки больших файлов)
            preload: true,              // Предзагрузка
            volume: state.volume,       // Текущая громкость
            format: [track.format],     // Формат для корректного декодирования

            // === Колбэки ===
            onload: () => {
                console.log(`[Audio] Загружен: ${track.title}`);
                // Автоматически начинаем играть если shouldPlay
                if (state.isPlaying) {
                    sound.play();
                }
            },

            onplay: () => {
                console.log(`[Audio] Воспроизведение: ${track.title}`);
                actions.setIsPlaying(true);
                startProgressLoop();
            },

            onpause: () => {
                actions.setIsPlaying(false);
                stopProgressLoop();
            },

            onend: () => {
                console.log('[Audio] Трек завершён');
                actions.setIsPlaying(false);
                actions.nextTrack();  // Автопереход на следующий трек
            },

            onstop: () => {
                actions.setIsPlaying(false);
                stopProgressLoop();
            },

            onloaderror: (_id, error) => {
                console.error(`[Audio] Ошибка загрузки:`, error);
                actions.setIsPlaying(false);
            },

            onplayerror: (_id, error) => {
                console.error(`[Audio] Ошибка воспроизведения:`, error);
                // Пробуем перезагрузить
                sound.once('unlock', () => {
                    sound.play();
                });
            },
        });

        howlRef.current = sound;

        // Если shouldPlay — сразу играем
        if (state.isPlaying) {
            sound.play();
        }

        // Cleanup при размонтировании или смене трека
        return () => {
            stopProgressLoop();
            // Не останавливаем при смене трека — это handled выше
        };
    }, [state.currentTrack?.id]); // Зависим только от ID трека

    // === Синхронизация play/pause ===
    useEffect(() => {
        const howl = howlRef.current;
        if (!howl) return;

        if (state.isPlaying && !howl.playing()) {
            howl.play();
        } else if (!state.isPlaying && howl.playing()) {
            howl.pause();
        }
    }, [state.isPlaying]);

    // === Синхронизация громкости ===
    useEffect(() => {
        Howler.volume(state.volume);
    }, [state.volume]);

    // === Цикл обновления прогресса (requestAnimationFrame) ===
    const startProgressLoop = useCallback(() => {
        const update = () => {
            const howl = howlRef.current;
            if (!howl || !howl.playing()) {
                rafRef.current = requestAnimationFrame(update);
                return;
            }

            const seek = howl.seek() as number;  // Текущая позиция в секундах
            const duration = howl.duration();     // Общая длительность

            if (duration > 0) {
                const progress = seek / duration;
                actions.setProgress(progress);
                actions.setCurrentTime(seek);
            }

            rafRef.current = requestAnimationFrame(update);
        };

        rafRef.current = requestAnimationFrame(update);
    }, [actions]);

    const stopProgressLoop = useCallback(() => {
        if (rafRef.current) {
            cancelAnimationFrame(rafRef.current);
            rafRef.current = null;
        }
    }, []);

    // === Seek (перемотка) ===
    const seek = useCallback((progressPercent: number) => {
        const howl = howlRef.current;
        if (!howl) return;

        const duration = howl.duration();
        if (duration > 0) {
            const seekTime = (progressPercent / 100) * duration;
            howl.seek(seekTime);
            actions.setProgress(progressPercent / 100);
            actions.setCurrentTime(seekTime);
        }
    }, [actions]);

    // Cleanup при размонтировании компонента
    useEffect(() => {
        return () => {
            stopProgressLoop();
            if (howlRef.current) {
                howlRef.current.stop();
                howlRef.current.unload();
            }
        };
    }, []);

    return { seek };
}
```

---

## 6.3 Хук синхронизации Lyrics — `src/hooks/useLyrics.ts`

```typescript
// === useLyrics.ts — Синхронизация Lyrics с аудио ===
// Каждые 100мс проверяет currentTime и обновляет активную строку
// Использует findCurrentLyricIndex (бинарный поиск)

import { useEffect, useRef } from 'react';
import { usePlayer } from '@/providers/PlayerProvider';
import { findCurrentLyricIndex } from '@/lib/lrcParser';

export function useLyrics() {
    const { state, actions } = usePlayer();
    const intervalRef = useRef<ReturnType<typeof setInterval> | null>(null);

    useEffect(() => {
        // Если нет lyrics или трек не играет — не запускаем
        if (!state.lyrics.length || !state.isPlaying) {
            return;
        }

        // Запускаем интервал обновления (100мс = 10 раз в секунду)
        intervalRef.current = setInterval(() => {
            const currentTime = state.currentTime;
            const newIndex = findCurrentLyricIndex(state.lyrics, currentTime);

            // Обновляем только если индекс изменился
            if (newIndex !== state.currentLyricIndex) {
                actions.setCurrentLyricIndex(newIndex);
            }
        }, 100);

        // Cleanup
        return () => {
            if (intervalRef.current) {
                clearInterval(intervalRef.current);
            }
        };
    }, [state.lyrics, state.isPlaying, state.currentTime, state.currentLyricIndex, actions]);

    // Также обновляем при паузе (чтобы показать правильную строку)
    useEffect(() => {
        if (!state.lyrics.length) return;
        
        const newIndex = findCurrentLyricIndex(state.lyrics, state.currentTime);
        if (newIndex !== state.currentLyricIndex) {
            actions.setCurrentLyricIndex(newIndex);
        }
    }, [state.currentTime]);
}
```

---

## 6.4 Компонент LyricsViewer — `src/components/lyrics/LyricsViewer.tsx`

```tsx
// === LyricsViewer.tsx — Отображение синхронизированного текста ===
// Подсвечивает текущую строку, плавно скроллит к ней

import React, { useEffect, useRef } from 'react';
import { motion } from 'framer-motion';
import { FileText } from 'lucide-react';
import { usePlayer } from '@/providers/PlayerProvider';
import { useLyrics } from '@/hooks/useLyrics';
import { cn } from '@/lib/utils';

export function LyricsViewer() {
    const { state } = usePlayer();
    const scrollRef = useRef<HTMLDivElement>(null);
    const activeLineRef = useRef<HTMLDivElement>(null);

    // Включаем синхронизацию
    useLyrics();

    // === Автоскролл к активной строке ===
    useEffect(() => {
        if (!activeLineRef.current || !scrollRef.current) return;

        // Плавный скролл к центру контейнера
        activeLineRef.current.scrollIntoView({
            behavior: 'smooth',
            block: 'center',
        });
    }, [state.currentLyricIndex]);

    // === Нет текста ===
    if (!state.lyrics.length) {
        return (
            <div className="h-full flex flex-col items-center justify-center text-white/20 p-8">
                <FileText className="w-12 h-12 mb-3" />
                <p className="text-sm text-center">
                    Нет текста для этого трека
                </p>
                <p className="text-xs mt-1 text-center">
                    Нажмите правой кнопкой на трек → Редактировать → Загрузить LRC
                </p>
            </div>
        );
    }

    return (
        <div className="h-full flex flex-col">
            {/* Заголовок */}
            <div className="px-4 py-3 border-b border-white/5">
                <h3 className="text-sm font-semibold text-white/80">Текст песни</h3>
                <p className="text-xs text-white/30 truncate">
                    {state.currentTrack?.title}
                </p>
            </div>

            {/* Список строк */}
            <div
                ref={scrollRef}
                className="flex-1 overflow-y-auto px-4 py-6 space-y-1
                           scrollbar-thin scrollbar-thumb-purple-500/20"
            >
                {state.lyrics.map((line, index) => {
                    const isActive = index === state.currentLyricIndex;
                    const isPast = index < state.currentLyricIndex;

                    return (
                        <motion.div
                            key={`${index}-${line.time}`}
                            ref={isActive ? activeLineRef : null}
                            animate={{
                                scale: isActive ? 1.05 : 1,
                                opacity: isActive ? 1 : isPast ? 0.35 : 0.5,
                            }}
                            transition={{
                                duration: 0.3,
                                ease: [0.4, 0, 0.2, 1],
                            }}
                            className={cn(
                                'py-2.5 px-3 rounded-xl text-center transition-colors duration-300',
                                isActive
                                    ? 'text-purple-300 font-semibold'
                                    : 'text-white/50'
                            )}
                        >
                            {isActive && (
                                <motion.div
                                    layoutId="lyrics-glow"
                                    className="absolute inset-0 rounded-xl bg-purple-500/10"
                                    transition={{ duration: 0.3 }}
                                />
                            )}
                            <span className="relative z-10 text-sm leading-relaxed">
                                {line.text || '♪'}
                            </span>
                        </motion.div>
                    );
                })}

                {/* Отступ внизу для центрирования последних строк */}
                <div className="h-32" />
            </div>
        </div>
    );
}
```

---

## 6.5 Хук Drag & Drop — `src/hooks/useDragDrop.ts`

```typescript
// === useDragDrop.ts — Drag & Drop импорт файлов ===
// Позволяет перетаскивать папки/файлы прямо в окно приложения

import { useEffect, useState, useCallback } from 'react';

export interface DragDropState {
    isDragging: boolean;
    isDragOver: boolean;
}

export function useDragDrop(onDropFiles: (paths: string[]) => void) {
    const [isDragging, setIsDragging] = useState(false);

    useEffect(() => {
        // === Drag Enter — пользователь вошёл в окно с файлами ===
        const handleDragEnter = (e: DragEvent) => {
            e.preventDefault();
            e.stopPropagation();
            
            // Проверяем, что перетаскивают файлы (не текст)
            if (e.dataTransfer?.types.includes('Files')) {
                setIsDragging(true);
            }
        };

        // === Drag Over — файл над окном ===
        const handleDragOver = (e: DragEvent) => {
            e.preventDefault();
            e.stopPropagation();
        };

        // === Drag Leave — файл ушёл из окна ===
        const handleDragLeave = (e: DragEvent) => {
            e.preventDefault();
            e.stopPropagation();
            
            // Проверяем, что мы действительно покинули окно (а не дочерний элемент)
            if (e.relatedTarget === null) {
                setIsDragging(false);
            }
        };

        // === Drop — файл брошен ===
        const handleDrop = (e: DragEvent) => {
            e.preventDefault();
            e.stopPropagation();
            setIsDragging(false);

            if (!e.dataTransfer) return;

            const files: string[] = [];

            // В Tauri можно получить пути к файлам через DataTransfer
            // Используем webkitGetAsEntry для директорий
            const items = e.dataTransfer.items;
            
            for (let i = 0; i < items.length; i++) {
                const item = items[i];
                const entry = item.webkitGetAsEntry?.();
                
                if (entry) {
                    // Рекурсивно обходим директории
                    collectFiles(entry, files);
                }
            }

            // Также проверяем обычные файлы
            for (let i = 0; i < e.dataTransfer.files.length; i++) {
                const file = e.dataTransfer.files[i];
                // В Tauri, file.path содержит абсолютный путь (не доступен в обычном браузере)
                const path = (file as any).path as string | undefined;
                if (path && isAudioFile(path)) {
                    files.push(path);
                }
            }

            if (files.length > 0) {
                onDropFiles([...new Set(files)]); // Убираем дубликаты
            }
        };

        // Подписываемся на события
        document.addEventListener('dragenter', handleDragEnter);
        document.addEventListener('dragover', handleDragOver);
        document.addEventListener('dragleave', handleDragLeave);
        document.addEventListener('drop', handleDrop);

        return () => {
            document.removeEventListener('dragenter', handleDragEnter);
            document.removeEventListener('dragover', handleDragOver);
            document.removeEventListener('dragleave', handleDragLeave);
            document.removeEventListener('drop', handleDrop);
        };
    }, [onDropFiles]);

    return { isDragging };
}

// === Рекурсивный обход директории ===
function collectFiles(entry: any, files: string[]): void {
    if (entry.isFile) {
        entry.file((file: File) => {
            const path = (file as any).path as string | undefined;
            if (path && isAudioFile(path)) {
                files.push(path);
            }
        });
    } else if (entry.isDirectory) {
        const reader = entry.createReader();
        reader.readEntries((entries: any[]) => {
            for (const childEntry of entries) {
                collectFiles(childEntry, files);
            }
        });
    }
}

// === Проверка аудио расширения ===
function isAudioFile(path: string): boolean {
    const ext = path.split('.').pop()?.toLowerCase();
    return ['mp3', 'flac', 'wav', 'ogg', 'm4a', 'aac', 'wma', 'opus'].includes(ext || '');
}
```

---

## 6.6 Интеграция в App.tsx

Добавь в `AppContent` Drag & Drop overlay:

```tsx
// === Обновлённый AppContent ===
function AppContent() {
    const { state, actions } = usePlayer();
    const audioPlayer = useAudioPlayer();  // Запускаем аудио-движок
    
    // Drag & Drop
    const [importing, setImporting] = useState(false);
    
    const handleDropFiles = useCallback(async (paths: string[]) => {
        try {
            setImporting(true);
            const { importDroppedPaths } = await import('@/lib/importTracks');
            const result = await importDroppedPaths(paths);
            toast.success(`Импортировано ${result.imported} треков`);
            window.location.reload();
        } catch (error) {
            toast.error('Ошибка импорта');
        } finally {
            setImporting(false);
        }
    }, []);
    
    const { isDragging } = useDragDrop(handleDropFiles);

    // ... остальной код AppContent

    return (
        <div className="h-screen w-screen flex flex-col bg-aurora-bg ...">
            {/* Drag & Drop Overlay */}
            {isDragging && (
                <div className="fixed inset-0 z-[100] bg-purple-600/20 backdrop-blur-sm
                                border-4 border-dashed border-purple-400 rounded-3xl m-4
                                flex items-center justify-center pointer-events-none">
                    <div className="text-center text-white">
                        <p className="text-2xl font-bold">Отпустите файлы</p>
                        <p className="text-sm opacity-70">Для импорта в библиотеку</p>
                    </div>
                </div>
            )}
            
            {/* Основная область */}
            {/* ... */}
        </div>
    );
}
```

---

## 6.7 Подключение Seek в PlayerBar

Обнови `PlayerBar.tsx` — добавь обработчик seek:

```tsx
// В PlayerBar.tsx — обнови прогресс-бар:
import { useAudioPlayer } from '@/hooks/useAudioPlayer';

export function PlayerBar() {
    const { state, actions } = usePlayer();
    const { seek } = useAudioPlayer();  // Получаем функцию seek
    
    // ...

    // В Slider замени onValueChange:
    <Slider
        value={[state.progress * 100]}
        max={100}
        step={0.1}
        onValueChange={([val]) => seek(val)}  // ← Перемотка!
        className="flex-1"
    />
    
    // ...
}
```

---

## ✅ Итог ШАГА 6

| Компонент | Описание |
|-----------|----------|
| `lrcParser.ts` | Парсинг LRC: regex таймстампы, offset, множественные метки, бинарный поиск |
| `useAudioPlayer.ts` | Howler.js: загрузка, play/pause, seek, громкость, RAF прогресс, автопереход |
| `useLyrics.ts` | Интервал 100мс: сверка currentTime с lyrics, обновление индекса |
| `LyricsViewer.tsx` | Framer Motion анимация, scrollIntoView, подсветка активной строки |
| `useDragDrop.ts` | DragEnter/Leave/Over/Drop, рекурсивный обход директорий |

---

**Напиши "далее" — перейдём к ШАГУ 7: Сборка в .exe/.dmg и настройка иконки.**
