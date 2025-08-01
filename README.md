# Прогнозирование вегетационного периода с помощью спутниковых снимков и машинного обучения

Этот проект представляет собой end-to-end решение для определения начала и конца вегетационного периода на основе спутниковых данных Sentinel-2 и метеорологических данных ERA5-Land. Для решения задачи классификации используется ансамбль моделей машинного обучения.

## Описание проекта

Определение точных дат начала и конца вегетации имеет решающее значение для сельского хозяйства, мониторинга экосистем и изучения изменения климата. Данный проект автоматизирует этот процесс, используя общедоступные геопространственные данные и методы машинного обучения.

Модель обучается предсказывать бинарную цель: наступил ли вегетационный период (`1`) или еще нет/уже закончился (`0`). Для этого были созданы два отдельных набора данных и обучены две специализированные модели:
1.  **Модель начала периода**: определяет переход от состояния "нет вегетации" к "вегетация" весной.
2.  **Модель конца периода**: определяет переход от "вегетация" к "нет вегетации" осенью.

## Ключевые особенности

- **Автоматизированный сбор данных**: Пайплайн для загрузки данных со спутников Sentinel-2 через **Google Earth Engine** и метеоданных ERA5-Land через **Copernicus CDS API**.
- **Инженерия признаков**: Расчет ключевых вегетационных индексов (NDVI, NDWI, EVI) и использование временных признаков (день года, месяц) для повышения точности модели.
- **Ансамблирование моделей**: Использование `VotingClassifier` для объединения предсказаний трех сильных моделей (Random Forest, XGBoost, MLP), что обеспечивает более высокую и стабильную производительность.
- **Комплексная оценка**: Оценка моделей с использованием метрик Accuracy, F1-score, ROC-AUC, а также визуализация матрицы ошибок и ROC-кривой.
- **Структурированный код**: Проект разделен на логические скрипты для подготовки данных и обучения, что делает его воспроизводимым и расширяемым.

## Технологический стек

- **Анализ данных**: Python, Pandas, NumPy
- **Машинное обучение**: Scikit-learn, XGBoost
- **Геопространственные данные**: Google Earth Engine API (`earthengine-api`)
- **Метеоданные**: Copernicus Climate Data Store API (`cdsapi`), `xarray`, `cfgrib`
- **Визуализация**: Matplotlib, Seaborn

## Методология

Процесс работы над проектом можно разделить на следующие этапы:

1.  **Сбор данных**:
    - Для заданных координат и временных интервалов (весенние и осенние месяцы за несколько лет) были загружены мультиспектральные данные со спутника Sentinel-2.
    - Каждая запись была обогащена соответствующими метеорологическими данными (температура, солнечное излучение, испарение и др.) из реанализа ERA5-Land.

2.  **Разметка данных (Feature Engineering & Labeling)**:
    - По мультиспектральным каналам были рассчитаны индексы NDVI, NDWI и EVI.
    - Временные метки были преобразованы в циклические признаки (день года, месяц).
    - Целевая переменная `Target` была создана на основе пороговой даты: для модели **начала** периода все, что после 1 мая, размечено как `1` (вегетация); для модели **конца** — все, что после 12 октября, размечено как `0` (нет вегетации).
    - Для каждой точки временной ряд обрезался после первого наступления целевого события, чтобы модель училась именно на моменте перехода.

3.  **Моделирование**:
    - Данные были разделены на обучающую и тестовую выборки.
    - Был создан пайплайн предобработки, включающий заполнение пропусков (медианой) и масштабирование признаков (`StandardScaler`).
    - Были обучены три базовые модели:
        - `RandomForestClassifier`
        - `XGBClassifier`
        - `MLPClassifier` (нейронная сеть)
    - Финальная модель — это ансамбль `VotingClassifier` с мягким голосованием (`voting='soft'`), который усредняет вероятности предсказаний базовых моделей.

## Результаты

Ансамблевая модель показала высокую точность на тестовых данных для обоих периодов.

| Модель                   | ROC-AUC | F1-Score (класс 1) | Accuracy |
| ------------------------ | ------- | ------------------ | -------- |
| **Начало периода**       | **0.999** | **0.96**           | **0.98** |
| **Конец периода**        | **0.996** | **0.97**           | **0.98** |

Визуализации, такие как матрица ошибок и ROC-кривая, подтверждают высокое качество классификации и сохраняются в папку `models/` после обучения.
## Установка и запуск

1.  **Клонируйте репозиторий**:
    ```bash
    git clone https://github.com/ВАШ_НИК/Vegetation-Period-Prediction.git
    cd Vegetation-Period-Prediction
    ```

2.  **Создайте и активируйте виртуальное окружение**:
    ```bash
    python -m venv venv
    source venv/bin/activate  # Для Linux/macOS
    # или
    venv\Scripts\activate    # Для Windows
    ```

3.  **Установите зависимости**:
    ```bash
    pip install -r requirements.txt
    ```

4.  **Настройте API-доступы**:
    - **Google Earth Engine**: Пройдите аутентификацию в терминале:
      ```bash
      earthengine authenticate
      ```
    - **Copernicus CDS**: Создайте файл `.cdsapirc` в вашей домашней директории (`C:\Users\ИМЯ_ПОЛЬЗОВАТЕЛЯ` на Windows или `~` на Linux/macOS) с вашими учетными данными:
      ```
      url: https://cds.climate.copernicus.eu/api/v2
      key: ВАШ_UID:ВАШ_API_КЛЮЧ
      ```

## Порядок работы

1.  **Подготовка данных**:
    Запустите скрипт `1_data_preparation.py`. Он выполнит все шаги по сбору данных и создаст в папке `data/` файлы `final_start.csv` и `final_end.csv`.
    *Внимание: этот процесс может занять значительное время из-за большого количества запросов к API.*
    ```bash
    python 1_data_preparation.py
    ```
    *Если вы загрузили готовые CSV-файлы в папку `data/`, этот шаг можно пропустить.*

2.  **Обучение модели**:
    Запустите скрипт `2_train_model.py`, указав флаг `--period_type` со значением `start` или `end`.

    - **Обучение модели для начала периода:**
      ```bash
      python 2_train_model.py --period_type start
      ```
    - **Обучение модели для конца периода:**
      ```bash
      python 2_train_model.py --period_type end
      ```
    Результаты (файл модели `.pkl` и графики `.png`) будут сохранены в папку `models/`.
