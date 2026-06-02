# ШАГ 5: Фронтенд UI (React + Tailwind + Glassmorphism)

В этом шаге создаём все React-компоненты: плеер с эффектом матового стекла, список треков, модальное окно редактирования.

---

## 5.1 Глобальное состояние — `src/providers/PlayerProvider.tsx`

React Context хранит текущий трек, плейлист, состояние воспроизведения:

```tsx
// === PlayerProvider.tsx — Глобальное состояние плеера ===
// Через этот контекст все компоненты получают доступ к:
// текущему треку, списку треков, состоянию play/pause

import React, { createContext, useContext, useState, useCallback, useRef } from 'react';
import type { Track, LyricsLine } from '@/types';

// === Интерфейс состояния плеера ===
interface PlayerState {
    // Текущий трек
    currentTrack: Track | null;
    // Все треки (плейлист)
    tracks: Track[];
    // Воспроизведение
    isPlaying: boolean;
    // Прогресс (0.0 — 1.0)
    progress: number;
    // Текущее время в секундах
    currentTime: number;
    // Громкость (0.0 — 1.0)
    volume: number;
    // Текст песни (распарсенный LRC)
    lyrics: LyricsLine[];
    // Индекс текущей строки lyrics
    currentLyricIndex: number;
    // Показывать ли панель lyrics
    showLyrics: boolean;
}

// === Интерфейс методов плеера ===
interface PlayerActions {
    setCurrentTrack: (track: Track | null) => void;
    setTracks: (tracks: Track[]) => void;
    togglePlay: () => void;
    setIsPlaying: (playing: boolean) => void;
    setProgress: (progress: number) => void;
    setCurrentTime: (time: number) => void;
    setVolume: (volume: number) => void;
    nextTrack: () => void;
    prevTrack: () => void;
    setLyrics: (lyrics: LyricsLine[]) => void;
    setCurrentLyricIndex: (index: number) => void;
    toggleLyrics: () => void;
}

// === Комбинированный тип контекста ===
interface PlayerContextType {
    state: PlayerState;
    actions: PlayerActions;
    // Ref для доступа к текущему состоянию в колбэках (без ререндеров)
    stateRef: React.RefObject<PlayerState>;
}

// === Создаём контекст ===
const PlayerContext = createContext<PlayerContextType | null>(null);

// === Хук для использования контекста ===
export function usePlayer() {
    const context = useContext(PlayerContext);
    if (!context) {
        throw new Error('usePlayer must be used within PlayerProvider');
    }
    return context;
}

// === Провайдер ===
export function PlayerProvider({ children }: { children: React.ReactNode }) {
    // Основное состояние
    const [currentTrack, setCurrentTrackState] = useState<Track | null>(null);
    const [tracks, setTracks] = useState<Track[]>([]);
    const [isPlaying, setIsPlaying] = useState(false);
    const [progress, setProgress] = useState(0);
    const [currentTime, setCurrentTime] = useState(0);
    const [volume, setVolumeState] = useState(0.8);
    const [lyrics, setLyrics] = useState<LyricsLine[]>([]);
    const [currentLyricIndex, setCurrentLyricIndex] = useState(-1);
    const [showLyrics, setShowLyrics] = useState(false);

    // Ref для доступа к состоянию в колбэках без замыканий
    const stateRef = useRef<PlayerState>({
        currentTrack: null, tracks: [], isPlaying: false,
        progress: 0, currentTime: 0, volume: 0.8,
        lyrics: [], currentLyricIndex: -1, showLyrics: false,
    });

    // Обновляем ref при изменении состояния
    const updateRef = useCallback(() => {
        stateRef.current = {
            currentTrack, tracks, isPlaying, progress,
            currentTime, volume, lyrics, currentLyricIndex, showLyrics,
        };
    }, [currentTrack, tracks, isPlaying, progress, currentTime, volume, lyrics, currentLyricIndex, showLyrics]);

    // Методы с автоматическим обновлением ref
    const setCurrentTrack = useCallback((track: Track | null) => {
        setCurrentTrackState(track);
        // Сбрасываем lyrics при смене трека
        setLyrics([]);
        setCurrentLyricIndex(-1);
        setProgress(0);
        setCurrentTime(0);
    }, []);

    const togglePlay = useCallback(() => {
        setIsPlaying(prev => !prev);
    }, []);

    const setVolume = useCallback((v: number) => {
        setVolumeState(v);
    }, []);

    const nextTrack = useCallback(() => {
        if (tracks.length === 0 || !currentTrack) return;
        const currentIndex = tracks.findIndex(t => t.id === currentTrack.id);
        const nextIndex = (currentIndex + 1) % tracks.length;
        setCurrentTrack(tracks[nextIndex]);
    }, [tracks, currentTrack, setCurrentTrack]);

    const prevTrack = useCallback(() => {
        if (tracks.length === 0 || !currentTrack) return;
        const currentIndex = tracks.findIndex(t => t.id === currentTrack.id);
        const prevIndex = (currentIndex - 1 + tracks.length) % tracks.length;
        setCurrentTrack(tracks[prevIndex]);
    }, [tracks, currentTrack, setCurrentTrack]);

    const toggleLyrics = useCallback(() => {
        setShowLyrics(prev => !prev);
    }, []);

    // Собираем состояние
    const state: PlayerState = {
        currentTrack, tracks, isPlaying, progress,
        currentTime, volume, lyrics, currentLyricIndex, showLyrics,
    };

    const actions: PlayerActions = {
        setCurrentTrack, setTracks, togglePlay, setIsPlaying,
        setProgress, setCurrentTime, setVolume, nextTrack, prevTrack,
        setLyrics, setCurrentLyricIndex, toggleLyrics,
    };

    updateRef();

    return (
        <PlayerContext.Provider value={{ state, actions, stateRef }}>
            {children}
        </PlayerContext.Provider>
    );
}
```

---

## 5.2 Обёртка Glassmorphism — `src/components/layout/GlassPanel.tsx`

```tsx
// === GlassPanel.tsx — Компонент с эффектом матового стекла ===
// Использует backdrop-blur + полупрозрачный фон + тонкую границу

import React from 'react';
import { cn } from '@/lib/utils';

interface GlassPanelProps {
    children: React.ReactNode;
    className?: string;
    intensity?: 'light' | 'medium' | 'strong';
}

export function GlassPanel({ children, className, intensity = 'medium' }: GlassPanelProps) {
    const intensityClasses = {
        // Лёгкий эффект — для карточек
        light: 'backdrop-blur-md bg-white/5 border-white/5',
        // Средний — для панелей
        medium: 'backdrop-blur-xl bg-black/40 border-white/10',
        // Сильный — для модальных окон
        strong: 'backdrop-blur-2xl bg-black/70 border-white/15',
    };

    return (
        <div
            className={cn(
                'rounded-2xl shadow-2xl',
                intensityClasses[intensity],
                className
            )}
            style={{
                // Дополнительная тень для глубины
                boxShadow: `
                    0 8px 32px rgba(0, 0, 0, 0.3),
                    inset 0 1px 0 rgba(255, 255, 255, 0.05)
                `,
            }}
        >
            {children}
        </div>
    );
}
```

---

## 5.3 Боковая панель — `src/components/layout/Sidebar.tsx`

```tsx
// === Sidebar.tsx — Боковая панель навигации ===
// Минималистичная, с glassmorphism эффектом

import React from 'react';
import { Music, FolderPlus, Settings, ListMusic } from 'lucide-react';
import { cn } from '@/lib/utils';

interface SidebarProps {
    onImportClick: () => void;
    currentView: 'tracks' | 'settings';
    onViewChange: (view: 'tracks' | 'settings') => void;
}

export function Sidebar({ onImportClick, currentView, onViewChange }: SidebarProps) {
    return (
        <aside className="w-64 h-full flex flex-col p-4 gap-2">
            {/* Логотип */}
            <div className="flex items-center gap-3 px-3 py-4 mb-4">
                <div className="w-10 h-10 rounded-xl bg-gradient-to-br from-purple-600 to-blue-600 
                                flex items-center justify-center shadow-lg shadow-purple-500/20">
                    <Music className="w-5 h-5 text-white" />
                </div>
                <div>
                    <h1 className="text-lg font-bold text-white tracking-tight">Aurora</h1>
                    <p className="text-xs text-white/40">Music Player</p>
                </div>
            </div>

            {/* Навигация */}
            <nav className="flex flex-col gap-1 flex-1">
                <NavButton
                    icon={<ListMusic className="w-5 h-5" />}
                    label="Библиотека"
                    active={currentView === 'tracks'}
                    onClick={() => onViewChange('tracks')}
                />
                <NavButton
                    icon={<FolderPlus className="w-5 h-5" />}
                    label="Импорт папки"
                    active={false}
                    onClick={onImportClick}
                />
                <NavButton
                    icon={<Settings className="w-5 h-5" />}
                    label="Настройки"
                    active={currentView === 'settings'}
                    onClick={() => onViewChange('settings')}
                />
            </nav>

            {/* Мини-плеер (когда трек играет) */}
            <MiniPlayerPreview />
        </aside>
    );
}

// === Кнопка навигации ===
function NavButton({
    icon, label, active, onClick
}: {
    icon: React.ReactNode;
    label: string;
    active: boolean;
    onClick: () => void;
}) {
    return (
        <button
            onClick={onClick}
            className={cn(
                'flex items-center gap-3 px-3 py-2.5 rounded-xl text-sm font-medium',
                'transition-all duration-200 ease-out',
                'hover:bg-white/10 hover:text-white',
                active
                    ? 'bg-white/10 text-white'
                    : 'text-white/50'
            )}
        >
            {icon}
            {label}
        </button>
    );
}

// === Мини-превью трека внизу сайдбара ===
function MiniPlayerPreview() {
    // Здесь можно показать обложку текущего трека
    return (
        <div className="mt-auto pt-4 border-t border-white/5">
            <div className="text-xs text-white/30 text-center">
                Aurora v1.0.0
            </div>
        </div>
    );
}
```

---

## 5.4 Элемент трека — `src/components/tracklist/TrackItem.tsx`

```tsx
// === TrackItem.tsx — Одна строка трека в списке ===
// Показывает обложку, название, исполнителя, длительность
// При клике — запускает трек, при правом клике — контекстное меню

import React, { useState } from 'react';
import { Play, Pause, Clock, MoreHorizontal, Pencil, Trash2 } from 'lucide-react';
import type { Track } from '@/types';
import { cn } from '@/lib/utils';
import { usePlayer } from '@/providers/PlayerProvider';

interface TrackItemProps {
    track: Track;
    index: number;
    isPlaying: boolean;
    isCurrent: boolean;
    onEdit: (track: Track) => void;
    onDelete: (id: number) => void;
}

export function TrackItem({ track, index, isCurrent, onEdit, onDelete }: TrackItemProps) {
    const { actions } = usePlayer();
    const [hovered, setHovered] = useState(false);
    const [menuOpen, setMenuOpen] = useState(false);

    const handleClick = () => {
        actions.setCurrentTrack(track);
        actions.setIsPlaying(true);
    };

    return (
        <div
            className={cn(
                'group relative flex items-center gap-4 px-4 py-3 rounded-xl',
                'transition-all duration-200 cursor-pointer',
                'hover:bg-white/5',
                isCurrent && 'bg-white/10'
            )}
            onClick={handleClick}
            onMouseEnter={() => setHovered(true)}
            onMouseLeave={() => { setHovered(false); setMenuOpen(false); }}
        >
            {/* Номер / Иконка Play */}
            <div className="w-8 text-center">
                {hovered || isCurrent ? (
                    <Play className={cn(
                        'w-4 h-4 mx-auto',
                        isCurrent ? 'text-purple-400' : 'text-white/60'
                    )} />
                ) : (
                    <span className="text-sm text-white/30">{index + 1}</span>
                )}
            </div>

            {/* Обложка */}
            <div className="w-12 h-12 rounded-lg bg-gradient-to-br from-purple-900/50 to-blue-900/50 
                            flex items-center justify-center overflow-hidden flex-shrink-0">
                {track.coverPath ? (
                    <img
                        src={track.coverPath}
                        alt={track.title}
                        className="w-full h-full object-cover"
                    />
                ) : (
                    <span className="text-lg font-bold text-white/20">
                        {track.title.charAt(0).toUpperCase()}
                    </span>
                )}
            </div>

            {/* Информация о треке */}
            <div className="flex-1 min-w-0">
                <div className={cn(
                    'text-sm font-medium truncate',
                    isCurrent ? 'text-purple-300' : 'text-white/90'
                )}>
                    {track.title}
                </div>
                <div className="text-xs text-white/40 truncate">
                    {track.artist}
                    {track.album && ` • ${track.album}`}
                </div>
            </div>

            {/* Формат (FLAC/MP3) */}
            <div className="hidden sm:block">
                <span className={cn(
                    'text-xs px-2 py-0.5 rounded-full',
                    track.isLossless
                        ? 'bg-green-500/10 text-green-400'
                        : 'bg-white/5 text-white/30'
                )}>
                    {track.format.toUpperCase()}
                </span>
            </div>

            {/* Длительность */}
            <div className="flex items-center gap-1 text-xs text-white/30 w-14 justify-end">
                <Clock className="w-3 h-3" />
                {track.durationFormatted}
            </div>

            {/* Кнопка меню (только при наведении) */}
            <div className={cn(
                'relative',
                hovered ? 'opacity-100' : 'opacity-0',
                'transition-opacity duration-200'
            )}>
                <button
                    onClick={(e) => { e.stopPropagation(); setMenuOpen(!menuOpen); }}
                    className="p-1.5 rounded-lg hover:bg-white/10 text-white/40 hover:text-white"
                >
                    <MoreHorizontal className="w-4 h-4" />
                </button>

                {/* Выпадающее меню */}
                {menuOpen && (
                    <div className="absolute right-0 top-8 z-50 w-40 py-1 rounded-xl 
                                    bg-gray-900 border border-white/10 shadow-2xl">
                        <MenuItem
                            icon={<Pencil className="w-4 h-4" />}
                            label="Редактировать"
                            onClick={() => { onEdit(track); setMenuOpen(false); }}
                        />
                        <MenuItem
                            icon={<Trash2 className="w-4 h-4" />}
                            label="Удалить"
                            danger
                            onClick={() => { onDelete(track.id); setMenuOpen(false); }}
                        />
                    </div>
                )}
            </div>
        </div>
    );
}

function MenuItem({ icon, label, danger, onClick }: {
    icon: React.ReactNode;
    label: string;
    danger?: boolean;
    onClick: () => void;
}) {
    return (
        <button
            onClick={onClick}
            className={cn(
                'w-full flex items-center gap-2 px-3 py-2 text-sm',
                'hover:bg-white/5 transition-colors',
                danger ? 'text-red-400 hover:text-red-300' : 'text-white/70'
            )}
        >
            {icon}
            {label}
        </button>
    );
}
```

---

## 5.5 Список треков — `src/components/tracklist/TrackList.tsx`

```tsx
// === TrackList.tsx — Основной список всех треков ===
// С заголовком, статистикой и виртуальным скроллом

import React, { useState, useEffect } from 'react';
import { Search, Music, Loader2 } from 'lucide-react';
import { TrackItem } from './TrackItem';
import { usePlayer } from '@/providers/PlayerProvider';
import { getAllTracks, searchTracks, deleteTrack } from '@/lib/db';
import type { Track } from '@/types';
import { ScrollArea } from '@/components/ui/scroll-area';
import { EditTrackModal } from '@/components/edit/EditTrackModal';

export function TrackList() {
    const { state, actions } = usePlayer();
    const [tracks, setTracks] = useState<Track[]>([]);
    const [loading, setLoading] = useState(true);
    const [searchQuery, setSearchQuery] = useState('');
    const [editTrack, setEditTrack] = useState<Track | null>(null);

    // Загрузка треков из БД
    const loadTracks = async () => {
        try {
            setLoading(true);
            const data = searchQuery
                ? await searchTracks(searchQuery)
                : await getAllTracks();
            setTracks(data);
            actions.setTracks(data);
        } catch (error) {
            console.error('Ошибка загрузки треков:', error);
        } finally {
            setLoading(false);
        }
    };

    useEffect(() => {
        loadTracks();
    }, [searchQuery]);

    const handleDelete = async (id: number) => {
        if (!confirm('Удалить трек из библиотеки? Файл останется на диске.')) return;
        await deleteTrack(id);
        loadTracks();
    };

    const handleEditSave = () => {
        setEditTrack(null);
        loadTracks();
    };

    // Общая длительность всех треков
    const totalDuration = tracks.reduce((sum, t) => sum + t.durationSeconds, 0);
    const totalMinutes = Math.floor(totalDuration / 60);

    return (
        <div className="h-full flex flex-col">
            {/* Заголовок */}
            <div className="flex items-center justify-between px-6 py-4">
                <div>
                    <h2 className="text-xl font-bold text-white">Библиотека</h2>
                    <p className="text-sm text-white/40 mt-0.5">
                        {tracks.length} треков • {totalMinutes} минут
                        {tracks.some(t => t.isLossless) && (
                            <span className="ml-2 text-green-400">
                                ({tracks.filter(t => t.isLossless).length} Lossless)
                            </span>
                        )}
                    </p>
                </div>

                {/* Поиск */}
                <div className="relative w-64">
                    <Search className="absolute left-3 top-1/2 -translate-y-1/2 w-4 h-4 text-white/30" />
                    <input
                        type="text"
                        placeholder="Поиск треков..."
                        value={searchQuery}
                        onChange={(e) => setSearchQuery(e.target.value)}
                        className="w-full pl-10 pr-4 py-2 rounded-xl bg-white/5 border border-white/10
                                   text-sm text-white placeholder:text-white/30
                                   focus:outline-none focus:border-purple-500/50 focus:bg-white/10
                                   transition-all"
                    />
                </div>
            </div>

            {/* Список треков */}
            {loading ? (
                <div className="flex-1 flex items-center justify-center">
                    <Loader2 className="w-8 h-8 text-purple-400 animate-spin" />
                </div>
            ) : tracks.length === 0 ? (
                <div className="flex-1 flex flex-col items-center justify-center text-white/30">
                    <Music className="w-16 h-16 mb-4" />
                    <p className="text-lg">Библиотека пуста</p>
                    <p className="text-sm mt-1">Импортируйте папку с музыкой</p>
                </div>
            ) : (
                <ScrollArea className="flex-1 px-2">
                    <div className="pb-4">
                        {tracks.map((track, index) => (
                            <TrackItem
                                key={track.id}
                                track={track}
                                index={index}
                                isPlaying={state.isPlaying}
                                isCurrent={state.currentTrack?.id === track.id}
                                onEdit={setEditTrack}
                                onDelete={handleDelete}
                            />
                        ))}
                    </div>
                </ScrollArea>
            )}

            {/* Модалка редактирования */}
            {editTrack && (
                <EditTrackModal
                    track={editTrack}
                    onClose={() => setEditTrack(null)}
                    onSave={handleEditSave}
                />
            )}
        </div>
    );
}
```

---

## 5.6 Панель плеера — `src/components/player/PlayerBar.tsx`

```tsx
// === PlayerBar.tsx — Нижняя панель управления плеером ===
// Glassmorphism эффект: backdrop-blur + полупрозрачный фон
// Содержит: обложку, информацию, кнопки управления, прогресс-бар, громкость

import React from 'react';
import {
    Play, Pause, SkipBack, SkipForward, Volume2, VolumeX,
    ListMusic, Heart
} from 'lucide-react';
import { usePlayer } from '@/providers/PlayerProvider';
import { Slider } from '@/components/ui/slider';
import { cn } from '@/lib/utils';

export function PlayerBar() {
    const { state, actions } = usePlayer();
    const track = state.currentTrack;

    if (!track) {
        // Минимальная панель когда нет трека
        return (
            <div className="h-20 border-t border-white/5 bg-black/40 backdrop-blur-xl
                            flex items-center justify-center text-white/20 text-sm">
                Выберите трек для воспроизведения
            </div>
        );
    }

    return (
        <div className="h-24 px-6 flex items-center gap-6 
                        border-t border-white/10 
                        bg-black/50 backdrop-blur-2xl">
            
            {/* === ЛЕВАЯ ЧАСТЬ: Обложка + Информация === */}
            <div className="flex items-center gap-3 w-64">
                {/* Обложка */}
                <div className="w-14 h-14 rounded-xl bg-gradient-to-br from-purple-900 to-blue-900 
                                flex items-center justify-center overflow-hidden flex-shrink-0
                                shadow-lg shadow-purple-500/10">
                    {track.coverPath ? (
                        <img src={track.coverPath} alt="" className="w-full h-full object-cover" />
                    ) : (
                        <span className="text-xl font-bold text-white/30">
                            {track.title.charAt(0).toUpperCase()}
                        </span>
                    )}
                </div>

                {/* Название + Исполнитель */}
                <div className="min-w-0 flex-1">
                    <div className="text-sm font-medium text-white truncate">
                        {track.title}
                    </div>
                    <div className="text-xs text-white/40 truncate">
                        {track.artist}
                    </div>
                    {track.isLossless && (
                        <span className="text-[10px] text-green-400 bg-green-400/10 px-1.5 py-0.5 rounded">
                            Lossless
                        </span>
                    )}
                </div>
            </div>

            {/* === ЦЕНТР: Кнопки управления + Прогресс === */}
            <div className="flex-1 flex flex-col items-center gap-2 max-w-xl">
                {/* Кнопки */}
                <div className="flex items-center gap-4">
                    <button
                        onClick={actions.prevTrack}
                        className="p-2 rounded-full text-white/50 hover:text-white hover:bg-white/10 
                                   transition-all"
                    >
                        <SkipBack className="w-5 h-5" />
                    </button>

                    <button
                        onClick={actions.togglePlay}
                        className="p-3 rounded-full bg-purple-600 hover:bg-purple-500 
                                   text-white shadow-lg shadow-purple-500/25
                                   transition-all hover:scale-105 active:scale-95"
                    >
                        {state.isPlaying ? (
                            <Pause className="w-6 h-6" />
                        ) : (
                            <Play className="w-6 h-6 ml-0.5" />
                        )}
                    </button>

                    <button
                        onClick={actions.nextTrack}
                        className="p-2 rounded-full text-white/50 hover:text-white hover:bg-white/10 
                                   transition-all"
                    >
                        <SkipForward className="w-5 h-5" />
                    </button>
                </div>

                {/* Прогресс-бар + время */}
                <div className="w-full flex items-center gap-3">
                    <span className="text-xs text-white/40 w-10 text-right tabular-nums">
                        {formatTime(state.currentTime)}
                    </span>

                    <Slider
                        value={[state.progress * 100]}
                        max={100}
                        step={0.1}
                        onValueChange={([val]) => {
                            // Seek будет реализован в useAudioPlayer
                        }}
                        className="flex-1"
                    />

                    <span className="text-xs text-white/40 w-10 tabular-nums">
                        {track.durationFormatted}
                    </span>
                </div>
            </div>

            {/* === ПРАВАЯ ЧАСТЬ: Громкость + Lyrics === */}
            <div className="flex items-center gap-3 w-48 justify-end">
                {/* Кнопка Lyrics */}
                <button
                    onClick={actions.toggleLyrics}
                    className={cn(
                        'p-2 rounded-lg transition-all',
                        state.showLyrics
                            ? 'text-purple-400 bg-purple-400/10'
                            : 'text-white/40 hover:text-white hover:bg-white/10'
                    )}
                    title="Текст песни"
                >
                    <ListMusic className="w-5 h-5" />
                </button>

                {/* Громкость */}
                <div className="flex items-center gap-2">
                    <button
                        onClick={() => actions.setVolume(state.volume === 0 ? 0.8 : 0)}
                        className="text-white/40 hover:text-white transition-colors"
                    >
                        {state.volume === 0 ? (
                            <VolumeX className="w-5 h-5" />
                        ) : (
                            <Volume2 className="w-5 h-5" />
                        )}
                    </button>
                    <Slider
                        value={[state.volume * 100]}
                        max={100}
                        step={1}
                        onValueChange={([val]) => actions.setVolume(val / 100)}
                        className="w-24"
                    />
                </div>
            </div>
        </div>
    );
}

// === Форматирование времени (секунды → "m:ss") ===
function formatTime(seconds: number): string {
    if (!seconds || seconds < 0) return '0:00';
    const m = Math.floor(seconds / 60);
    const s = Math.floor(seconds % 60);
    return `${m}:${s.toString().padStart(2, '0')}`;
}
```

---

## 5.7 Модальное окно редактирования — `src/components/edit/EditTrackModal.tsx`

```tsx
// === EditTrackModal.tsx — Редактирование метаданных трека ===
// Позволяет: изменить название, исполнителя, загрузить обложку, добавить LRC

import React, { useState, useRef } from 'react';
import { X, ImagePlus, FileText, Save } from 'lucide-react';
import { motion, AnimatePresence } from 'framer-motion';
import type { Track } from '@/types';
import { updateTrack } from '@/lib/db';
import { GlassPanel } from '@/components/layout/GlassPanel';
import { cn } from '@/lib/utils';

interface EditTrackModalProps {
    track: Track;
    onClose: () => void;
    onSave: () => void;
}

export function EditTrackModal({ track, onClose, onSave }: EditTrackModalProps) {
    const [title, setTitle] = useState(track.title);
    const [artist, setArtist] = useState(track.artist);
    const [album, setAlbum] = useState(track.album);
    const [lyricsLrc, setLyricsLrc] = useState(track.lyricsLrc || '');
    const [coverPreview, setCoverPreview] = useState<string | null>(track.coverPath);
    const [saving, setSaving] = useState(false);
    
    const fileInputRef = useRef<HTMLInputElement>(null);
    const coverInputRef = useRef<HTMLInputElement>(null);

    // === Сохранение изменений ===
    const handleSave = async () => {
        try {
            setSaving(true);
            await updateTrack(track.id, {
                title: title.trim() || track.title,
                artist: artist.trim() || 'Unknown Artist',
                album: album.trim(),
                lyricsLrc: lyricsLrc.trim() || null,
                coverPath: coverPreview,
            });
            onSave();
        } catch (error) {
            console.error('Ошибка сохранения:', error);
            alert('Не удалось сохранить изменения');
        } finally {
            setSaving(false);
        }
    };

    // === Загрузка обложки ===
    const handleCoverSelect = (e: React.ChangeEvent<HTMLInputElement>) => {
        const file = e.target.files?.[0];
        if (!file) return;

        // Читаем файл как Data URL (баз64)
        const reader = new FileReader();
        reader.onload = (event) => {
            const result = event.target?.result as string;
            setCoverPreview(result);
            // В реальном приложении здесь нужно сохранить файл через Tauri FS
            // и записать путь в БД. Для простоты храним base64.
        };
        reader.readAsDataURL(file);
    };

    // === Загрузка LRC файла ===
    const handleLrcSelect = (e: React.ChangeEvent<HTMLInputElement>) => {
        const file = e.target.files?.[0];
        if (!file) return;

        const reader = new FileReader();
        reader.onload = (event) => {
            const text = event.target?.result as string;
            setLyricsLrc(text);
        };
        reader.readAsText(file);
    };

    return (
        <AnimatePresence>
            <motion.div
                initial={{ opacity: 0 }}
                animate={{ opacity: 1 }}
                exit={{ opacity: 0 }}
                className="fixed inset-0 z-50 flex items-center justify-center p-4
                           bg-black/60 backdrop-blur-sm"
                onClick={onClose}
            >
                <motion.div
                    initial={{ scale: 0.9, opacity: 0, y: 20 }}
                    animate={{ scale: 1, opacity: 1, y: 0 }}
                    exit={{ scale: 0.9, opacity: 0, y: 20 }}
                    transition={{ type: 'spring', damping: 25, stiffness: 300 }}
                    onClick={(e) => e.stopPropagation()}
                >
                    <GlassPanel intensity="strong" className="w-[520px] max-h-[80vh] flex flex-col">
                        {/* Заголовок */}
                        <div className="flex items-center justify-between px-6 py-4 border-b border-white/10">
                            <h3 className="text-lg font-semibold text-white">
                                Редактировать трек
                            </h3>
                            <button
                                onClick={onClose}
                                className="p-1.5 rounded-lg hover:bg-white/10 text-white/40 hover:text-white 
                                           transition-colors"
                            >
                                <X className="w-5 h-5" />
                            </button>
                        </div>

                        {/* Содержимое */}
                        <div className="flex-1 overflow-y-auto px-6 py-5 space-y-5">
                            
                            {/* === Обложка === */}
                            <div className="flex justify-center">
                                <div
                                    className="relative w-32 h-32 rounded-2xl overflow-hidden cursor-pointer
                                               bg-gradient-to-br from-purple-900/50 to-blue-900/50
                                               border-2 border-dashed border-white/20 hover:border-purple-400/50
                                               transition-all group"
                                    onClick={() => coverInputRef.current?.click()}
                                >
                                    {coverPreview ? (
                                        <img
                                            src={coverPreview}
                                            alt="Cover"
                                            className="w-full h-full object-cover"
                                        />
                                    ) : (
                                        <div className="absolute inset-0 flex flex-col items-center justify-center">
                                            <ImagePlus className="w-8 h-8 text-white/30 group-hover:text-purple-400 transition-colors" />
                                            <span className="text-xs text-white/30 mt-1">Обложка</span>
                                        </div>
                                    )}
                                    {/* Hover overlay */}
                                    <div className="absolute inset-0 bg-black/50 opacity-0 group-hover:opacity-100
                                                    flex items-center justify-center transition-opacity">
                                        <span className="text-xs text-white">Изменить</span>
                                    </div>
                                </div>
                                <input
                                    ref={coverInputRef}
                                    type="file"
                                    accept="image/*"
                                    className="hidden"
                                    onChange={handleCoverSelect}
                                />
                            </div>

                            {/* === Поля ввода === */}
                            <div className="space-y-4">
                                <div>
                                    <label className="block text-sm text-white/60 mb-1.5">
                                        Название трека
                                    </label>
                                    <input
                                        type="text"
                                        value={title}
                                        onChange={(e) => setTitle(e.target.value)}
                                        className="w-full px-4 py-2.5 rounded-xl bg-white/5 border border-white/10
                                                   text-white text-sm placeholder:text-white/20
                                                   focus:outline-none focus:border-purple-500/50 focus:bg-white/10
                                                   transition-all"
                                    />
                                </div>

                                <div>
                                    <label className="block text-sm text-white/60 mb-1.5">
                                        Исполнитель
                                    </label>
                                    <input
                                        type="text"
                                        value={artist}
                                        onChange={(e) => setArtist(e.target.value)}
                                        className="w-full px-4 py-2.5 rounded-xl bg-white/5 border border-white/10
                                                   text-white text-sm placeholder:text-white/20
                                                   focus:outline-none focus:border-purple-500/50 focus:bg-white/10
                                                   transition-all"
                                    />
                                </div>

                                <div>
                                    <label className="block text-sm text-white/60 mb-1.5">
                                        Альбом
                                    </label>
                                    <input
                                        type="text"
                                        value={album}
                                        onChange={(e) => setAlbum(e.target.value)}
                                        className="w-full px-4 py-2.5 rounded-xl bg-white/5 border border-white/10
                                                   text-white text-sm placeholder:text-white/20
                                                   focus:outline-none focus:border-purple-500/50 focus:bg-white/10
                                                   transition-all"
                                    />
                                </div>
                            </div>

                            {/* === LRC Текст === */}
                            <div>
                                <div className="flex items-center justify-between mb-1.5">
                                    <label className="text-sm text-white/60">
                                        Текст песни (LRC формат)
                                    </label>
                                    <button
                                        onClick={() => fileInputRef.current?.click()}
                                        className="flex items-center gap-1.5 text-xs text-purple-400 
                                                   hover:text-purple-300 transition-colors"
                                    >
                                        <FileText className="w-3.5 h-3.5" />
                                        Загрузить .lrc файл
                                    </button>
                                </div>
                                <textarea
                                    value={lyricsLrc}
                                    onChange={(e) => setLyricsLrc(e.target.value)}
                                    placeholder="[00:00.00] Первая строка&#10;[00:05.00] Вторая строка..."
                                    rows={6}
                                    className="w-full px-4 py-3 rounded-xl bg-white/5 border border-white/10
                                               text-white text-sm placeholder:text-white/20 font-mono
                                               focus:outline-none focus:border-purple-500/50 focus:bg-white/10
                                               transition-all resize-none"
                                />
                                <input
                                    ref={fileInputRef}
                                    type="file"
                                    accept=".lrc,.txt"
                                    className="hidden"
                                    onChange={handleLrcSelect}
                                />
                            </div>
                        </div>

                        {/* Кнопки */}
                        <div className="flex items-center justify-end gap-3 px-6 py-4 border-t border-white/10">
                            <button
                                onClick={onClose}
                                className="px-4 py-2 rounded-xl text-sm text-white/60 hover:text-white 
                                           hover:bg-white/5 transition-all"
                            >
                                Отмена
                            </button>
                            <button
                                onClick={handleSave}
                                disabled={saving}
                                className="flex items-center gap-2 px-6 py-2 rounded-xl
                                           bg-purple-600 hover:bg-purple-500 text-white text-sm font-medium
                                           transition-all hover:scale-[1.02] active:scale-[0.98]
                                           disabled:opacity-50 disabled:cursor-not-allowed"
                            >
                                {saving ? (
                                    <span>Сохранение...</span>
                                ) : (
                                    <>
                                        <Save className="w-4 h-4" />
                                        Сохранить
                                    </>
                                )}
                            </button>
                        </div>
                    </GlassPanel>
                </motion.div>
            </motion.div>
        </AnimatePresence>
    );
}
```

---

## 5.8 Главный компонент App — `src/App.tsx`

```tsx
// === App.tsx — Корневой компонент ===
// Объединяет Sidebar, TrackList, PlayerBar, LyricsViewer

import React, { useState, useCallback } from 'react';
import { PlayerProvider, usePlayer } from '@/providers/PlayerProvider';
import { Sidebar } from '@/components/layout/Sidebar';
import { TrackList } from '@/components/tracklist/TrackList';
import { PlayerBar } from '@/components/player/PlayerBar';
import { LyricsViewer } from '@/components/lyrics/LyricsViewer';
import { importFolder } from '@/lib/importTracks';
import { Toaster } from '@/components/ui/sonner';
import { toast } from 'sonner';

function AppContent() {
    const [currentView, setCurrentView] = useState<'tracks' | 'settings'>('tracks');
    const { state } = usePlayer();

    // === Импорт папки ===
    const handleImport = useCallback(async () => {
        try {
            toast.info('Сканирование папки...');
            const result = await importFolder();
            
            if (result.imported > 0) {
                toast.success(`Импортировано ${result.imported} треков`);
            }
            if (result.skipped > 0) {
                toast.info(`${result.skipped} треков уже в библиотеке`);
            }
            if (result.errors.length > 0) {
                toast.error(`${result.errors.length} ошибок`);
                console.error('Ошибки импорта:', result.errors);
            }
            
            // Перезагружаем страницу чтобы отобразить новые треки
            window.location.reload();
        } catch (error) {
            toast.error('Ошибка импорта');
            console.error(error);
        }
    }, []);

    return (
        <div className="h-screen w-screen flex flex-col bg-aurora-bg text-aurora-text overflow-hidden">
            {/* Основная область */}
            <div className="flex-1 flex min-h-0">
                {/* Сайдбар */}
                <Sidebar
                    onImportClick={handleImport}
                    currentView={currentView}
                    onViewChange={setCurrentView}
                />

                {/* Контент */}
                <main className="flex-1 min-h-0 flex">
                    {/* Список треков */}
                    <div className={state.showLyrics ? 'flex-1' : 'flex-1'}>
                        {currentView === 'tracks' && <TrackList />}
                        {currentView === 'settings' && <SettingsView />}
                    </div>

                    {/* Панель Lyrics (справа, появляется по кнопке) */}
                    {state.showLyrics && state.currentTrack && (
                        <div className="w-96 border-l border-white/10">
                            <LyricsViewer />
                        </div>
                    )}
                </main>
            </div>

            {/* Нижняя панель плеера */}
            <PlayerBar />

            {/* Уведомления */}
            <Toaster 
                position="bottom-right"
                toastOptions={{
                    style: {
                        background: 'rgba(18, 18, 26, 0.95)',
                        border: '1px solid rgba(255,255,255,0.1)',
                        color: '#e2e8f0',
                    },
                }}
            />
        </div>
    );
}

// === Заглушка для настроек ===
function SettingsView() {
    return (
        <div className="h-full flex items-center justify-center text-white/30">
            <div className="text-center">
                <h2 className="text-xl font-semibold mb-2">Настройки</h2>
                <p>Скоро появятся...</p>
            </div>
        </div>
    );
}

// === Корневой компонент с Provider ===
export default function App() {
    return (
        <PlayerProvider>
            <AppContent />
        </PlayerProvider>
    );
}
```

---

## 5.9 Обновлённый `src/main.tsx`

```tsx
// === main.tsx — Точка входа React ===
import React from 'react';
import ReactDOM from 'react-dom/client';
import App from './App';
import './index.css';

ReactDOM.createRoot(document.getElementById('root')!).render(
    <React.StrictMode>
        <App />
    </React.StrictMode>
);
```

---

## 5.10 Утилита `cn()` — `src/lib/utils.ts`

Убедись что файл существует (создаётся shadcn/ui):

```typescript
// === utils.ts — Вспомогательные функции ===
import { type ClassValue, clsx } from 'clsx';
import { twMerge } from 'tailwind-merge';

/**
 * Объединяет Tailwind классы с умным мержем
 * Пример: cn('px-2', isActive && 'bg-purple-600', className)
 */
export function cn(...inputs: ClassValue[]) {
    return twMerge(clsx(inputs));
}
```

---

## ✅ Итог ШАГА 5

| Компонент | Описание |
|-----------|----------|
| `PlayerProvider.tsx` | React Context: трек, плейлист, play/pause, lyrics |
| `GlassPanel.tsx` | Обертка с 3 уровнями glassmorphism |
| `Sidebar.tsx` | Боковая панель: логотип, навигация, импорт |
| `TrackItem.tsx` | Строка трека с hover-эффектами, контекстным меню |
| `TrackList.tsx` | Список с поиском, загрузкой, статистикой |
| `PlayerBar.tsx` | Панель плеера: обложка, кнопки, прогресс, громкость |
| `EditTrackModal.tsx` | Модалка: название, исполнитель, обложка, LRC |
| `App.tsx` | Сборка всего layout: Sidebar + Content + Lyrics + Player |

---

**Напиши "далее" — перейдём к ШАГУ 6: Логика Аудио (useAudioPlayer на Howler.js) и LyricsViewer с синхронизацией.**
