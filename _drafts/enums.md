---
layout: post
title: Enums
---

We have used ActiveRecord's enumerated type extensively ever since it was introduced in Rails 4.1. Although it's getting a facelift in Rails 5, there are still plenty of gotchas to be aware of when using it.

[Up first][1] is [`update_all`][2], a low level method that builds a single SQL UPDATE statement and sends it directly to the database.

{% highlight ruby %}
class Post < ActiveRecord::Base
  enum status: { draft: 1, published: 2 }
end

Post.update_all(status: :published)
{% endhighlight %}

It shouldn't come as a surprise that using a symbol with `update_all` does not work. It's because `update_all` is designed to work with primitive types, and it does not afford special treatment for enums. This is why it doesn't run callbacks or validations. For `update_all` to work, any values passed to it must go through ActiveRecord's typecasting behaviour. In other words, `:pulished` has to be translated to its raw value, the integer part of the element. But this isn't hard to do.

{% highlight ruby %}
Post.update_all(status: Post.statuses[:published])
{% endhighlight %}

Next up are `where` query methods. Arguments passed to `where` *do* get typecast by ActiveRecord, but not in a way you might expect.

{% highlight ruby %}
Post.where(status: 'draft')
{% endhighlight %}

I expected this to work the first time tried it, but it fails silently. Under the hood, Rails calls `to_i` on 'draft', which returns 0, and this is what gets used in the generated query. This happens when `where` typecasts the value for integer-type columns. If I used a symbol, we would see `nil` instead of 0[1^]. When using `where` ActiveRecord does not know that `status` is defined as an enum, and treats it according to its schema definition, an integer.

{% highlight sql %}
SELECT "posts".* FROM "posts" WHERE "posts"."status" = $1  [["status", 0]]
{% endhighlight %}

But in this case, one can and should use the generated scope `Post.draft`, even when querying through an association.

{% highlight ruby %}
class User < ActiveRecord::Base
  has_many :published_posts, -> {
    Post.published
  }, class_name: 'Post'
end
{% endhighlight %}

However, there are times when using `where` is necessary. For example, if I have a table of posts I want to filter by status by passing its value as a parameter.

{% highlight slim %}
= link_to 'Published', params.merge(status: :published)
{% endhighlight %}

We can use the same approach we used with `update_all`, but it's better to create a scope that handles the type conversion for us.

class Post < ActiveRecord::Base
  enum status: { draft: 1, published: 2 }

  scope :status, ->(status) {
    where(status: statuses[status])
  }
end

I should mention that this behaviour has been made consistent in Rails 5. Going forward `where` will recognise `status` as an enum, and will do the conversion for us.

Another thing I've seen frequently is enum type columns being defined as string type. There are many ways to implement enums, and you can use types other than integer, but ActiveRecord only supports integer. The trouble is, if you use something else you'll get silent failures instead of strings.

```
 add_column :posts, :string, :status
```

While nothing stipulates that enumerated types should be stored as integers, unfortunately ActiveRecord enums requires that you do. And if you store as strings, thing will tick along nicely until you get a nasty surprise.

Using enums in forms. https://github.com/rails/rails/issues/16185

1. This think `update_all` will detect and typcast enums.
2. This think where(enum: :value) will detect the enum.
3. T


  [1]: https://github.com/rails/rails/issues/17242
  [2]: http://apidock.com/rails/v4.0.2/ActiveRecord/Relation/update_all
  [3]:

  [^1]: This is the same reason why `update_all` does not run validations and callbacks.
  [^2]: Although Symbol does not define `to_i`, ActiveModel [rescues the error](https://github.com/rails/rails/blob/master/activemodel/lib/active_model/type/integer.rb#L43).
