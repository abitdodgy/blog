---
layout: post
title: The Dummies Guide to Using Enums in Rails
---

I've used ActiveRecord's built in enumerated type a lot since it was introduced in Rails 4.1. Although it's getting a facelift in Rails 5, there are still plenty of gotchas to look out for.

Up first is `update_all`, a low level method that builds a single SQL UPDATE statement and sends it directly to the database.

{% highlight ruby %}
class Post < ActiveRecord::Base
  enum status: { draft: 0, published: 1 }
end

Post.update_all(status: :published)
{% endhighlight %}

It shouldn't come as a surprise that using a symbol with `update_all` does not work. It's designed to work with primitive types and doesn't typecast values. It doesn't run callbacks or validations either. For `update_all` to work, any values it's passed must go through ActiveRecord's typecasting behaviour.

In other words, `:published` has to be translated to its raw value, the integer part of the element. This isn't hard to do.

{% highlight ruby %}
Post.update_all(status: Post.statuses[:published])
{% endhighlight %}

Next up are `where` query methods. Arguments passed to `where` *do* get typecast by ActiveRecord, but not in a way you might expect.

{% highlight ruby %}
Post.where(status: 'draft')
{% endhighlight %}

The above code fails silently. Under the hood, Rails calls `to_i` on 'draft', and its return value, `0`, gets used in the generated query. This happens when `where` typecasts the value for integer-type columns. If I use a symbol, we would see `nil` instead of 0[^1].

When using `where`, ActiveRecord does not know that `status` is defined as an enum, and treats it according to its schema definition, an integer.

{% highlight sql %}
SELECT "posts".* FROM "posts" WHERE "posts"."status" = $1  [["status", 0]]
{% endhighlight %}

In this case, however, one can and should use the generated scope `Post.draft`, even when querying through an association.

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

{% highlight ruby %}
class Post < ActiveRecord::Base
  enum status: { draft: 0, published: 1 }

  scope :status, ->(status) {
    where(status: statuses[status])
  }
end
{% endhighlight %}

I should mention that this behaviour [has been made consistent][2] in Rails 5. Going forward `where` will recognise `status` as an enum, and will do the conversion for us.

I frequently see enum columns defined with string type. There are many ways to implement enums, and you can use types other than integer, but ActiveRecord only supports integer. If you use strings you'll get silent failures and nasty surprises.


```
 add_column :posts, :string, :status, default: 0
```

```ruby
 p = Post.create!
 p.draft?
 #=> true
 p.status
 #=> 0
 p.published!
 p.published?
 #=> true
 p.status
 #=> 0 # BOOM!
```


Enums are misused frequently in controllers and views. I recently [answered a question][1] on Stack Overflow that had this code:

```erb
<%= link_to "Waiting", property_path(property, {:status => 'Waiting for Response'}), method: :patch) %>
<%= link_to "Registered", property_path(property, {:status => 'Registered'}), method: :patch) %>
# Two more of these...
```

Not only is this needlessly verbose, but you must remember to change the view code each time you add a new status. It's best to generate these links automatically.

```erb
<% Property.statuses.each_key do |status| %>
  <%= link_to status, property_path(property, { status: status }), method: :patch %>
<% end %>
```

The controller was in worse shape.

```ruby
def approve
  if params[:status]== 'Registered'
     @property.update_attributes(:status => 1)
     redirect_to :back, flash: {notice: "Property Registered."}
  elsif params[:status]== 'Waiting for Response'
     @property.update_attributes(:status => 3)
     redirect_to :back, flash: {notice: "Waiting for Response"}
  elsif
    # and more...
  end
end
```

Given the new code, it could be condensed into this.

```ruby
def approve
  @property.update!(status: params[:status])
  redirect_to :back, notice: t(".#{params[:status]}")
end
```

Which brings me to my next point. Don't validate enums; you don't have to. Rails does it automatically for you.

```ruby
post.update(status: :rubbish)
#=> ArgumentError: 'rubbish' is not a valid status
```

And use the handy generated methods when you can: `post.published!` instead of `post.update(status: :published)`.

Finally, use a database index. You will likely need to filter results by an enum value.


```
add_index :posts, :status
```

I hope you found this post useful.

  [1]: http://stackoverflow.com/a/35595700/276959
  [2]: https://github.com/rails/rails/commit/c51f9b61ce1e167f5f58f07441adcfa117694301
  [^1]: Although Symbol does not define `to_i`, ActiveModel [rescues the error](https://github.com/rails/rails/blob/master/activemodel/lib/active_model/type/integer.rb#L43).
