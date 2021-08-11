---
layout: post
title:  "date_trunc - заменяем N SQL запросов одним"
date:   2021-08-09 17:01:40 +0300
categories: postgres
---
Довольно часто в админке требуется добавить некоторый дэшборд в котором можно будет наблюдать за различной активностью. Например:
```
Вывести статистику по последним n неделям кто сколько написал постов?
```

Самое простое решение как это обычно бывает не является оптимальным.

## контекст

В нашем проекте потребовалось вывести статистику отвечающую на следующий вопрос:
```
Сколько кандидатов админы порекомендовали клиентам за последние 7 недель?
```

Самый простой вариант решения:
{% highlight ruby %}
admin_ids = User.where(admin: true).pluck(:id)
week_begin = DateTime.now.beginning_of_week

result = []

7.times do
  week_end = week_begin.end_of_week
  suggested_candidates_count = Job::Application::Log
                               .where(created_at: week_begin..week_end, user_id: admin_ids, action: 'suggested')
                               .group(:user_id)
                               .count

  result.push(
    week_begin: week_begin,
    week_end: week_end,
    all_suggested_candidates_by_user: suggested_candidates_count,
  )
  week_begin = week_begin.prev_week
end

render json: { suggestions: result }
{% endhighlight %}

Этот код считает количесто порекомендованных кандидатов каждым админом за последние 7 недель и возвращает эту статистику.

## проблема

Что если потребуется статистика за 30 недель? В нашем простом решении будет 30 SQL запросов. Количество запросов зависит от количества недель которые требуется отобразить. Можно также замерить все это:

```
Total allocated: 2.01 MB (24433 objects)
Total retained:  170.44 kB (1807 objects)
Time: 3 sec
```

Время замерил просто запомнив таймстамп в начале операции и в конце - разница между этими значениями и есть время выполнения операции в моих замерах;

Может быть можно это переписать чтобы количество запросов не зависело от количества недель?

## решение

Можно использовать функцию `date_trunc` и группировать записи по неделям сразу внутри `Postgres`. Например следующий SQL запрос как-раз вернет необходимую статистику по неделям:

{% highlight sql %}
SELECT
  date_trunc('week', created_at::date) AS week,
  jsonb_agg(
    DISTINCT jsonb_build_object(
      'id', suggested_candidates_logs.user_id,
      'count', suggested_candidates_logs.logs_count
    )
  ) AS suggested_candidates
FROM job_application_logs
LEFT OUTER JOIN (
  SELECT user_id, date_trunc('week', created_at::date) AS week, COUNT(id) AS logs_count
  FROM job_application_logs
  WHERE job_application_logs.action = 'suggested' AND user_id IN (:user_ids)
  GROUP BY week, user_id
) suggested_candidates_logs ON suggested_candidates_logs.week = date_trunc('week', job_application_logs.created_at::date)
WHERE created_at > :week_begin AND action = 'suggested'
GROUP BY date_trunc('week', job_application_logs.created_at::date);
{% endhighlight %}

Подставив вместо `:user_ids` id-шники админов, вместо `:week_begin` начало интересующего нас промежутка, и можно получить все необходимые данные за 1 SQL запрос. Но сложность восприятия нашего кода выросла - поэтому нужно убедиться что это того стоило и провести новые замеры:

```
Total allocated: 1.98 MB (20494 objects)
Total retained:  185.22 kB (1986 objects)
Time: 0 sec
```

Мы не особо выиграли по использованию памяти, но наш кастомный SQL запрос выполняется гораздо быстрее 30-ти запросов выполненных с помощью ActiveRecord.

## итого

Мы можем получать еженедельную статистику средствами самого `Postgres` - используя 1 запрос с `date_trunc`. Однако это решение увеличивает сложность восприятия нашего кода, поэтому нужно взвесить все за и против - может быть в конкретном случае лучше оставить несколько понятных и простых запросов вместо одного?  
