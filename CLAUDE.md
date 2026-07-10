# Montage — пайплайн автомонтажа видео

## Старт (первым делом при открытии проекта)
1. Прочитай скилл **`montage-pipeline`** (`.claude/skills/montage-pipeline/`) — оркестратор всего пайплайна.
2. Покажи пользователю краткое содержание из **`README.md`** и два пути старта:
   новый стиль → скилл **`brand-intake`**; есть клип для монтажа → Stage 1 в `montage-pipeline`.
3. Веди по стадиям из `montage-pipeline`, останавливаясь на двух воротах: `REVIEW.md` и финальный рендер.
Ничего не рендерить и не пушить без явной команды пользователя.

## Что это
Автоматический монтаж «говорящей головы»: Python анализирует видео (транскрипт, дубли, лицо), Claude размечает вырезки и акценты, Remotion рендерит финал одним проходом (jump-cut, зумы на лицо, акцентные слова с эффектом печати). Стек: Python 3.11 (.venv) + Remotion 4 (Node 22) + FFmpeg + Deepgram API.

## Карта
- `pipeline/` — Python-скрипты анализа (transcribe → indexed → edl → review → faces → props → audio). Шортсы: `shorts.py`.
- `remotion/` — Remotion-проект; `src/Main169.tsx` (16:9), `src/Shorts916.tsx` (9:16 вертикаль), `src/Root.tsx` — регистрация. Пропсы генерятся пайплайном; в репо лежат `props/*.example.json` для запуска Studio из коробки.
- `remotion/public/` — сюда кладёшь `source.mp4` (прокси исходника), шрифт, `key.wav`. Все `*.wav`/прокси — в .gitignore.
- `work/<id>/` — артефакты по каждому видео (в git не попадают): words.json, delete.json, edl.json, faces.json, accents.json, REVIEW.md, publish.md.
- `docs/` — `PIPELINE.md` (анализ), `REMOTION.md` (композиции/рендер), `brandbook.md` (шаблон стиля — задай свой).
- `.claude/skills/` — `montage-pipeline` (оркестратор), `brand-intake` (онбординг), `publish-pack` (тексты для публикации), `remotion` (доменные знания Remotion).
- `out/` — финальные рендеры (в git не попадают). `.env` — `DEEPGRAM_API_KEY` (не коммитится).

## Команды
```powershell
# анализ (venv-python), подробности в docs/PIPELINE.md
.venv\Scripts\python.exe pipeline\<script>.py ...
# Remotion (из ./remotion)
npx remotion studio                      # превью — ТОЛЬКО так
npx tsc --noEmit                         # typecheck
# declick-дорожка спикера (после props.py; перезапускать после любой правки монтажа):
.venv\Scripts\python.exe pipeline\audio.py remotion\props\main169.json
# финал — только по явной команде:
npx remotion render src/index.ts Main169 ../out/main169.mp4 --props=props/main169.json --image-format=png --crf=14 --x264-preset=slow
```

## Правила
- **Превью НЕ рендерить файлом** — смотреть в Remotion Studio. Финальный рендер только по явной команде. ProRes не делать.
- **Звук спикера не обрабатывать** (нормализация/шумодав запрещены). Исключение: 6-мс дещелчок на краях склеек.
- Композиция всегда равна целевому разрешению (1920×1080 / 1080×1920) — иначе мыло.
- Правка монтажа = `work/<id>/delete.json` → `edl.py` → `props.py` → **`audio.py`** (не руками в edl.json). audio.py перезапускать после каждой правки.
- Версии-пины: TypeScript 5.9.3 (7.x ломает Remotion), zod 4.3.6, mediapipe 0.10.21. `@remotion/media`: тримы `<Video>`/`<Audio>` — в КАДРАХ.
- Remotion `id` композиции: без `_` (дефис — ок). Исходники HEVC → прокси all-intra H.264 в `remotion/public`.
- Стиль текста в кадре — по `docs/brandbook.md` (задай свой бренд через `brand-intake` → `brand.config.json`).
- PowerShell 5.1: без `&&`. Пути с пробелами/кириллицей — в кавычках.

## Детали
- Пайплайн анализа, форматы JSON, пороги пауз — `docs/PIPELINE.md`
- Композиции, зумы, акценты, качество рендера — `docs/REMOTION.md`
