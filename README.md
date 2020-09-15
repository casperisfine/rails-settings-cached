## Rails Settings Cached

This a plugin that makes managing a table of
а global key, value pairs easy. Think of it like a global Hash stored in your database,
that uses simple ActiveRecord like methods for manipulation. Keep track of any global
setting that you don't want to hard code into your rails app. You can store any kind
of object. Strings, numbers, arrays, or any object.

> 🚨 BREAK CHANGES WARNING:
> rails-settings-cached 2.x has redesigned the API, the new version will compatible with the stored setting values by an older version.
> When you want to upgrade 2.x, you must read the README again, and follow guides to change your Setting model.
> 0.x stable branch: https://github.com/huacnlee/rails-settings-cached/tree/0.x

## Status

[![Gem Version](https://badge.fury.io/rb/rails-settings-cached.svg)](https://rubygems.org/gems/rails-settings-cached) [![CI Status](https://travis-ci.org/huacnlee/rails-settings-cached.svg)](http://travis-ci.org/huacnlee/rails-settings-cached) [![codecov.io](https://codecov.io/github/huacnlee/rails-settings-cached/coverage.svg?branch=master)](https://codecov.io/github/huacnlee/rails-settings-cached?branch=master)

## Setup

Edit your Gemfile:

```bash
$ bundle add rails-settings-cached
```

Generate your settings:

```bash
$ rails g settings:install
```

If you want custom model name:

```bash
$ rails g settings:install
```

Or use a custom name:

```bash
$ rails g settings:install SiteConfig
```

You will get `app/models/setting.rb`

```rb
class Setting < RailsSettings::Base
  # cache_prefix { "v1" }
  field :app_name, default: "Rails Settings Cache Demo"
  field :host, default: "http://example.com"
  field :default_locale, default: "zh-CN"
  field :readonly_item, type: :integer, default: 100, readonly: true
  field :user_limits, type: :integer, default: 20
  field :exchange_rate, type: :float, default: 0.123
  field :admin_emails, type: :array, default: %w[admin@rubyonrails.org]
  field :captcha_enable, type: :boolean, default: true

  # Override array separator, default: /[\n,]/ split with \n or comma.
  field :tips, type: :array, separator: /[\n]+/

  field :notification_options, type: :hash, default: {
    send_all: true,
    logging: true,
    sender_email: "foo@bar.com"
  }

  # lambda default value
  field :welcome_message, type: :string, default: -> { "welcome to #{self.app_name}" }
end
```

You must use `field` method to statement the setting keys, otherwise you can't use it.

Now just put that migration in the database with:

```bash
$ rails db:migrate
```

## Usage

The syntax is easy.  First, let's create some settings to keep track of:

```ruby
irb > Setting.host
"http://example.com"
irb > Setting.host = "https://your-host.com"
irb > Setting.host
"https://your-host.com"

irb > Setting.user_limits
20
irb > Setting.user_limits = "30"
irb > Setting.user_limits
30
irb > Setting.user_limits = 45
irb > Setting.user_limits
45

irb > Setting.captcha_enable
1
irb > Setting.captcha_enable?
true
irb > Setting.captcha_enable = "0"
irb > Setting.captcha_enable
false
irb > Setting.captcha_enable = "1"
irb > Setting.captcha_enable
true
irb > Setting.captcha_enable = "false"
irb > Setting.captcha_enable
false
irb > Setting.captcha_enable = "true"
irb > Setting.captcha_enable
true
irb > Setting.captcha_enable?
true

irb > Setting.admin_emails
["admin@rubyonrails.org"]
irb > Setting.admin_emails = %w[foo@bar.com bar@dar.com]
irb > Setting.admin_emails
["foo@bar.com", "bar@dar.com"]
irb > Setting.admin_emails = "huacnlee@gmail.com,admin@admin.com\nadmin@rubyonrails.org"
irb > Setting.admin_emails
["huacnlee@gmail.com", "admin@admin.com", "admin@rubyonrails.org"]

irb > Setting.notification_options
{
  send_all: true,
  logging: true,
  sender_email: "foo@bar.com"
}
irb > Setting.notification_options = {
  sender_email: "notice@rubyonrails.org"
}
irb > Setting.notification_options
{
  sender_email: "notice@rubyonrails.org"
}
```

## Readonly field

Sometimes you may need to use Setting before Rails is initialized, for example `config/devise.rb`

```rb
Devise.setup do |config|
  if Setting.omniauth_google_client_id.present?
    config.omniauth :google_oauth2, Setting.omniauth_google_client_id, Setting.omniauth_google_client_secret
  end
end
```

In this case, you must define the `readonly` field:

```rb
class Setting < RailsSettings::Base
  # cache_prefix { "v1" }
  field :omniauth_google_client_id, default: ENV["OMNIAUTH_GOOGLE_CLIENT_ID"], readonly: true
  field :omniauth_google_client_secret, default: ENV["OMNIAUTH_GOOGLE_CLIENT_SECRET"], readonly: true
end
```

### Caching flow:

```
Setting.host -> Check Cache -> Exist - Get value of key for cache -> Return
                   |
                Fetch all key and values from DB -> Write Cache -> Get value of key for cache -> return
                   |
                Return default value or nil
```

In each Setting keys call, we will load the cache/db and save in [RequestStore](https://github.com/steveklabnik/request_store) to avoid hit cache/db.

Each key update will expire the cache, so do not add some frequent update key.

## Change cache key

Some times you may need to force update cache, now you can use `cache_prefix`

```ruby
class Setting < RailsSettings::Base
  cache_prefix { "you-prefix" }
  ...
end
```

In testing, you need add `Setting.clear_cache` for each Test case:

```rb
class ActiveSupport::TestCase
  teardown do
    Setting.clear_cache
  end
end
```

-----

## How to manage Settings in the admin interface?

If you want to create an admin interface to editing the Settings, you can try methods in following:

config/routes.rb

```rb
namespace :admin do
  resource :settings
end
```


app/controllers/admin/settings_controller.rb

```rb
module Admin
  class SettingsController < ApplicationController
    before_action :get_setting, only: [:edit, :update]

    def create
      setting_params.keys.each do |key|
        Setting.send("#{key}=", setting_params[key].strip) unless setting_params[key].nil?
      end
      redirect_to settings_path, notice: "Setting was successfully updated."
    end

    private
      def setting_params
        params.require(:setting).permit(:host, :user_limits, :admin_emails,
          :captcha_enable, :notification_options)
      end
  end
end
```

app/views/admin/settings/show.html.erb

```erb
<%= form_for(Setting.new, url: admin_settings_path) do |f| %>
  <div class="form-group">
    <label class="control-label">Host</label>
    <%= f.text_field :host, value: Setting.host, class: "form-control", placeholder: "http://localhost"  %>
  </div>

  <div class="form-group form-checkbox">
    <label>
      <%= f.check_box :captcha_enable, checked: Setting.captcha_enable? %>
      Enable/Disable Captcha
    </label>
  </div>

  <div class="form-group">
    <label class="control-label">Admin Emails</label>
    <%= f.text_area :admin_emails, value: Setting.admin_emails.join("\n"), class: "form-control" %>
  </div>

  <div class="form-group">
    <label class="control-label">Notification options</label>
    <%= f.text_area :notification_options, value: YAML.dump(Setting.notification_options), class: "form-control", style: "height: 180px;"  %>
    <div class="form-text">
      Use YAML format to config the SMTP_html
    </div>
  </div>

  <div>
    <%= f.submit 'Update Settings' %>
  </div>
<% end %>
```

## Other documents

- [Backward compatible to support 0.x scoped settings](docs/backward-compatible-to-scoped-settings.md)

## Use cases:

- [ruby-china/homeland](https://github.com/ruby-china/homeland) - master
- [forem/forem](https://github.com/forem/forem) - 2.x
- [siwapp/siwapp](https://github.com/siwapp/siwapp) - 2.x
- [aidewoode/black_candy](https://github.com/aidewoode/black_candy) - 2.x
- [thebluedoc/bluedoc](https://github.com/thebluedoc/bluedoc/blob/master/app/models/setting.rb) - 2.x
- [tootsuite/mastodon](https://github.com/tootsuite/mastodon) - 0.6.x
- [helpyio/helpy](https://github.com/helpyio/helpy) - 0.5.x


And more than [1K repositories](https://github.com/huacnlee/rails-settings-cached/network/dependents) used.
