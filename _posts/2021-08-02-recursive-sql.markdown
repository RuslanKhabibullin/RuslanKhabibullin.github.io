---
layout: post
title:  "Рефакторинг с помощью рекурсивного SQL запроса"
date:   2021-08-02 11:29:40 +0300
categories: postgres
---
Иногда наши данные в базе имеют иерархический вид и нам нужно эти данные эффективно отдавать и обслуживать. Для решения этой задачи можно использовать рекурсивные SQL запросы.

## контекст

Имеется сущность `preferences_themes` которая хранит различные темы и их уровень в иерархии. Из этих тем пользователь может выбрать области своей экспертизы. Пример:
```
id: 181, name: "Finance", level: "main_theme",
id: 182, name: "Consulting", level: "main_theme"
id: 184, name: "Sales", level: "main_theme"
id: 185, name: "General business", level: "main_theme"
id: 183, name: "Technology", level: "main_theme"

id: 207, name: "Back-end", level: "sub_theme"
id: 208, name: "Data Sciences/ Analysis", level: "sub_theme"
id: 209, name: "Design", level: "sub_theme"
id: 210, name: "Devops", level: "sub_theme"
id: 211, name: "Front-end", level: "sub_theme"

id: 260, name: "Ruby", level: "focus"
```
C помощью связующей таблицы `preferences_themes_parents` (`theme_id`, `parent_id`) мы связываем `main_theme` вместе с `sub_theme`, `sub_theme` вместе с `focus` и т.д.

{% highlight ruby %}
<!-- PreferencesTheme -->
has_many :themes_parents, class_name: 'PreferencesThemesParent', foreign_key: :theme_id
has_many :themes_children, class_name: 'PreferencesThemesParent', foreign_key: :parent_id
has_many :parents, through: :themes_parents, class_name: 'PreferencesTheme', foreign_key: :parent_id
has_many :children, through: :themes_children, class_name: 'PreferencesTheme', source: :theme, foreign_key: :theme_id

<!-- PreferencesThemesParent -->
belongs_to :theme, class_name: 'PreferencesTheme', foreign_key: :theme_id
belongs_to :parent, class_name: 'PreferencesTheme', foreign_key: :parent_id
{% endhighlight %}

Тем самым пользователь теперь может выбирать область своей экспертизы: `Technology -> Back-end -> Ruby`.

## проблема

При переписывании приложения на SPA бэкенд отдает на фронт такой ответ:

{% highlight javascript %}
{
  "main_themes": [{ "id": 183, "name": "Technology", "level": "main_theme", "children": [207, 208, 209, 210, 211] }, ...],
  "sub_themes": [{ "id": 207, "name": "Back-end", "level": "sub_theme", "children": [260, 247, 248, 249, 251, 253, 255, ...] }, ...],
  "skills_families": [{ "id": 907, name: "Databases", "level": "skills_family", "children": [...] }, ...],
  "focuses": [{ "id": 260, "name": "Ruby", "level": "focus", "children": [] }, ...]
}
{% endhighlight %}

Мы возвращаем все темы сгруппированные по уровню в иерархии + данные о под-темах для каждой темы (`children`). Порядок иерархии обговорен заранее и выглядит следующим образом:
```
main_themes => sub_themes => skills_families => focuses
```
Это все нужно для того чтобы фронтенд смог правильно все отобразить на форме. Это решение работает пока тем не так много. Но по мере роста данных возвращать все темы - плохая идея;
Также у нас тут получается 8 SQL запросов - по 2 на категорию (второй из-за команды `includes` для подгрузки `children`).

Замеры с продакшен дампа:

```
GET api/v1/preferences_themes:

Completed 200 OK in 4967.41ms (Views: 4839.21ms | DB: 128.2ms)
```

5 секунд - многовато. А что по памяти? (`memory_profiler` gem)

```
Total allocated: 33.26 MB (422177 objects)
Total retained:  458.71 kB (4270 objects)
```

33 МБ - также много для того чтобы обслужить 1 эндпоинт, обращение к которому происходит достаточно часто. Что мы можем сделать чтобы все это исправить?

## решение

Можно использовать рекурсивный SQL запрос + возвращать только те темы и иерархию, которые требуются пользователю в конкретный момент времени.

Пример SQL запроса который загружает всю иерархию тем для `Technology` секции (id=183) за 1 запрос:

{% highlight sql %}
WITH RECURSIVE preferences_theme_family AS (
  SELECT theme_id, parent_id, name, level
  FROM preferences_themes_parents
      INNER JOIN preferences_themes ON preferences_themes.id = theme_id
  WHERE parent_id = 183
  UNION
  SELECT preferences_themes_parents.theme_id,
         preferences_themes_parents.parent_id,
         preferences_themes.name,
         level
  FROM preferences_themes_parents
      INNER JOIN preferences_themes ON preferences_themes.id = preferences_themes_parents.theme_id
      INNER JOIN preferences_theme_family ON preferences_theme_family.theme_id = preferences_themes_parents.parent_id
)
 SELECT
        level,
        json_agg(
            json_build_object(
                'id', theme_id,
                'name', name,
                'level', level,
                'children', children
                )::json
            ) AS themes
FROM preferences_theme_family
    LEFT OUTER JOIN (
        SELECT
               parent_id,
               json_agg(
                   json_build_object(
                       'id', theme_id,
                       'level', level
                       )::json
                   ) children
        FROM preferences_theme_family
        GROUP BY parent_id
    ) preferences_theme_children ON preferences_theme_children.parent_id = preferences_theme_family.theme_id
GROUP BY level
{% endhighlight %}

Этот запрос можно спрятать под слой абстракции (`QueryObject`) и вызывать например так: `PreferencesThemeHierarchyQuery.new(183).all`;

Также потребуется изменить/добавить эндпоинт и подгружать иерархию только для выбранной пользователем темы; Это увеличит количество запросов, но сократит время выполнения каждого запроса. Это имеет смысл в моем случае потому что пользователям чаще всего интересна только их область экспертизы и не имеет смысла подгружать все остальные.

С учетом вышеперечисленных изменении можно сделать новые замеры:

```
GET api/v1/preferences_themes?parent_id=183:
Completed 200 OK in 398.25ms (Views: 319.1ms | DB: 79.15ms)
```

`398 ms` вместо 5 секунд. Также появилась некоторая сложность в виде нашего кастомного `SQL` кода вместо методов `ActiveRecord`, но в данный момент нам важнее производительность. А что там по памяти? 

```
Total allocated: 1.92 MB (16320 objects)
Total retained:  103.22 kB (1165 objects)
```

`1.92 MB` вместо `33.26 MB` - намного лучше.

## итого

Если ваши данные имеют иерархический вид и вы неудовлетворены скоростью работы эндпоинтов отдающих эти данные, то можно использовать рекурсивные SQL запросы. Также можно переделать саму логику эндпоинта если проект/форма допускает некоторые изменения в угоду эффективности. Также следует провести замеры до/после чтобы принять окончательное решение в пользу рекурсивного `SQL` запроса или стандартных методов `ActiveRecord`. Если прирост небольшой, то имеет смысл использовать методы `ActiveRecord` т.к кастомный `SQL` запрос усложнит логику и увеличит порог входа для других разработчиков.
