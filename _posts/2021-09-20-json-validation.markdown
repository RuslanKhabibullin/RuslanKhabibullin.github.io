---
layout: post
title:  "Валидация JSON в Rails"
date:   2021-09-20 13:24:40 +0300
categories: rails
---
Иногда нам приходится добавлять JSON аттрибут в нашу базу данных для удобной поддержки слабо-структурированных данных. Например:
- социальные сети пользователя. Он может указать Вконтакте, Facebook, Twitter и так далее. Все это можно хранить в 1 JSON аттрибуте;
- настройки нотификации пользователя. Какие письма он хочет получать, какие нет;
- конфиги/сериализованные представления объектов - представление объекта `IceCube` для рекурентных событий, конфиги на основании которых приложение должно выглядеть/работать по другому и т.д.;

## контекст

Допустим имеется некий аналог трелло с дополнительными фичами:
- Пользователь может создать доску для своих карточек - введя название доски
- Пользователь может создать карточку внутри своей доски - надо ввести название, описание и указать приоритет карточки
- Пользователь может увидеть созданную карточку и свое имя в поле "Автор" у созданной карточки
- Администратор может слегка изменить форму создания карточки + создать шаблон по умолчанию для всех новых карточек:
  - Aдмин может изменить плейсхолдеры в форме для названия, описания карточки
  - Админ может задать шаблон для новых карточек - предустановленные значения названия, описания и приоритета
  - Админ может задать пользователя для шаблонной карточки - его имя будет отображаться в графе "Автор"

Все эти настройки админа решено хранить в поле JSON(B) таблицы `card_templates`. Структура конфига:
{% highlight javascript %}
{
  form_title: "Create your card",
  title_hint: "Enter card title",
  title_placeholder: "My new card",
  description_hint: "Enter card description",
  description_placeholder: "Need to increase my profit",
  default_card: {
    title: "My new card",
    description: "Need to increase my profit",
    priority: {
      value: "M",
      effort: "M",
      number_value: 1
    },
    author: {
      name: "John Doe"
    }
  } 
}
{% endhighlight %}

## проблема

Как было упомянуто выше, мы используем JSON из-за его гибкой структуры, но это не значит что пользователь может ввести все что захочет. Мы (разработчики приложения) сами задаем удобную для нас структуру JSON аттрибута и пишем логику исходя из этой структуры. Любое отклонение от зафиксированной схемы может вызвать ошибку в приложении. Например, мы сделали API для того чтобы администратор мог создавать свои шаблоны, но практически сразу начали замечать что все созданные конфиги отличаются по структуре от той, что мы ожидаем - нам нужна валидация. Увы, `ActiveRecord` на данный момент не позволяет нам удобно описывать валидации для наших JSON данных, поэтому нам нужно придумать что-то еще.

## data transfer object

Нам нужен некоторый объект, который получает любой хеш, возвращая при этом данные в строго заданной структуре. Для таких целей может подойти DTO (Data transfer object), задача которого как раз в этом и состоит - он нужен для передачи данных между разными частями приложения, при этом эти данные он может структурировать так, как это требуется для работы приложения.

## решение

Мы решили что воспользуемся паттерном DTO и теперь нужно его реализовать. Или не нужно? :)

`dry-struct` ([link](https://dry-rb.org/gems/dry-struct/1.0/)) - один из гемов `dry-rb` реализует как раз то, что нам нужно, предлагая удобный способ для описания наших структур. Файловая иерархия DTO в нашем проекте:

![Файловая иерархия DTO](/assets/images/2021-09-20_1.png)

Каждый файл по отдельности:

**types.rb**
{% highlight ruby %}
module Types
  include Dry.Types()
end
{% endhighlight %}

Нэймспейс где будут храниться типы из `dry-struct` для описания наших структур

**base_dto.rb**
{% highlight ruby %}
class BaseDto < Dry::Struct
  transform_keys(&:to_sym)

  DEFAULT_STRING_VALUE = ''.freeze
end
{% endhighlight %}

Базовый класс для всех наших DTO. Здесь же есть инструкция что ключи входного хеша должны быть преобразованы в символы

**templates/card_settings_dto.rb**
{% highlight ruby %}
module Templates
  class CardSettingsDto < BaseDto
    DEFAULT_CARD_VALUE = Templates::CardSettings::ExampleDto.new.to_h.freeze

    attribute :form_title, Types::Strict::String.default(DEFAULT_STRING_VALUE)
    attribute :title_hint, Types::Strict::String.default(DEFAULT_STRING_VALUE)
    attribute :title_placeholder, Types::Strict::String.default(DEFAULT_STRING_VALUE)
    attribute :description_hint, Types::Strict::String.default(DEFAULT_STRING_VALUE)
    attribute :description_placeholder, Types::Strict::String.default(DEFAULT_STRING_VALUE)
    attribute :default_card, Templates::CardSettings::ExampleDto.default(DEFAULT_CARD_VALUE)
  end
end
{% endhighlight %}

Описали структуру конфига, который может создать администратор. Аттрибут `default_card` - сложная структура, описываемая в другом DTO классе

**templates/card_settings/example_dto.rb**
{% highlight ruby %}
module Templates
  module CardSettings
    class ExampleDto < BaseDto
      DEFAULT_AUTHOR = Templates::CardSettings::AuthorDto.new.to_h.freeze
      DEFAULT_PRIORITY = Templates::CardSettings::PriorityDto.new.to_h.freeze

      attribute :title, Types::Strict::String.default(DEFAULT_STRING_VALUE)
      attribute :description, Types::Strict::String.default(DEFAULT_STRING_VALUE)
      attribute :priority, Templates::CardSettings::PriorityDto.default(DEFAULT_PRIORITY)
      attribute :author, Templates::CardSettings::AuthorDto.default(DEFAULT_AUTHOR)
    end
  end
end
{% endhighlight %}

Описали структуру дефолтной карточки. Аттрибуты `priority` и `author` - вложенные объекты, описанные в других DTO классах

**templates/card_settings/priority_dto.rb**
{% highlight ruby %}
module Templates
  module CardSettings
    class PriorityDto < BaseDto
      DEFAULT_PRIORITY_VALUE = 'm'.freeze
      PRIORITY_VALUES = %w[s m l xl].freeze
      DEFAULT_PRIORITY_NUMBER_VALUE = 1
      PRIORITY_NUMBER_VALUES = (0..3).to_a.freeze

      attribute :value, Types::Strict::String
        .default(DEFAULT_PRIORITY_VALUE)
        .constrained(included_in: PRIORITY_VALUES)
      attribute :effort, Types::Strict::String
        .default(DEFAULT_PRIORITY_VALUE)
        .constrained(included_in: PRIORITY_VALUES)
      attribute :number_value, Types::Coercible::Integer
        .default(DEFAULT_PRIORITY_NUMBER_VALUE)
        .constrained(included_in: PRIORITY_NUMBER_VALUES)
    end
  end
end
{% endhighlight %}

Описали шейп объекта приоритетов + добавили констрейнты для допустимых значении.

**templates/card_settings/author_dto.rb**
{% highlight ruby %}
module Templates
  module CardSettings
    class AuthorDto < BaseDto
      attribute :name, Types::Strict::String.default(DEFAULT_STRING_VALUE)
    end
  end
end
{% endhighlight %}

Шейп объекта `author` состоит просто из его имени.

## примеры использования

Пустой хеш:

{% highlight ruby %}
[1] pry(main)> empty_hash = {}
=> {}
[2] pry(main)> card_settings = ::Templates::CardSettingsDto.new(empty_hash)
=> #<Templates::CardSettingsDto form_title="" title_hint="" title_placeholder="" description_hint="" description_placeholder="" default_card={:title=>"", :description=>"", :priority=>{:value=>"m", :effort=>"m", :number_value=>1}, :author=>{:name=>""}}>
[3] pry(main)> card_settings.to_h
=> {:form_title=>"",
 :title_hint=>"",
 :title_placeholder=>"",
 :description_hint=>"",
 :description_placeholder=>"",
 :default_card=>{:title=>"", :description=>"", :priority=>{:value=>"m", :effort=>"m", :number_value=>1}, :author=>{:name=>""}}}
{% endhighlight %}

Как видно из примера выше, наш DTO проставляет значения согласно тем значениям по умолчанию, что мы задали.

Хеш, содержащий неправильные значения (нарушающее `constrained` правила):

{% highlight ruby %}
[4] pry(main)> invalid_hash = { default_card: { priority: { value: 'wrong' } } }
=> {:default_card=>{:priority=>{:value=>"wrong"}}}
[5] pry(main)> card_settings = ::Templates::CardSettingsDto.new(invalid_hash)
Dry::Struct::Error: [Templates::CardSettings::PriorityDto.new] "wrong" (String) has invalid type for :value violates constraints (included_in?(["s", "m", "l", "xl"], "wrong") failed)
from /home/ruslan/.rbenv/versions/3.0.2/lib/ruby/gems/3.0.0/gems/dry-types-1.5.1/lib/dry/types/schema.rb:330:in `rescue in block in resolve_unsafe'
Caused by Dry::Types::SchemaError: "wrong" (String) has invalid type for :value violates constraints (included_in?(["s", "m", "l", "xl"], "wrong") failed)
from /home/ruslan/.rbenv/versions/3.0.2/lib/ruby/gems/3.0.0/gems/dry-types-1.5.1/lib/dry/types/schema.rb:330:in `rescue in block in resolve_unsafe'
Caused by Dry::Types::ConstraintError: "wrong" violates constraints (included_in?(["s", "m", "l", "xl"], "wrong") failed)
from /home/ruslan/.rbenv/versions/3.0.2/lib/ruby/gems/3.0.0/gems/dry-types-1.5.1/lib/dry/types/constrained.rb:42:in `call_unsafe'
{% endhighlight %}

При передаче значении, непредусмотренных `constrained` директивой мы получаем исключение.

Хеш, содержащий правильные значения, но имеющий лишние аттрибуты + некоторые аттрибуты структуры отсутствуют:
{% highlight ruby %}
some_hash = { unused_attribute: 'UID12345', form_title: 'Create your card', title_hint: 'Enter card title', description_hint: 'Enter card description', default_card: { title: 'My new card', author: { name: 'John Doe' }, priority: { value: 'm', effort: 's', number_value: 2 } } }
=> {:unused_attribute=>"UID12345",
 :form_title=>"Create your card",
 :title_hint=>"Enter card title",
 :description_hint=>"Enter card description",
 :default_card=>{:title=>"My new card", :author=>{:name=>"John Doe"}, :priority=>{:value=>"m", :effort=>"s", :number_value=>2}}}
[9] pry(main)> card_settings = ::Templates::CardSettingsDto.new(some_hash)
=> #<Templates::CardSettingsDto form_title="Create your card" title_hint="Enter card title" title_placeholder="" description_hint="Enter card description" description_placeholder="" default_card=#<Templates::CardSettings::ExampleDto title="My new card" description="" priority=#<Templates::CardSettings::PriorityDto value="m" effort="s" number_value=2> author=#<Templates::CardSettings::AuthorDto name="John Doe">>>
[10] pry(main)> card_settings.to_h
=> {:form_title=>"Create your card",
 :title_hint=>"Enter card title",
 :title_placeholder=>"",
 :description_hint=>"Enter card description",
 :description_placeholder=>"",
 :default_card=>{:title=>"My new card", :description=>"", :priority=>{:value=>"m", :effort=>"s", :number_value=>2}, :author=>{:name=>"John Doe"}}}
{% endhighlight %}

Наш DTO вырезал ненужные аттрибуты + добавил пропущенные аттрибуты, согласно их значению по умолчанию.

## заключение

C помощью паттерна DTO и ее реализации в виде `dry-types` можно достаточно просто добавить валидацию к нашим JSON данным. Также это позволит явно описать шейп нашего JSON - объекта и обращаться к нему, чтобы вспомнить что-же хранится в нашем объекте и какие у него могут быть значения :)  
