Диспиплина: «Цифровая трансформация высокотехнологичных процессов»
---------------------

Недобежкин Никита, группа J4151

Отчет по практической работе: Цифровые трансформации в биотехнологии

Задание №1: «Stardist_training»
-------------

1. Конфигурация сети (без использования GPU):
```
n_rays = 32,
grid = (2,2),
train_learning_rate = 0.0003,
use_gpu = False
```

2. Конфигурия обучения:
```
epochs = 100,
steps_per_epoch = 400
```

3. Критерии оценивания:

Модель сохранилась по пути: //content/drive/MyDrive/DTC part 6/stardist_model_one

![image](https://github.com/MrShaller/MrShaller/assets/62774239/9790d19b-2fff-41ff-b4dd-b2e6e15e1602)


Задание №2: «Stardist_usage»
------------------------------------------
Критерии оценивания:

1. Сравнение результатов сегментации с ручной разметкой:

Исходное изображение, ручная разметка и результат сегментации:

<p float="left">
  <img src="https://github.com/MrShaller/MrShaller/assets/62774239/880beed1-91e5-4ce0-bc77-c30a12d5e1ae" width="300"/>
  <img src="https://github.com/MrShaller/MrShaller/assets/62774239/b5c64c52-e34d-4c67-ad24-2df35aab2021" width="262"/>
  <img src="https://github.com/MrShaller/MrShaller/assets/62774239/1bde3a7a-d229-44aa-9af6-c3cc85caa73e" width="265"/>
</p>

2. Значение IOU:

IOU = 0.6376538014508436

<img src="https://github.com/MrShaller/MrShaller/assets/62774239/68fc5c4a-7f8f-4616-abb1-479b3e558cf2" width="250"/>

3. Количество обнаруженных кластеров с помощью ручной разметки и нейросетью:

<img src="https://github.com/MrShaller/MrShaller/assets/62774239/cb6d41b8-32a2-4ef2-9894-27a54a96c844" width="450"/>

4. Расчет OTR:

$N_{px}^b$ = 215 515

a = 1997.15

$k_{l}a$ = 499.29

OTR = 4.89

Задание №3: «Classical_method_usage»
-----------------------------
Критерии оценивания:

1. Результат применения классического метода детектирования пузырей:

<img src="https://github.com/MrShaller/MrShaller/assets/62774239/3e56bd17-a33f-49c8-8f62-747de5fec5a8" width="300"/>

2. Ссылка на папку с обученной моделью:
```
https://drive.google.com/drive/folders/19mM-6d_IM_ic2LAECrscRF9N1dSgzBrP?usp=sharing
```
