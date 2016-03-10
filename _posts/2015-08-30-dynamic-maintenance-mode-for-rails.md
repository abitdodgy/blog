---
layout: post
title: Adding a dynamic maintenance mode you Rails app
---

In an ideal world, you would update your application with zero downtime, and your users would be rewarded with snazzy new features without noticing how they got there. Of course, this is not always the case. There are times when downtime is inevitable. Having a proper and scalable maintenance strategy to deal with downtime is necessary, especially when making changes to the database. The following post describes a strategy we recently used.

We had two requirements:

1. Allow *some* users to access the application while it's undergoing maintenance.
2. Serve a custom, internationalized template.

There are several ways to implement a maintenance strategy, but they typically work by bypassing requests to the application server and serving a static HTML page. Heroku has a built-in [maintenance feature][1] that works in a similar way. But it meets none of our requirements because it bypasses the app entirely.

In our case, requests must hit the application so that it can determine whether to allow access, and what language to serve to the user.

We implemented this easily using `ENV` variables, a controller action, and Rails' built in I18n support. Here's how we did it.

## Remembering the user's location

For better usability, we should remember the user's intended location before redirecting him or her away. This way, when the app exits maintenance, we can redirect users back to the page they originally wanted. This is sometimes referred to as *friendly forwarding*. Friendly forwarding is a feature we can reuse in other places, so it's a good idea to implement it in a concern[^2] and mix it into `application_controller.rb`.

{% highlight ruby %}
# app/controllers/concerns/friendly_forwarding.rb
def redirect_back_or(default, options = {})
  location = session.delete(:forwarding_url) || default
  redirect_to location, options
end

def store_location
  session[:forwarding_url] = request.url if request.get?
end
{% endhighlight %}

When called, `store_location` will save the request URL in the session, but only if it's a GET request [^1]. The `redirect_back_or` method takes a default route to use if no forwarding URL is present in the session, and an options hash. This way we can forward the same options that [`redirect_to`][2] accepts, including flash messages.

## Maintenance mode

To implement maintenance mode, we use a concern that we mix into `application_controller.rb`.

{% highlight ruby %}
module MaintenanceMode
  extend ActiveSupport::Concern

  included do
    before_action :handle_maintenance
  end

private

  def handle_maintenance
    if maintenance_mode_enabled?
      unless remote_address_whitelisted?
        store_location
        redirect_to maintenance_path
      end
    end
  end

  def maintenance_mode_enabled?
    ENV['MAINTENANCE_MODE'].present?
  end

  def maintenance_mode_disabled?
    !maintenance_mode_enabled?
  end

  def remote_address_whitelisted?
    maintainer_ips.split(',').include?(request.remote_ip)
  end

  def maintainer_ips
    ENV['MAINTAINER_IPS'] || String.new
  end
end
{% endhighlight %}

Unless the current IP address is whitelisted, the `MaintenanceMode` concern redirects to the maintenance page if the mode is enabled.

{% highlight ruby %}
class ApplicationController < ActionController::Base
  include FriendlyForwarding
  include MaintenanceMode
end
{% endhighlight %}

All that remains is to add a controller that renders the maintenance page, a route, and some tests.

{% highlight ruby %}
# config/routes.rb
get :maintenance, to: 'maintenance#show'
{% endhighlight %}

{% highlight ruby %}
class MaintenanceController < ApplicationController
  skip_before_action :handle_maintenance

  def show
    render :show, status: 503
  end
end
{% endhighlight %}

The `skip_before_action` ensures that we don't check for maintenance mode when we are viewing the maintenance page itself. This stops the application from going into an infinite loop. We have to explicitly use `render` to set the HTTP status code to *503*, which stands for `:service_unavailable`. The default status code in Rails is *200*, or `:success`.

We can stop now, but for better usability we should redirect away from the maintenance page and back to the app if the mode is disabled. It's a small change, but it's important--especially if your maintenance page has no navigation--because many users will refresh the page hoping the app will appear again. If we don't redirect the user back, he or she could be stuck until frustrated enough to type the app's root URL in the address bar. Plus, we made the effort to add friendly forwarding, and unless we make this change we will not have a chance to use it here.

For `maintenance_controller#show` we should redirect back to the application under two conditions:

1. If maintenance mode is disabled.
2. If the IP address is whitelisted.

We can use a `before_action` to add this extra functionality.

{% highlight ruby %}
class MaintenanceController < ApplicationController
  # ...
  before_action :redirect_if_maintenance_disabled
  # ...

private

  def redirect_if_maintenance_disabled
    if maintenance_mode_disabled? || remote_address_whitelisted?
      redirect_back_or root_path
    end
  end
end
{% endhighlight %}

## Using it

To enter maintenance mode, we set an `ENV` variable on Heroku.

{% highlight text %}
heroku config:set MAINTENANCE_MODE=enabled
{% endhighlight %}

From the app's perspective, it doesn't really matter what the value of `MAINTENANCE_MODE` is (or its name, for that matter), so *enabled* serves for clarity. Our logic checks for the presence of the variable, not its value.

To allow access to an IP address, we set another `ENV` variable. Its value should be a comma-delimited list of IP addresses for whom we want to enable access.

{% highlight text %}
heroku config:set MAINTAINER_IPS=1.2.3.4,9.8.7.6
{% endhighlight %}

And finally, to exit maintenance mode, we unset the variables.

{% highlight text %}
heroku config:unset MAINTENANCE_MODE
{% endhighlight %}

## Testing it

We want to test a number of conditions:

1. The app redirects to the maintenance page when the mode is enabled.
2. The app does not redirect to the maintenance page when the mode is disabled, or if the current IP address is whitelisted.
3. The app will redirect away from the maintenance page if the mode is disabled, or if the current IP address is whitelisted.
4. The app will redirect back to the user's intended location once maintenance mode is disabled.

{% highlight ruby %}
require 'test_helper'

class MaintenanceModeTest < ActionDispatch::IntegrationTest
  teardown do
    ENV.delete('MAINTENANCE_MODE')
    ENV.delete('MAINTAINER_IPS')
  end

  test "does not redirect to maintenance page if mode is disabled" do
    get login_path
    assert_response :success
  end

  test "redirects to maintenance page if mode is enabled" do
    ENV['MAINTENANCE_MODE'] = 'enabled'
    get login_path
    assert_redirected_to maintenance_path
    follow_redirect!
    assert_response :service_unavailable
  end

  test "does not redirect to maintenance page if mode is enabled and IP is whitelisted" do
    ENV['MAINTENANCE_MODE'] = 'enabled'
    ENV['MAINTAINER_IPS'] = '1.2.3.4'
    get login_path, {}, { 'REMOTE_ADDR' => '1.2.3.4' }
    assert_response :success
  end

  test "redirects away from maintenance page when mode is disabled" do
    get maintenance_path
    assert_redirected_to root_path
    follow_redirect!
    assert_response :success
  end

  test "redirects away from maintenance page when mode is enabled and IP is whitelisted" do
    ENV['MAINTENANCE_MODE'] = 'enabled'
    ENV['MAINTAINER_IPS'] = '1.2.3.4'
    get maintenance_path, {}, { 'REMOTE_ADDR' => '1.2.3.4' }
    assert_redirected_to root_path
  end

  test "redirects back to requested page when mode is disabled" do
    ENV['MAINTENANCE_MODE'] = 'enabled'
    get login_path
    assert_redirected_to maintenance_path
    ENV['MAINTENANCE_MODE'] = nil
    get maintenance_path
    assert_redirected_to login_path
  end
end
{% endhighlight %}

And there it is, a flexible and dynamic maintenance mode for your Rails application.

## Caveats

This feature is useful when you need to restrict access but keep your app running. Otherwise, you might want to implement a maintenance page at the DNS or [web server][3] level, or use Heroku's built in solution.

[1]: https://devcenter.heroku.com/articles/maintenance-mode
[2]: http://api.rubyonrails.org/classes/ActionController/Redirecting.html#method-i-redirect_to
[3]: https://viget.com/extend/server-maintenance-mode-for-rails-capistrano-and-apache2

### Footnotes

[^1]: There actually is [an HTTP specification](https://www.w3.org/Protocols/rfc2616/rfc2616-sec10.htmlg) for redirecting POST requests. But it appears that most frameworks don't handle this requirement adequatly, so we generally don't want to redirect back if the user was posting a form. For more on this topic, see [this article on Programmers](http://programmers.stackexchange.com/questions/99894/why-doesnt-http-have-post-redirect).
[^2]: For example, when redirecting to the login page after a user requests a protected resource.
