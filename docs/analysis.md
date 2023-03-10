## Предварительный анализ задачи определения мошеннических финансовых операций



### 1. Цели проектируемой антифрод-системы в соответствии с требованиями заказчика

1. Разработать ML-модель, классифицирующую банковские транзакции на 2 класса: мошеннические и корректные, на основе данных заказчика;


2. Разработать архитектуру системы для обучения и инференса модели в облаке, обеспечив защиту данных клиентов;


3. Создать MVP системы за 3 месяца, полноценный продукт --- за 6 месяцев, потратив не более 10 млн рублей без учета выплат заработной платы специалистам;


4. Обеспечить производительность (способность выдерживать пиковую нагрузку) и корректность работы (см. п. 2)


### 2. Целевая метрика

Выделим требования заказчика к качеству:

1. Доля корректных транзакций, которые классифицированы как мошеннические (`FP`), не должна превышать 5%. Считая, что целевой класс 1 означет мошенническую
транзакцию, а 0 --- корректную, получим:

```math
FPR = \frac{FP}{FP  + TN} \le 0.05 
```


2. У ближайших конкурентов на каждую сотню транзакций фиксируется не более двух мошеннических, приводящих к потере денежных средств. 
Общий ущерб клиентов за месяц не превышает 500 тыс. руб. Из этого мы можем выделить:

```math
Accuracy = \frac{TP}{TP + FP + TN + FN} \ge 0.02
```

К потере денежных средств приводят мошеннические транзакции, которые не были обнаружены (FN). Следовательно, общий ущерб можно посчитать, как:

```math
TotalLoss = FN * AvgLoss \le 500.000
```

где `AvgLoss` --- средний размер ущерба от мошеннических транзакций. 

Хотя метрика `Accuracy` не является достаточно репрезентативной в случае дисбаланса классов (в случае анти-фрод системы он
скорее всего будет значительный). Поэтому я бы дополнительно смотрела на другие метрики: `Presision`, `Recall`, `ROC-AUC`, `F1-Score`. 

Учитывая формальные требования, можно составить агрегированную метрику:

```math
AggrQuality = w1 \cdot (0.05 - FPR) + w2 \cdot (Accuracy - 0.02) + w3 \cdot (500.000 - FN * AvgLoss) \rightarrow max
```

где  `w1, w2, w3` --- веса отдельных метрик. Поскольку все метрики одинаково значимы для бизнеса, веса `w1, w2` можно брать равными единице,
однако для `w3` необходимо подобрать вес так, чтобы значение `(500.000 - FN * AvgLoss)` лежало в одном диапазоне с `FPR` и `Accuracy`. 
Это можно сделать уже после получения реальных данных, оценив значение `AvgLoss`.


### 3 Анализ особенностей проекта на основе Machine Learning Canvas

#### 1 Prediction Task

- Задача бинарной классификации: мошенническая / корректная транзакция.
- Входные данные: данные о транзакции (табличные данные) с числовыми и категориальными признаками.
- Выходные данные: вероятность (0.0 - 1.0) того, что транзакция мошенническая.
- Таргетный класс (1): мошеннические транзакции.


#### 2 Decisions

- Выход модели (вероятность мошеннической транзакции) сравнивается с пороговым значением `threshold`:
если `probability >= threshold`, транзакция считается мошеннической. Иначе --- корректной. 
- Параметр `threshold` напрямую влияет на конечный выход системы и должен быть подобран на этапе тестирования модели таким образом,
чтобы значение целевой метрики было максимальным.

#### 3 Value Proposition

- Конечный пользователь: клиенты банка, которые хотят быть защищенными от мошеннических операций, но при этом иметь возможность
свободно пользоваться банковской системой без угрозы бана за каждую транзакцию. Внедрение системы позволит повысить сохранность их денежных средств,
что в свою очередь повлияет на их удовлетворенность системой и лояльность банку.


#### 4 Data Collection

- Существует исторический массив данных об операциях клиентов, однако неизвестно, существует ли разметка и как она проводилась. Если готовой разметки нет,
необходимо будет использовать дополнительные источники данных внутри компании, что может увеличить сроки выполнения работы и потребовать привлечения дополнительных
сотрудников и консультантов со стороны банка.

#### 5 Data Sources
- Источник данных: внутренние данные банка. Другие источники не привлекаются.


#### 6 Offline Evaluation
- Метрики: целевая метрика (п.2) + стандартные метрики бинарной классификации (`Accuracy`, `Precision`, `Recall`, `F1-Score`, `ROC-AUC`)
- Для валидирования на этапе  MVP достаточно разбить на train / val / test имеющийся набор данных. Поскольку данные носят временной характер,
разбиение необходимо проводить с учетом правильной последовательности операций для каждого пользователя. Можно провести кросс-валидацию на временных рядах для более объективной
оценки.
- На заключительном этапе было бы полезно получить свежие данные, собранные за период разработки, как итоговый тестовый датасет.


#### 7 Making predictions
- Инференс происходит при каждой транзакции
- Производительность: пиковая нагрузка 400 операций / сек., средняя 50 операций / сек. *с учетом времени предобработки данных*.


#### 8 Building models
- Основной ML-компонент --- модель-классификатор, работающая на табличных данных. Можно экспериментировать как с нейронными сетями, так и с бустингами/бэггингом.
- Дообучение модели на новых данных должно быть регулярным (предположительно, раз в 3-6 месяцев). Можно проводить внеплановый анализ качества и дообучение при падении целевых метрик качества
ниже допустимого порога (указаны в п.2)


#### 9 Features
- Система должна работать на данных заказчика с присущим ему специфичным набором полей и признаков. В следствие этого использование открытых датасетов при разработки модели не допускается,
т.к. данные будут не совместимы.
- Если в данных присутствуют техтовые признаки, то может понадобиться привлечение языковых моделей / алгоритмов типа TF-IDF для выделения эмбеддингов.

#### 10 Live monitoring
- Отслеживать целевые метрики (п.2) и проводить внеплановый анализ и дообучение при их падении ниже допустимого порога.


### 4 Декомпозиция системы

Система должна иметь две компоненты: одна для машинного обучения, которая предназначена для проведения экспериментов, запуска обучения / дообучения моделей
и вторая для работы самого сервиса.

- Репозиторий кода: Github + Actions / Gitlab + Gitlab CI.


#### Компонента обучения 

- Хранилище данных: HDFS.
- Репозиторий моделей: DVC для версионирования + MinIO для хранения весов.
- Обучение моделей: кластер с PySpark и Pytorch.
- Трекинг экспериментов: ClearML / Weights&Biases / MLFlow.

#### Компонента сервиса

- Хранение обрабатываемых логов транзакций: Elastic Search.
- Журналирование и мониторинг: Prometheus + Grafana.
- Потоковая обработка: Kafka.

### 5 Задачи проекта

#### №1
Провести анализ существующих подходов для anti-fraud detection. Подходы искать в источниках
типа GitHub, PapersWithCode, GoogleScholar, Kaggle и т.д. Выделить не менее 3х и не более 6 методов, которые являются наиболее перспективными
с точки зрения качества. Составить ранжированный список с указанием ссылок на первоисточники (статьи, репозитории, тд.), с кратким описанием методов.
Выполнить за первые 2 недели проекта.


#### №2
Провести EDA данных заказчика: посмотреть на объем; посчитать статистики, корреляции; выявить пропуски и возможные причины;
определить наличие разметки. Запросить объяснения признаков при необходимости и возможности.  Выполнить за первые 2 недели проекта.


#### №3
Предложить схему валидации моделей с учетом специфики данных заказчика и разработать конвейры для валидации качества и быстродействия. В качестве метрик
использовать метрики из п.2 (целевую и дополнительные). Выполнить со 2 по 4 недели проекта.


#### №4
Провести анализ облачных провайдеров, которые обеспечивают все необходимые функции системы и защищенность данных. Провести расчет затрат на создание требуемой
инфраструктуры, выбрать провайдер с оптимальным соотношением цена-качество. Выполнить за первые 2 недели проекта.


#### №5
Спроектировать архитектуру системы для обучения модели и развернуть ее в облаке выбранного провайдера. Перенести туда данные заказчика для обучения. Настроить роли доступа.
Выполнить за первые 4 недели проекта.


#### №6
Провести эксперименты с обучением выбранных моделей, провалидировать их. Оценить время работы, найти способы ускорения инференса, если она не удовлетворяет
требованиям по быстродействию. Составить отчет со сравнительным анализом. Выполнить за 5-8 недели проекта.

[Ссылка на GitHub](https://github.com/OlgaChaganova/anti-fraud-mlops)
