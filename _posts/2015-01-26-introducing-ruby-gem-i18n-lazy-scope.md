---
layout: post
title: Introducing Ruby Gem I18nLazyScope
---

Lately, I've been working on some projects that require internationalization. I find it annoying to have to type `namespace.key` each time I translate something. I *can* use [lazy lookup][2]--`t('.key')`--but it defaults to `locale.resource.key`, which is not always desirable.

Wouldn't it be nice if I could use lazy lookup and have granular control over the namespace? So, for example, if I typed `t(:key)` in `users#show` the scope would default to `en.views.users.show.key`.

So, to have my cake and eat it, I wrote [I18nLazyLookup][1]. **It lets you use lazy lookup with custom scopes**.

## How does it work?

The library inserts a **customisable** namespace in the scope between the `locale` and the `resource` keys. The following table shows the differences in bold between I18nLazyLookup and [i18n-rails][3].

| ------------|--------------------------------------------------------------|
| Controllers | `locale.`**`controllers`**`.controller_name.action_name.key` |
| Mailers     | `locale.`**`mailers`**`.mailer_name.action_name.key`         |
| Views       | `locale.`**`views`**`.template_or_partial_path.key`          |

## Tell me more

Say you are in `users_controller#show` and you want to flash a welcome message.

{% highlight ruby %}
def create
  if @user.save
    redirect_to @user, notice: t('welcome_msg')
  end
end
{% endhighlight %}

The scope defaults to `en.welcome_msg`. This is not a good way to structure translations as they'll soon become messy and unmanagable. It's better to scope them under a namespace.

{% highlight yaml %}
en:
  controllers:
    users:
      create:
        welcome_msg: "You are such a star!"
{% endhighlight %}

{% highlight ruby %}
redirect_to @user, notice: t('controllers.users.create.welcome_msg')
{% endhighlight %}

But that's a lot to type. It would be nice if we could write `t('.welcome_msg')`, but we can't because [rails-I18n][3] scopes the translation to `en.users.create.welcome_msg`. Our *controller* namespace is missing.

And that's where *I18nLazyLookup* comes in. **It lets you use lazy lookup with custom namespaces**. This makes changing the structure of translations easy.

{% highlight ruby %}
redirect_to @user, notice: t_scoped(:welcome_msg)
{% endhighlight %}

You can customise the namespace using an initializer if the defaults don't work for you.

{% highlight ruby %}
# app/config/initializers/i18n_lazy_lookup.rb
I18nLazyScope.configure do |config|
  config.action_controller_scope = [:custom, :scope]
  config.action_mailer_scope     = [:custom, :scope]
  config.action_view_scope       = [:custom, :scope]
end
{% endhighlight %}

The namespace will now resolve to the following:

|-------------|-------------------------------------------------------|
| Controllers | `locale.custom.scope.controller_name.action_name.key` |
| Mailers     | `locale.my.custom.scope.mailer_name.action_name.key`  |
| Views       | `locale.my.custom.scope.template_or_partial_path.key` |

Check out the [project repo on Github][2] for more details.

[1]: https://github.com/abitdodgy/i18n_lazy_scope
[2]: http://guides.rubyonrails.org/i18n.html#lazy-lookup
[3]: https://github.com/svenfuchs/rails-i18n
