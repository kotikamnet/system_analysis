
```table-of-contents
```


## Задание 

Цель: Научиться определять основные элементы предметной области. Ознакомиться с процессом моделирования данных. Научиться создавать логические ERD (понимать связи, ключи, нормализацию, типы данных).

Задача: Пусть требуется создать систему, предназначенную для менеджера музыкальных групп. Такая система должна обеспечивать хранение сведений о группах, включающих название группы, год образования и страну, состав исполнителей, положение в последнем хит-параде; репертуар группы. Сведения о каждой песне из репертуара группы - это ее название, композитор, автор текста. Необходимо также хранить данные о последней гастрольной поездке каждой группы: название гастрольной программы, названия населенных пунктов, дата начала и окончания выступлений, средняя цена билета (зависит от места выступления и положения группы в хит-параде). Возможно появление новой группы и изменение состава исполнителей. Каждая песня может быть в репертуаре только одной группы. 

Менеджеру могут потребоваться следующие сведения: 
– Автор текста, композитор и дата создания песни с данным названием? В репертуар какой группы она входит? 
– Репертуар наиболее популярной группы? 
– Цена билета на последний концерт указанной группы? 
– Состав исполнителей группы с заданным названием, их возраст и амплуа? 
– Место и продолжительность гастролей группы с заданным названием? 
– Какие группы в текущем году отмечают юбилей 
– Самый молодой вокалист? Какую группу он представляет? 
– В каких группах средний возраст исполнителей не превышает 20 лет? 

Необходимо создать логическую ERD данной системы в нотации Мартина в draw.io.


## Ход выполнения

Выявить: 
1. Сущности
2. Атрибуты сущностей
3. Связи между сущностями и их типы (кардинальность)
4. Ключи и степень нормализации
	1. В таблице не должно быть дублирующих строк. В каждой ячейке таблицы хранится атомарное значение (одно не составное значение) В столбце хранятся данные одного типа. Отсутствуют массивы и списки в любом виде
	2. Таблица должна иметь ключ. Все неключевые столбцы таблицы должны зависеть от полного ключа (в случае если он составной)
	3. В таблицах должны отсутствовать неключевые столбцы, которые зависят от других неключевых столбцов. - Декомпозиция
5. Ограничения
6. Нейминг
7. Типы данных
8. Словарь

## Особенности моделирования сущностей

В ходе анализа были выявлены *основные* сущности - **Группы (groups), Песни (songs), Гастроли (tours), Концерты (concerts) Хит-парады (charts), Участники группы (group_members)**.

Между таблицами **Песни (songs) и Хит-парады (charts)** была создана промежуточная таблица **songs_charts** для хранения места конкретной песни в конкретном хит-параде.

В ходе декомпозиции было решено создать отдельную таблицу **Музыканты (musicians)**, в которую можно было бы заносить как участников групп, так и композиторов, и авторов текста, так как их хранение подходит под один формат - полное имя и год рождения. Таблица **Участники группы (group_members)** может стать промежуточной между **musicians и groups**, так как это добавит гибкости в изменение участников и добавление одному музыканту нескольких ролей. Таблица **songs** будет ссылаться на **musicians** дважды - через **composer_id и lyrics_author** (будем считать, что у песни может быть один композитор и автор текста).

Также были отдельно выделены таблицы-справочники - **Роли (roles), Населенные пункты (cities) и Страны (countries)**, так при занесении этих данных скорее всего будет наблюдаться дупликация.

## ERD 

![img1.drawio.png](Homeworks/attachments/img1.drawio.png)

## Словарь данных

https://docs.google.com/spreadsheets/d/1yP1APCrIhrZGJQqxGUzbT6rsoQaI6pT053JuLjSc78I/edit?usp=sharing

## Проверка условий

На этом этапе проверяем, можно ли написать SQL скрипты для отображения всех необходимых данных. 

– Автор текста, композитор и дата создания песни с данным названием? В репертуар какой группы она входит? 

```sql
-- Надо объединить три таблицы - songs, musicians, groups
-- Из songs взять Название песни song_name
-- Из musicians взять имя Композитора и Автора текста (full_name) по fk и Дату создания creation_date
-- Из groups взять Название группы group_name по fk

SELECT 
    s.song_name AS 'Название песни',
    m.full_name AS 'Композитор',
    l.full_name AS 'Автор текста',
    s.creation_date AS 'Дата создания песни',
    g.group_name AS 'Название группы'
FROM songs s
JOIN musicians m ON s.composer_id = m.person_id
JOIN musicians l ON s.lyrics_author_id = l.musician_id
JOIN groups g ON s.group_id = g.group_id
WHERE s.song_name = 'Название песни';
```

– Репертуар наиболее популярной группы? 

```sql
-- Считаем, что наиболее популярная группа = группа, среднее место всех песен в последнем хит-параде выше других
-- Выбрать все песни группы с более высокой средней позицией всех песен группы
-- Выбрать название песни, где id группы в таблицах songs и groups совпадают

SELECT s.song_name  
FROM songs s  
WHERE s.group_id = (  
	SELECT g.group_id  
	FROM groups g  
	JOIN songs s2 ON s2.group_id = g.group_id  
	JOIN songs_charts sc ON sc.song_id = s2.song_id  -- Объединить таблицы groups, songs, songs_charts 
	WHERE sc.chart_id = (SELECT MAX(chart_id) FROM charts)  -- где id хит-парада соответствует последнему по дате
	GROUP BY g.group_id  -- сгруппировать по муз. группе
	ORDER BY AVG(sc.song_position) ASC  -- отобразить среднее значение по возрастанию
	LIMIT 1  -- оставить только одну первую позицию
	);

```

– Цена билета на последний концерт указанной группы? 

```sql
-- Выбрать цену билета на концерт заданной группы из объединенной таблицы concerts, tours и groups, где дата концерта максимальная равна максимальной дате (из объединенной таблицы concerts, tours и groups) 

SELECT c.ticket_price  
FROM concerts c  
JOIN tours t ON c.tour_id = t.tour_id  
JOIN groups g ON t.group_id = g.group_id  
WHERE g.name = 'Название группы'  
AND c.concert_date = (  
	SELECT MAX(c2.concert_date)  
	FROM concerts c2  
	JOIN tours t2 ON c2.tour_id = t2.tour_id  
	JOIN groups g2 ON t2.group_id = g2.group_id  
	WHERE g2.name = 'Название группы'  
	);
```

– Состав исполнителей группы с заданным названием, их возраст и амплуа? 

```sql
-- Выбрать имя, рассчитать возраст по дате рождения и роль музыканта
-- Из объединенных таблиц groups, group_members, musicians, roles
-- По названию заданной группы 

SELECT 
    p.full_name AS "Исполнитель",
    DATE_PART('year', AGE(CURRENT_DATE, m.birth_date)) AS "Возраст",
    r.role_name AS "Амплуа"
FROM groups g
JOIN group_members gm ON g.group_id = gm.group_id
JOIN musicians m ON gm.person_id = m.person_id
JOIN roles r ON gm.role_id = r.role_id
WHERE g.name = 'Название группы';
```

– Место и продолжительность гастролей группы с заданным названием? 

```sql
-- Длительность тура ищем по end_date - start_date, которые у нас уже просчитаны в таблице tours по датам концертов

SELECT  
g.group_name AS "группа",  
t.tour_name AS "тур",  
t.start_date,  
t.end_date,  
t.end_date - t.start_date AS 'длительность'  
FROM groups g  
JOIN tours t ON g.group_id = t.group_id  
WHERE g.name = 'Название группы';


-- Города тура ищем по местам проведения city_name концертов в таблице concerts

SELECT DISTINCT city.city_name  
FROM groups g  
JOIN tours t ON g.group_id = t.group_id  
JOIN concerts c ON t.tour_id = c.tour_id  
JOIN cities city ON c.city_id = city.city_id  
WHERE g.group_name = 'Название группы';

```

– Какие группы в текущем году отмечают юбилей 

```sql
-- (текущий год - год основания founded_year) кратен 10
SELECT  
g.group_name AS "группа",  
g.founded_year,  
EXTRACT(YEAR FROM CURRENT_DATE) - g.founded_year AS anniversary  
FROM groups g  
WHERE  
(EXTRACT(YEAR FROM CURRENT_DATE) - g.founded_year) % 10 = 0;
```

– Самый молодой вокалист? Какую группу он представляет? 

```sql
-- 1. Ищем вокалиста Искать по объединенным таблицам musicians, group_members, roles, groups. Отобрать по role_name в таблице group_members = вокалист
-- 2. Самый молодой = максимальный birth_date. Сравниваем его дату рождения с самой поздней, т.е birth_date = мексимальный birth_date из выборки musicians, group_members, roles, где role_name = вокалист

SELECT 
    m.full_name AS "вокалист",
    g.group_name AS "группа"
FROM musicians m
JOIN group_members gm ON m.person_id = gm.person_id
JOIN roles r ON gm.role_id = r.role_id
JOIN groups g ON gm.group_id = g.group_id
WHERE r.role_name = 'вокалист'
AND m.birth_date = (
    SELECT MAX(m2.birth_date)
    FROM musicians m2
    JOIN group_members gm2 ON m2.person_id = gm2.person_id
    JOIN roles r2 ON gm2.role_id = r2.role_id
    WHERE r2.role_name = 'вокалист'
);
```

– В каких группах средний возраст исполнителей не превышает 20 лет? 

```sql
-- Из объединенных таблиц groups, musicians, group_members проведем группировку по названию группы groups, участники group_members которой имеют средний birth_date в таблице musicians не больше 20 лет.   

SELECT 
    g.group_name AS "группа",
    AVG(DATE_PART('year', AGE(CURRENT_DATE, m.birth_date))) AS avg_age
FROM groups g
JOIN group_members gm ON g.group_id = gm.group_id
JOIN musicians m ON gm.person_id = m.person_id
GROUP BY g.group_id, g.group_name
HAVING AVG(DATE_PART('year', AGE(CURRENT_DATE, p.birth_date))) <= 20;
```
