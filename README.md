# 🎵 Spotify Popularity Prediction

Предсказание популярности треков Spotify на основе метаданных из **Spotify**, **MusicBrainz** и **Deezer**.

![Python](https://img.shields.io/badge/python-3.10+-blue)

---

## 📊 Результаты

Сравнение 6 моделей на тестовой выборке (1047 треков):

| Модель | R² | MAE | RMSE |
|--------|-----|-----|------|
| LinearRegression | 0.127 | 14.75 | 18.31 |
| Ridge (α=1) | 0.127 | 14.75 | 18.31 |
| Lasso (α=0.1) | 0.126 | 14.78 | 18.31 |
| SVR (RBF, C=10) | 0.173 | 14.19 | 17.82 |
| CatBoost (raw) | 0.342 | 12.25 | 15.89 |
| **CatBoost (clean)** | **0.426** | **11.35** | **14.21** |

**CV MAE (5-fold):** 11.95 ± 0.30 — модель **стабильна**.

---

## 🔍 Топ-10 драйверов популярности

| # | Признак | Важность |
|---|---------|----------|
| 1 | `deezer_artist_fans` | 29.9% |
| 2 | `album_age_years` | 8.4% |
| 3 | `duration_ms` | 8.2% |
| 4 | `deezer_gain` | 8.1% |
| 5 | `deezer_artist_albums` | 7.6% |
| 6 | `artist_begin_year` | 6.4% |
| 7 | `deezer_bpm` | 4.8% |
| 8 | `artist_career_duration_years` | 3.4% |
| 9 | `artist_end_year` | 2.1% |
| 10 | `super_genre_Hip-Hop/R&B/Soul` | 1.8% |

**Ключевой вывод:** популярность определяется в первую очередь **известностью артиста** (число фанатов в Deezer — 30%).

---

## 📁 Структура проекта

```
📦 spotify-popularity-prediction/
├── 📄 README.md
├── 📄 requirements.txt
├── 📄 .gitignore
│
├── 📂 notebooks/
│   ├── 📓 01_data_collection.ipynb       # Сбор данных
│   ├── 📓 02_eda.ipynb                   # EDA
│   ├── 📓 03_feature_engineering.ipynb   # Feature engineering
│   └── 📓 04_modeling.ipynb              # Моделирование
│
├── 📂 results/
│   ├── 📊 metrics.csv
│   └── 📂 figures/
│       ├── 🖼️ model_comparison.png
│       ├── 🖼️ feature_importance.png
│       ├── 🖼️ shap_bar.png
│       └── 🖼️ shap_summary.png
│
└── 📂 data/                              # Не коммитится
```

---

## 🚀 Воспроизведение

### 1. Установка зависимостей

```bash
pip install -r requirements.txt
```

### 2. Запуск ноутбуков
Откройте ноутбуки по порядку (в Colab или локально):
- **01_data_collection.ipynb** — собирает данные из Spotify + MusicBrainz + Deezer (~30 мин при первом запуске). Сохраняет data/01_raw_enriched.pkl.
- **02_eda.ipynb** — разведочный анализ данных. Сохраняет data/02_eda_summary.json.
- **03_feature_engineering.ipynb** — создаёт 48 признаков. Сохраняет data/03_features.pkl.
- **04_modeling.ipynb** — обучает 6 моделей. Сохраняет results/metrics.csv и графики.
  
**Важно**: каждый ноутбук загружает данные из предыдущего. Запускайте строго по порядку.

## 🔬 Методология
### Источники данных
| Источник | Что даёт | Как получено |
|----------|----------|--------------|
| Kaggle | 6300 треков Spotify | `kagglehub.dataset_download()` |
| MusicBrainz | Страна, лейбл, тип артиста, годы карьеры | `musicbrainzngs` + кэш |
| Deezer | BPM, gain, rank, fans, albums | Deezer API + кэш |

## Feature engineering (48 признаков)
Числовые:
- `duration_ms`, `explicit`;
- `album_data_completeness`, `artist_data_completeness`;
- `artist_begin_year`, `artist_end_year`;
- `artist_career_duration_years`;
- `album_age_years` (создан из `album_year`);
- `deezer_bpm`, `deezer_gain`, `deezer_artist_fans`, `deezer_artist_albums`.
  
Бинарные:
- `artist_is_active`, `career_duration_known`, `album_age_known`.
  
One-hot (35 столбцов):
- `release_country` (10 категорий);
- `release_status` (5);
- `artist_type` (6);
- `artist_country` (10);
- `super_genre` (4).

## Модели
|Тип	| Модели |
|----------|----------|
| Линейные |	LinearRegression, Ridge (L2), Lasso (L1) |
| Ядерные |	SVR с RBF-ядром (grid search по C, ε) |
| Ансамбль |	CatBoost (градиентный бустинг на деревьях) |

## Ключевые решения
1. Удаление `deezer_rank` — `data leakage` (`r = 0.67` с `popularity`, но измеряет то же самое).
2. Удаление 62 аномалий — треки с `popularity` = 0 при высокой известности артиста.
3. Регуляризация `CatBoost` — `depth=5, l2_leaf_reg=20` против переобучения.
4. Обработка unknown — оставлены отдельными категориями (информативны, `r = −0.16`).
5. Флаги для структурных пропусков — `career_duration_known`, `album_age_known`.

---

## 🎯 Возможные улучшения
- Добавить внешние признаки: Spotify (не был использован, так как нет доступа к Spotify Premium) `audio_features` (danceability, energy, valence), Google Trends, TikTok-виральность;
- Использовать target encoding для стран с малым n (вместо one-hot);
- Гиперпараметрический поиск (Optuna) — потенциально R² → 0.45;
- Robust-модели: HuberRegressor, quantile regression для устойчивости к выбросам;
- Отдельные модели для популярных/непопулярных треков.

## 📚 Источники данных
- [Kaggle: Spotify Tracks Dataset](https://www.kaggle.com/datasets/ambaliyagati/spotify-dataset-for-playing-around-with-sql)
- [MusicBrainz API](https://musicbrainz.org/doc/MusicBrainz_API)
- [Deezer API](https://developers.deezer.com/api)

## 🛠️ Технологии

| Технология | Назначение |
|------------|------------|
| **Python 3.10+** | Язык разработки |
| **Pandas, NumPy** | Обработка данных |
| **Scikit-learn** | Линейные модели, SVR, метрики |
| **CatBoost** | Градиентный бустинг |
| **SHAP** | Интерпретация модели |
| **Matplotlib, Seaborn** | Визуализация |
| **MusicBrainzngs, Requests** | Работа с API |
| **Kagglehub, gdown** | Загрузка данных |

---

## 👤 Автор
[Гареева Азалия]

GitHub: [@Azalea-12] [https://github.com/Azalea-12/spotify-popularity-prediction]

Email: gareevaazalia12@gmail.com
