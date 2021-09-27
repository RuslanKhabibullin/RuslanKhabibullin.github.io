---
layout: post
title:  "Мой первый опыт контрибьюта в Open Source"
date:   2021-09-27 12:43:40 +0300
categories: rails
---
Наконец то смог внести небольшой вклад в Ruby-сообщество и исправить маленький баг в геме :)

## контекст

[ar_lazy_peload](https://github.com/DmitryTsepelev/ar_lazy_preload) - удобный гем от [EvilMartians](https://evilmartians.com/) для устранения N+1 (особенно удобен при написании GraphQL API). Является неким аналогом предзагрузки ассоциации `.includes`, но загружает только те ассоциации что были запрошены по факту, а не автоматически все перечисленные в `.includes`.

Гем использовался в приложении Rails 6.1 + Ruby 3.0.2

## баг

Баг удалось обнаружить лишь благодаря контексту нашего приложения, в котором этому гему пришлось работать:
- Rails 6.1
- Ruby 3.0.2
- `enum` аттрибут в модели ActiveRecord, где одно из значении - `owner`
{% highlight ruby %}
require_dependency 'board'

class Board
  class Member < ::ApplicationRecord
    belongs_to :board
    belongs_to :user

    ADMIN_ROLES = %i[owner admin].freeze

    scope :admins, -> { where(role: ADMIN_ROLES) }

    enum role: { owner: 0, admin: 1, contributor: 2 }
  end
end
{% endhighlight %}

Попытки сделать запросы к этой модели вызывали ошибку `Stack level is too deep`, по которой я создал issue на странице гема: [link](https://github.com/DmitryTsepelev/ar_lazy_preload/issues/49)

## решение

Проследив по стек-трейсу ошибки было найдено проблемное место. Оказалось что метод `owner` уже определен в геме (через `delegate`). Фикс оказался достаточно простым, и я оформил пулл-реквест с решением проблемы - [link](https://github.com/DmitryTsepelev/ar_lazy_preload/pull/50).

## итого

Даже несмотря на то что баг который я пофиксил не являлся чем-то супер серьезным, тем не менее приятно от того, что тебе удалось помочь сообществу и ты теперь можешь видеть свое имя в контрибьюторах проекта :)

![Я в контрибьюторах проекта](/assets/images/2021-09-27_1.png)
