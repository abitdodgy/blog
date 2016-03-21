---
layout: post
title: Four Alternatives to Using ActiveRecord Callbacks and Observers
---

Callbacks suck, and many experienced Rails programmers will tell you so. It took me a couple of years to finally understand why, but I did eventually, and I have rarely used them since.

If we should avoid callbacks, what should we use instead?

First, let's look at a naive scenario. You have probably seen a similar example before.

{% highlight ruby %}
class User < ActiveRecord::Base
  after_create :send_welcome_email

private

  def send_welcome_email
    UserMailer.welcome_email(self).deliver_later
  end
end
{% endhighlight %}

A user registers and he or she is sent a welcome e-mail. We know that in most cases this is a bad[^1] idea. So how can we make this better?

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

This is better because our code is explicit and has no unexpected side effects. Calling `@user.save` saves the user instance, and that's it. Instead, the e-mail is sent explicitly.

In many cases this is fine, but there is a downside, and that is our code is not reusable. We might have to send an e-mail after user registration in other places (for example, from an admin panel or through an API call), and hardcoding this logic in the controller means we have to repeat ourselves.

### 2. Use a controller concern

Concerns are just Ruby modules we can use to share common functionality between different models and controllers. We can extract the bit of code we want to reuse and encapsulate it in a method in our module.

{% highlight ruby %}
module UserRegistration
  extend ActiveSupport::Concern

  def save_user_and_send_weclome_email(user, mailer: UserMailer)
    if user.save?
      mailer.welcome_email(user).deliver_later
      user
    else
      false
    end
  end
end
{% endhighlight %}

It's not necessary to pass the user as an argument; methods in concerns have access to the state of the objects they're mixed into. But I prefer to be more explicit than rely on state. Passing the user as an argument defines an explicit contract between the caller and the callee.

Here's how we might use it.

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

Concerns give us code reuse, but at the cost of increased abstraction. For example, you can't immediately see what `save_user_and_deliver_email` does, and to find out you must open another file. It is also not immediately obvious where else this method is being used.

### 3. Use a service object

Service objects encapsulate bits of reusable functionality in their own class. Because they're not modules, they don't bloat other classes or suffer from the same issues that concerns do. They are easy to reason about, and a joy to test.

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
      user
    else
      false
    end
  end
end
{% endhighlight %}

We can reuse this service object from anywhere--the console, a rake task, or another controller--free from side effects.

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

I find service objects are easy to extend. Let's say that, besides sending an e-mail, we need to create an activity item for a feed. This is an easy change to make.

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

### 4. Use a model method

If you don't find the benefits of using concerns and service objects compelling, you can replace callbacks with vanilla methods in your model.

{% highlight ruby %}
class User < ActiveRecord::Base
  def save_and_deliver_email(mailer: UserMailer)
    if save
      mailer.welcome_email(self).deliver_later
      self
    else
      false
    end
  end
end
{% endhighlight %}

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

I'm not a fan of this approach, even if it's an improvement on callbacks. In my opinion, an ActiveRecord model should be responsible for its persistence and internal business logic only. It should not concern itself with sending e-mails.

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

But I look for ways to avoid them. For example, it's better to use [a setter method][2] to downcase the email[^2].

{% highlight ruby %}
class User < ActiveRecord::Base
  before_create :set_token

  def email=(value)
    super(value.downcase)
  end

  # ...
end
{% endhighlight %}

Callbacks are not intrinsically bad, and they have their uses. But they give you a lot of rope to tie yourself with. They are attractive because they make certain tasks look and feel deceptively easy. It's always worth asking yourself if there's a better way.

[1]: https://en.wikipedia.org/wiki/Single_responsibility_principle
[2]: https://github.com/rails/rails/pull/19787/files

[^1]: Because it violates SRP. It tightly couples user creation with sending emails. It obfuscates the intention of your code. It leads to undesirable side effects. It makes your code deterministic. Etc...
[^2]: We *can* check for the presence of `value`, but we shouldn't; `value` will be an empty string when a form is submitted with a blank email. If we get errors because of `value` being `nil`, it's probably a bug in the application, and we want it to fail.
