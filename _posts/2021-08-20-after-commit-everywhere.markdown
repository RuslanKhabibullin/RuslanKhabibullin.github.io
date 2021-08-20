---
layout: post
title:  "after_commit и фоновые задачи"
date:   2021-08-20 12:30:40 +0300
categories: rails
---
Давайте взглянем на следующий код:

{% highlight ruby %}
class CreateUserWithWelcomeMessage
  attr_reader :email, :user
  private :email

  def initialize(email)
    @email = email
  end

  def call
    return false unless user.save

    UserMailer.welcome(user.id).deliver_later
    true
  end

  private

  def user
    @user ||= User.new(email: email)
  end
end

# user_mailer.rb

def welcome(user_id)
  user = User.find_by(id: user_id)
  return unless user

  ...
end
{% endhighlight %}

Здесь происходит создание пользователя и отправка письма с приветствием. Отправка письма производится асинхронно в фоновом воркере (используя `Sidekiq`, например) чтобы не задерживать сервер. Какие тут могут быть проблемы?

## проблема

Проблема не самая очевидная, потому что иногда код будет отрабатывать до конца и пользователь увидит письмо в своем почтовом ящике, но в остальных случаях поиск пользователя по id в методе `#welcome` будет возвращать `nil` - пользователь не найден. При этом, если бы отправка письма была синхронной (`UserMailer.welcome(user.id).deliver_now`), то никаких проблем бы не возникло. Почему так происходит?

Это происходит потому что на момент вызова асинхронного воркера транзакция в базе данных еще не завершилась.

## решение

Решений несколько:

- Делать все в одном процессе без фоновых задач :)
- Убедиться что код будет выполняться только после того, как произошел `commit` в базу данных и транзакция завершилась

Пожалуй, мы можем пропустить первый вариант и начать сразу со второго. `ActiveRecord` предоставляет нам самые разнообразные коллбеки, в том числе и `after_commit` - код в этом коллбеке будет выполняться только после того, как транзакция успешно "прошла" в базе данных. То, что нужно!

{% highlight ruby %}
class User < ApplicationRecord
  after_commit :send_welcome_email, on: :create

  private

  def send_welcome_email
    UserMailer.welcome(id).deliver_later
  end
end
{% endhighlight %}

Теперь, после создания профиля пользователь гарантированно получит письмо, отправленное в фоне.

А что если мы не хотим чтобы при каждом создании пользователя ему отправлялись письма? Например, мы решили быстро создать пользователя в консоли, но после его создания ему сразу отправляется какое-то письмо. Бизнес-логика приложения не должна содержаться в коллбеках модели - это становится очень сложно контролировать и адаптировать под различные ситуации. Правильнее будет использовать наш старый сервис, не трогая при этом саму модель `User`. Но как использовать `after_commit` в наших сервисах за пределами моделей `ActiveRecord`?

[after_commit_everywhere](https://github.com/Envek/after_commit_everywhere) - гем, позволяющии использовать `after_commit` и другие коллбеки за пределами моделей `ActiveRecord`. Используя этот гем, можно переписать наш сервис следующим образом:

{% highlight ruby %}
class CreateUserWithWelcomeMessage
  include AfterCommitEverywhere

  attr_reader :email, :user
  private :email

  def initialize(email)
    @email = email
  end

  def call
    ActiveRecord::Base.transaction do
      user.save!

      after_commit { UserMailer.welcome(user.id).deliver_later }

      true
    end
  rescue ActiveRecord::ActiveRecordError
    false
  end

  private

  def user
    @user ||= User.new(email: email)
  end
end
{% endhighlight %}

Теперь сервис работает без ошибок, а модель `User` не содержит в себе никаких неожиданностей.

## итого

Таким образом, благодаря `after_commit_everywhere` мы можем пользоватся преимуществами `after_commit` коллбека за пределами моделей и хранить нашу бизнес-логику там, где ей место - в сервисах (command, interactor и т.д) 
