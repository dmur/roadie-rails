# roadie-rails

> Making HTML emails comfortable for the Rails rockstars.

**Note:** This gem is still under heavy development. [This file is currently lying to you.][rdd]

This gem hooks up your Rails application with Roadie to help you generate HTML emails.

## Installation ##

Add this gem to your Gemfile and bundle.

```ruby
gem 'roadie-rails', '~> x.y.0'
```

## Usage ##

`roadie-rails` have two primary means of usage. The first on is the "Automatic usage", which does almost everything automatically. It's the easiest way to hit the ground running in order to see if `roadie` would be a good fit for your application.

As soon as you require some more "advanced" features (congratulations!), you should migrate to the "Manual usage" which entails that you do some things by yourself.

### Automatic usage ###

Include the `Roadie::Rails::Automatic` module inside your mailer. Roadie will do its magic when you try to deliver the message:

```ruby
class NewsletterMailer < ActionMailer::Base
  include Roadie::Rails::Automatic

  def user_newsletter(user)
    mail to: user.email, subject: subject_for_user(user)
  end

  private
  def subject_for_user(user)
    I18n.translate 'emails.user_newsletter.subject', name: user.name
  end
end

# email has the original body; Roadie has not been invoked yet
email = NewsletterMailer.user_newsletter(User.first)

# This triggers Roadie inlining and will deliver the email with inlined styles
email.deliver
```

This way does not leave you with many options in case you want to selectively inline certain emails, or just in certain cases. If you need the extra flexibility, look at the "Manual usage" below.

### Manual usage ###

Include the `Roadie::Rails::Mailer` module inside your `ActionMailer` and call `roadie_mail` with the same options that you would pass to `mail`.

```ruby
class NewsletterMailer < ActionMailer::Base
  include Roadie::Rails::Mailer

  def user_newsletter(user)
    roadie_mail to: user.email, subject: subject_for_user(user)
  end

  private
  def subject_for_user(user)
    I18n.translate 'emails.user_newsletter.subject', name: user.name
  end
end
```

This will inline the stylesheets right away, which sadly decreases performance for your tests where you might only want to inline in one of them. The upside is that you can selectively inline yourself.

```ruby
class NewsletterMailer < ActionMailer::Base
  include Roadie::Rails::Mailer

  def subscriber_newsletter(subscriber, options = {})
    use_roadie = options.fetch :use_roadie, true
    mail_factory(use_roadie, normal_mail_options)
  end

  private
  def mail_factory(use_roadie, options)
    if use_roadie
      roadie_mail options
    else
      mail options
    end
  end
end

# tests
describe NewsletterMailer do
  it "is emailed to the subscriber's email" do
    email = NewsletterMailer.subscriber_newsletter(subscriber, use_roadie: false)
    email.to.should == subscriber.email
  end

  it "inlines the emails by default" do
    email = NewsletterMailer.subscriber_newsletter(subscriber)
    email.should be_good_and_cool_and_all_that_jazz
  end
end
```

Or, perhaps by doing this:

```ruby
describe YourMailer do
  describe "email contents" do
    before do
      # Redirect all roadie mail calls to the normal mail method
      YourMailer.stub(:roadie_mail) { |*args, &block| YourMailer.mail(*args, &block) }
    end
    # ...
  end

  describe "inlined email contents" do
    # ...
  end
end
```

### Configuration ###

Roadie can be configured in three places, depending on how specific you want to be:

1. `Rails.application.config.roadie` (global, static).
2. `YourMailer#roadie_options` (mailer, dynamic).
3. Second argument to the `roadie_mail` (mail, specific and custom).

Each level will be merged with the level before it.

Only the first two methods are available to you if you use the `Automatic` module.

```ruby
# config/environments/production.rb
config.roadie.url_options = {host: "my-app.com", scheme: "https"}

# app/mailer/my_mailer.rb
class MyMailer
  include Roadie::Rails::Mailer

  protected
  def roadie_options
    {host: Product.current.host}
  end
end

# app/mailer/my_other_mailer.rb
class MyOtherMailer
  include Roadie::Rails::Mailer

  def some_mail(user)
    roadie_email {to: "foo@example.com"}, roadie_options_for(user)
  end

  private
  def roadie_options_for(user)
    {host: user.host, scheme: (user.use_ssl? ? 'https' : 'http')}
  end
end
```

### Templates ###

Use normal `stylesheet_link_tag` and `foo_path` methods when generating your email and Roadie will look for the precompiled files on your filesystem, or by asking the asset pipeline to compile the files for you if it cannot be found.

### Previewing ###

You can create a controller that gets the email and then renders the body from it.

```ruby
class Admin::EmailsController < AdminController
  def user_newsletter
    render_email NewsletterMailer.user_newsletter(current_user)
  end

  def subscriber_newsletter
    render_email NewsletterMailer.subscriber_newsletter(Subscriber.first || Subscriber.new)
  end

  private
  def render_email(email)
    respond_to do |format|
      format.html { render html: email.html_body.decoded }
      format.text { render text: email.text_body.decoded }
    end
  end
end
```

## Build status ##

Tested with [Travis CI](http://travis-ci.org) using [almost all combinations of](http://travis-ci.org/#!/Mange/roadie-rails):

* Ruby:
  * MRI 1.9.3
  * MRI 2.0.0
* Rails
  * 3.0
  * 3.1
  * 3.2
  * 4.0

Let me know if you want any other combination supported officially.

### Versioning ###

This project follows [Semantic Versioning][semver]. The 0.x branch is considered unstable.

## Documentation ##

* [Online documentation for gem](http://rubydoc.info/gems/roadie-rails/frames)
* [Online documentation for master](http://rubydoc.info/github/Mange/roadie-rails/master/frames)
* [Changelog](https://github.com/Mange/roadie-rails/blob/master/Changelog.md)

## Running specs ##

Start by setting up your machine, then you can run the specs like normal:

```bash
./setup.sh install
rake spec
```

## License ##

(The MIT License)

Copyright © 2013

* [Magnus Bergmark](https://github.com/Mange) <magnus.bergmark@gmail.com>

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the ‘Software’), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED ‘AS IS’, WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.


[roadie]: http://rubygems.org/gems/roadie
[semver]: http://semver.org/
[rdd]: http://tom.preston-werner.com/2010/08/23/readme-driven-development.html
