---
layout: post
title: Four alternatives to using ActiveRecord Callbacks and Observers
---

An exhaustive amount of literature has been written about ActiveRecord callbacks, and in particular why they can be a dreadful idea. I don't want to repeat what many others said; rather, I want to demonstrate four alternatives to using callbacks.

First, let's look at a scenario. You have probably seen this example before.

{% highlight ruby %}
class User < ActiveRecord::Base
  after_create :send_welcome_email

private

  def send_welcome_email
    UserMailer.welcome_email(self).deliver_later
  end
end
{% endhighlight %}

A user registers, and he or she is sent a welcome e-mail. We already know that in most cases this is a bad[^1] idea. Let's look other ways to do this without using callbacks.

### 1. Send the e-mail from the controller

The first and most obvious way is to send the e-mail from the controller.

{% highlight ruby %}
class UsersController < ApplicatinController::Base
  def create
    @user = User.new(user_params)
    if @user.save
      UserMailer.welcome_email(@user).deliver_later
      redirect_to @user
    else
      render :new
    end
  end
end
{% endhighlight %}

This is better because the code is explicit. Calling `@user.save` saves the user instance. And that's it. It does not send an e-mail as a side effect. Instead, we send the e-mail explicitly.

Moving the logic to the controller is an improvement, but our code is not reusable. We might have to send an e-mail after a user is created in other places (for example, from an admin panel or through an API call), and we will end up repeating ourselves.

### 2. Use a service object

I like service objects because they encapsulate bits of reusable functionality, and they are easy to test.

{% highlight ruby %}
# app/services/create_user.rb
class CreateUser
  attr_reader :user, :mailer

  def initialize(user, mailer: UserMailer)
    @user = user
    @mailer = mailer
  end

  def call
    if user.save
      mailer.welcome_email(user).deliver_later
    end
  end
end
{% endhighlight %}

This service object can be used anywhere--the console, a rake task, or another controller--free from side effects.

{% highlight ruby %}
class UsersController < ApplicatinController::Base
  def create
    @user = User.new(user_params)
    service = CreateUser.new(@user)
    if service.call
      redirect_to @user
    else
      render :new
    end
  end
end
{% endhighlight %}

It's also extensible. Let's say that, besides sending an e-mail, we need to create an activity. Extending the functionality of the service object is easy.

{% highlight ruby %}
class CreateUser
  # ...
  def call
    if user.save
      mailer.welcome_email(user).deliver_later
      UserActivityJob.perform_later(user)
    end
  end
end
{% endhighlight %}

### 3. Use a model method

If service objects are not your thing, you can replace the callback with a regular method in the model.

{% highlight ruby %}
class User < ActiveRecord::Base
  def save_and_deliver_email(mailer: UserMailer)
    if save
      mailer.welcome_email(self).deliver_later
    end
  end
end
{% endhighlight %}

Although this approach is better than using a callback, I don't like it. The model is responsible for persistence and its internal business logic. It should not concern itself with sending e-mails.

{% highlight ruby %}
class UsersController < ApplicatinController::Base
  def create
    @user = User.new(user_params)
    if @user.save_and_send_welcome_email
      redirect_to @user
    # ...
  end
end
{% endhighlight %}

Still, this is better.

### 4. Use a concern

Concerns are another tool at our disposal. And since a concern is a module, we can mix it where we need it, making it reusable.

{% highlight ruby %}
module UserRegistration
  extend ActiveSupport::Concern

  def save_user_and_send_weclome_email(user, mailer: UserMailer)
    if user.save
      mailer.welcome_email(user).deliver_later
    end
  end
end
{% endhighlight %}

We don't have to pass the user as an argument; concerns have access to instance variables in objects they're mixed into. But I consider this an anti-pattern. Passing the user as an argument defines an explicit contract between the caller and the method instead of relying on external state.

{% highlight ruby %}
class UsersController < ApplicatinController::Base
  include UserRegistration

  def create
    @user = User.new(user_params)
    if save_user_and_deliver_email(@user)
      redirect_to @user
    else
      render :new
    end
  end
end
{% endhighlight %}

## When do I use callbacks?

I use callbacks when dealing with the internal state of the object.

{% highlight ruby %}
class Invitation < ActiveRecord::Base
  before_create :set_token, :downcase_email

private

  def downcase_email
    email.downcase!
  end

  def set_token
    # A database uniqueness constraint prevents a clash
    self.token = SecureRandom.urlsafe_base64
  end
 end
{% endhighlight %}

But I look for ways to avoid them. For example, it's better to use [a setter method][2] to downcase the email.

{% highlight ruby %}
class User < ActiveRecord::Base
  before_create :set_token

  # We can check for the presence of `value`, but we don't want to.
  # It will be a blank string if a form is submitted with a blank email.
  # So we avoid a `Nil` error. Otherwise, a `Nil` error is something we want.
  def email=(value)
    super(value.downcase)
  end

  # ...
end
{% endhighlight %}

Callbacks are not intrinsically bad, and they have their uses. But they give you a lot of rope to tie yourself with. Callbacks are attractive because they make certain tasks look and feel deceptively easy. But it's worth asking yourself if there's a better way.

Can you think of otherways to avoid callbacks, and what are they?

[1]: https://en.wikipedia.org/wiki/Single_responsibility_principle
[2]: https://github.com/rails/rails/pull/19787/files

 [^1]: Because it violates SRP. It tighly couples user creation with sending emails. It obfuscates the intention of your code. It leads to undesirable side effects. It makes your code deterministic. Etc...
