# Предсказание времени выполнения задачи по ее описанию
Это решение заняло второе место в [всероссйском чемпионате](https://hacks-ai.ru/championships/758465) от Цифрового Прорыва, трек №3 Collector.

Public score (R2): **0.269**

Private score (R2): **0.226**

![image](https://user-images.githubusercontent.com/44319901/193449286-8a85a7fb-e528-47ae-88c9-ffe3dcbce202.png)

## Задача
> На основе личных параметров тимлидов, ответственных разработчиков, описания задачи в спринте и комментариев к ней
разработайте модель, которая сможет оценить время, которое будет затрачено на выполнение задачи.

## Метрика
В качестве метрики выступает R2

![image](https://user-images.githubusercontent.com/44319901/193451032-56c4dff4-a6dc-4791-be0b-067910cd3e12.png)

SS res - сумма квадратов остаточных ошибок.

SS tot - общая сумма ошибок.

## Данные
В данной задаче нам предоставляли 3 таблички:

1) `train/test_issues.csv` - общее описание задач, время выполнения которых нам надо предсказать

2) `train/test_comments.csv` - какие комментарии были сделаны к этой задаче и кем

3) `employees.csv` - некоторые характеристики людей, которые выполняли и назанчали задачи

![image](https://user-images.githubusercontent.com/44319901/193449461-efe73285-8865-44fb-a463-e5364ebb8c66.png)


## Решение
### Чистка датасета
В данном датасете было некоторое кол-во выбросов, на которые переобучался кэтбуст. Поэтому я просто убрал все такие значения из датасета
`df_train = df_train[df_train['overall_worklogs'] < 300000]`

### Текстовые признаки
В качестве текстовых признаков у на были названия задач и комментарии к ним. Для получения embedding-ов текстов я дообучил берт на задачу регресии. Использовал мультиязычную версию этой модели `bert-base-multilingual-cased`, так как тексты встречались как на русском, так и на английском языке. В качестве текстов подавал на вход только названия задач. Пробовал подавать `<название> [SEP] <склеенные комментарии>`, но это сильно ухудшало скор на паблике (хотя и поднимало на локальной валидации), так что в итоге отказался от данного подхода.

### Базовые признаки
Помимо эмбеддингов текстов в модель еще подавались:
* `project_id`,	`assignee_id`,	`creator_id` - категориальные признаки, предоставленные жюри
* Кол-во комментариев у задачи
* Агрегациионные признаки - среднее значение таргета по `project_id`,	`assignee_id`, `creator_id`, `hour`,	`weekday`

![image](https://user-images.githubusercontent.com/44319901/193450963-e984e191-4248-41bc-b1d8-64b79ce9c2e2.png)

### Модель
В качестве модели использовал обычный `CatBoostRegressor` c потюненными параметрами

```
params = {'n_estimators' : 3000,
          'learning_rate': .03,
          'max_depth' : 6,
          'cat_features' : cat_cols,
          'embedding_features': emb_cols,
          'l2_leaf_reg' : 2,
          'use_best_model': True,
          'task_type': 'GPU',
          'random_state': 42,
         }
````

### Отбор признаков
После сбора признаков и тюнинга гиперпараметров модели отбирал признаки при помощи `catboost_model.select_feature`. В итоге оставил где-то 400 из 800 первоначальных признаков и это тоже немного улучшило показатели модели.
