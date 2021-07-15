---
layout: post
title:  "Ошибки R14 и ActiveAdmin"
date:   2021-07-15 12:06:40 +0300
categories: rails active_admin
---

`ActiveAdmin` является удобным инструментом для быстрого создания `CRUD` админки, но если за ним не следить то он может очень быстро начать съедать больше памяти чем хотелось бы (R14 - `Memory Quota Exceeded`). Ниже приведу самые частые причины неожиданного роста потребления памяти:

## фильтры ActiveAdmin

По умолчанию `ActiveAdmin` создает фильтры по всем доступным аттрибутам и ассоциациям модели.
Пока данных немного это позволяет быстро создать админку со всеми фильтрами, но когда данных становится больше - это превращается в проблему.

Пример запроса из админки c продовскими данными (обратите внимание на время выполнения):

```
Started GET "/admin/student_resources" for 127.0.0.1 at 2021-07-15 12:53:04 +0300
Processing by Admin::StudentResourcesController#index as HTML
[primary]  User Load (0.6ms)  SELECT  "users".* FROM "users" WHERE "users"."id" = $1 ORDER BY "users"."id" ASC LIMIT $2  [["id", 43366], ["LIMIT", 1]]
[primary]  CaResource::StudentResource Load (5.4ms)  SELECT  "ca_resources".* FROM "ca_resources" WHERE "ca_resources"."type" IN ('CaResource::StudentResource') ORDER BY "ca_resources"."id" desc LIMIT $1 OFFSET $2 [["LIMIT", 30], ["OFFSET", 0]]
[primary]  Skill Load (888.3ms)  SELECT "skills".* FROM "skills"
  Rendered /home/ruslan/.rbenv/versions/2.4.1/lib/ruby/gems/2.4.0/gems/activeadmin-1.0.0/app/views/active_admin/resource/index.html.arb (132039.3ms)
Completed 200 OK in 132055ms (Views: 131137.4ms | ActiveRecord: 908.7ms)
```

Сами данные загрузились за 5.4ms, потом загрузились данные для фильтра `skills` за 888.3ms и дальше `ActiveAdmin` потратил чуть больше двух минут чтобы отдать нам страницу с фильтром в котором содержится 1_039_708 сущностей.

Пример без фильтра:
```
Started GET "/admin/student_resources" for 127.0.0.1 at 2021-07-15 13:04:39 +0300
Processing by Admin::StudentResourcesController#index as HTML
[primary]  User Load (1.0ms)  SELECT  "users".* FROM "users" WHERE "users"."id" = $1 ORDER BY "users"."id" ASC LIMIT $2  [["id", 43366], ["LIMIT", 1]]
[primary]  CaResource::StudentResource Load (7.7ms)  SELECT  "ca_resources".* FROM "ca_resources" WHERE "ca_resources"."type" IN ('CaResource::StudentResource') ORDER BY "ca_resources"."id" desc LIMIT $1 OFFSET $2  [["LIMIT", 30], ["OFFSET", 0]]
Completed 200 OK in 1253ms (Views: 1141.1ms | ActiveRecord: 47.1ms)
```

Страница загрузилась за 1 секунду! Хотя мы отображаем те же самые данные но без фильтра.

Советы:
- Отключайте ненужные фильтры, ведь лучший код это тот которого нет :)
- Можно написать свой собственный фильтр!

Если фильтр очень нужен то можно написать свой с автокомплитом который не будет нагружать сервер без необходимости.

1) Для автокомплита воспользуемcя JQuery Autocomplete и принициализируем наши инпут который будет отвечать за автокомплит и фильтр (`active_admin.js.coffee`):

{% highlight coffee %}
initializeFilterAutocompleteInputs = ->
  $('.autocomplete_filter_input').each () ->
    $(@).autocomplete(
      source: $(@).data('autocomplete-path'),
      minLength: 3
    )

$(document).ready ->
  initializeFilterAutocompleteInputs()
{% endhighlight %}

2) Говорим ActiveAdmin что будем использовать наш собственный инпут для фильтра в ActiveAdmin:

{% highlight ruby %}
filter :by_skill_names_in, label: 'Skills', as: :string, input_html: {
  class: 'autocomplete_filter_input', 'data-autocomplete-path': '/admin/student_resources/autocomplete_skill_names'
}

collection_action :autocomplete_skill_names, method: :get do
  skills = if params[:term].present?
    Skill
      .where(skillable_type: 'CaResource')
      .where('name ILIKE ?', "%#{params[:term]}%")
      .group(:name)
      .pluck(:name)
  else
    []
  end

  render json: skills.empty? ? [::I18n.t('api.errors.messages.not_found')] : skills
end
{% endhighlight %}

Мы сказали что будем использовать наш кастомный метод `by_skill_names` для того чтобы отфильтровать сущности + мы используем наш автокомплит инпут + создали эндпоинт для получения вариантов по автокомплиту. Дальше нам нужно определить этот самый `by_skill_names` метод для нашей модели

3) `ActiveAdmin` использует `Ransacker` под капотом, поэтому определяем наш метод для фильтра:

{% highlight ruby %}
ransacker :by_skill_names, formatter: proc { |skill_name|
  skill_table = ::Skill.arel_table
  skill_table
    .project(:skillable_id)
    .where(
      skill_table[:name].eq(skill_name).and(skill_table[:skillable_type].eq('CaResource'))
    )
  } do |parent|
    parent.table[:id]
  end
{% endhighlight %}

В этот метод передается терм из автокомплита и мы с помощью запроса на `Arel` пишем запрос.

Готово!

Замеры с нашим фильтром:
```
Started GET "/admin/student_resources" for 127.0.0.1 at 2021-07-15 13:48:11 +0300
Processing by Admin::StudentResourcesController#index as HTML
[primary]  User Load (1.0ms)  SELECT  "users".* FROM "users" WHERE "users"."id" = $1 ORDER BY "users"."id" ASC LIMIT $2  [["id", 43366], ["LIMIT", 1]]
[primary]  CaResource::StudentResource Load (3.9ms)  SELECT  "ca_resources".* FROM "ca_resources" WHERE "ca_resources"."type" IN ('CaResource::StudentResource') ORDER BY "ca_resources"."id" desc LIMIT $1 OFFSET $2  [["LIMIT", 30], ["OFFSET", 0]]
Completed 200 OK in 286ms (Views: 252.0ms | ActiveRecord: 24.2ms)
```

У нас есть тот самый фильтр но страница отдается гораздо быстрее (286ms против 132055ms). Стало быстрее в 461 раз

Замер если воспользоваться нашим фильтром и попробовать отфильтровать сущности:
```
Started GET "/admin/student_resources?utf8=%E2%9C%93&q%5Bby_skill_names_in%5D=Technology&commit=Filter&order=id_desc" for 127.0.0.1 at 2021-07-15 13:51:41 +0300
Processing by Admin::StudentResourcesController#index as HTML
  Parameters: {"utf8"=>"✓", "q"=>{"by_skill_names_in"=>"Technology"}, "commit"=>"Filter", "order"=>"id_desc"}
[primary]  User Load (0.7ms)  SELECT  "users".* FROM "users" WHERE "users"."id" = $1 ORDER BY "users"."id" ASC LIMIT $2  [["id", 43366], ["LIMIT", 1]]
[primary]  CaResource::StudentResource Load (4.9ms)  SELECT  "ca_resources".* FROM "ca_resources" WHERE "ca_resources"."type" IN ('CaResource::StudentResource') AND "ca_resources"."id" IN ((SELECT skillable_id FROM "skills" WHERE "skills"."name" = 'Technology' AND "skills"."skillable_type" = 'CaResource')) ORDER BY "ca_resources"."id" desc LIMIT $1 OFFSET $2  [["LIMIT", 30], ["OFFSET", 0]]
Completed 200 OK in 315ms (Views: 282.2ms | ActiveRecord: 22.9ms)
```

## N+1

Иногда клиент хочет видеть связанные с текущей сущностью данные на странице админки. Например `experiences` у `users`. И если просто добавить новую колонку для отображения то мы получим N+1 и запрос вида:
```
[standby]  Experience Load (0.5ms)  SELECT "experiences".* FROM "experiences" WHERE "experiences"."experiencable_id" = $1 AND "experiences"."experiencable_type" = $2 AND (experiencable_type = 'User') ORDER BY "experiences"."is_current_job" DESC, "experiences"."finish" DESC  [["experiencable_id", 154587], ["experiencable_type", "User"]]
...
[standby]  Experience Load (0.6ms)  SELECT "experiences".* FROM "experiences" WHERE "experiences"."experiencable_id" = $1 AND "experiences"."experiencable_type" = $2 AND (experiencable_type = 'User') ORDER BY "experiences"."is_current_job" DESC, "experiences"."finish" DESC  [["experiencable_id", 154582], ["experiencable_type", "User"]]
```
Проблема решается достаточно просто добавлением `includes :experiences`. Профит может быть не очень большой так как данные обычно пагинированные, но все таки лучше без N+1 :)


## Специфический случай

С помощью `NewRelic` и различных метрик в `Heroku` можно отловить что именно вызывает просадки по производительности в конкретном случае. Также помогает скачать дамп и проверить локально с реальными данными. Например случай из работы - нужно было добавить отображение статистики в админке:

{% highlight ruby %}
react_component('Admin', props: { dashboard: { trackerData: TrackerHistory.all.as_json } }, prerender: true, id: 'super', html_options: { class: 'root' })
{% endhighlight %}

Это был код вьюхи для дашборда админки поэтому никто долго не обращал внимания что там внутри в этой вьюхе происходит. Но спустя ~год это стало причиной множества R14 и очень долгой загрузки дашборда:

```
Name - dashboard; Status - 200; Type - document; Initiator - Other; Size - 10.3MB; Time - 29s
```

Страница весит 11 МБ :) Страница теперь превышает размер не только Doom для MS-DOS (2.32 MB), но и целого Doom 64 для консоли Nintendo-64 с 3D моделями (7.0MB)
Разработчик который это писал уже давно убежал а на момент ревью этот код никого не смутил;

Проблема решилась достаточно просто:
{% highlight ruby %}
react_component(
  'Admin', props: { dashboard: { trackerData: TrackerHistory.for_last_month.as_json } }, prerender: true, id: 'super', html_options: { class: 'root' }
)
{% endhighlight %}

И получаем:
```
Name - dashboard; Status - 200;	Type - document; Initiator - Other; Size - 653 kB; Time	2.24s 
```

Мораль - иногда тяжелый запрос(как в плане сложности так и в плане фактического размера возвращаемых данных) может находиться не в контроллере и вызываемых им сервисах, но и во вьюхе :)
