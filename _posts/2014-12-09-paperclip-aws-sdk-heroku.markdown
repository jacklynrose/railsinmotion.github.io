---
layout: post
title: Using paperclip & aws-sdk to store files on Heroku
---

While teaching my students at [General Assembly](http://generalassemb.ly/) about file uploads, we decided to go with [paperclip](https://github.com/thoughtbot/paperclip) and [aws-sdk](https://github.com/aws/aws-sdk-ruby) instead of [carrierwave](https://github.com/carrierwaveuploader/carrierwave) and [fog](https://github.com/fog/fog) like I'm used to, this lead to some painful issues as I forgot exactly what the cofiguration settings were once we were trying to integrate with [Amazon S3](http://aws.amazon.com/).

Right now as I'm writing this, I'm actually sitting in the class as they work on their projects and going through the process of sorting it out. This is why I'm writing a blog post for this, so I can write out the steps as I do it and give it to my students, hopefully it's handy for anyone else reading it too.

### 1. Setting up the project

This guide is going to assume that you already have heroku set up on your computer and that you have an Amazon account.

First things first is we need an application to test this out with, for this example we'll just create a basic project, scaffold an article model, and then from there we can actually work with there gems.

```
rails new blog
cd blog
rails g scaffold Article title content:text
rake db:migrate
```

With those four commands we will now have a basic blog set up, standard scaffolding stuff. We then need to get this up and working on Heroku.

Before we can push to heroku we need to change our Gemfile.

```ruby
gem 'sqlite3', group: [:development, :test]
gem 'pg', group: :production
gem 'rails_12factor', group: :production
```

And of course when we change our `Gemfile` we need to run `bundle`. After that we're ready to push up to Heroku.

```
git init
heroku create
git add .
git commit -am "Initial Commit with basic Article scaffolding"
git push heroku master
heroku run rake db:migrate
```

We can run the above commands from within the directory to initialize the git repository, create the heroku app, then commit and push our code up to heroku. If you run `heroku open` and add `/articles` to the end of the URL, you should see the scaffolded index page.

![Basic scaffolded app](/images/figure-1.1.png)

### 2. Basic paperclip

With a basic application working we can now start working with paperclip. I will note that if you took the end of this section (section 2) and pushed it to heroku then *"it will work"*. The reason it *"works"* is because Heroku has a filesystem you can *"write"* to, but once you restart the server or push new code all those things will be gone. To fix that, section 3 will be setting up the S3 settings.

Starting off we need to install the gem.

```ruby
gem 'paperclip'
```

Run `bundle` and then we can edit our article model.


```ruby
class Article < ActiveRecord::Base
  has_attached_file :photo
end
```

This is the basic method for giving your model an attached file, in the real world you likely want to have some special styles like a thumbnail version of the photo.

```ruby
class Article < ActiveRecord::Base
  has_attached_file :photo, styles: {
    thumb: "150x150>"
  }
end
```

As I learnt though later on, apparently paperclip requires that you have some kind of validation, I learnt that lesson in front of all 12 of my students. Don't judge, like I said, I don't usually use paperclip.

```ruby
class Article < ActiveRecord::Base
  has_attached_file :photo, styles: {
    thumb: "150x150>"
  }
  validates_attachment_content_type :photo, :content_type => /\Aimage\/.*\Z/
end
```

What we've now done is to validate that the content type of the `photo` on our model is some kind of image. FYI, you can have multiple styles, just add another key/value pair in the styles hash.

Next is the migration to get this working.

```
rails g migration AddPhotoToArticles
```

Open up that migration and change it to have these two methods.

```ruby
  def up
    add_attachment :articles, :photo
  end

  def down
    remove_attachment :articles, :photo
  end
```

Paperclip has added two new methods we can use in our migrations called `add_attachment` and `remove_attachment`, and we're going to use these, passing in the table name and the name we've given our attachment (`photo`).

Run `rake db:migrate` and with that you'll have your model set up, now we can focus on the view and that part I **always** forget, which is the strong parameters.

In your `app/views/articles/_form.html.erb` you need to enable multipart and add a new file field.

```erb
<!-- add the mutlipart bit -->
<%= form_for(@article, html: { multipart: true }) do |f| %>
  <% if @article.errors.any? %>
    <div id="error_explanation">
      <h2><%= pluralize(@article.errors.count, "error") %> prohibited this article from being saved:</h2>

      <ul>
      <% @article.errors.full_messages.each do |message| %>
        <li><%= message %></li>
      <% end %>
      </ul>
    </div>
  <% end %>

  <div class="field">
    <%= f.label :title %><br>
    <%= f.text_field :title %>
  </div>
  <div class="field">
    <%= f.label :content %><br>
    <%= f.text_area :content %>
  </div>

  <!-- add this field -->
  <div class="field">
    <%= f.label :photo %><br>
    <%= f.field_field :photo %>
  </div>

  <div class="actions">
    <%= f.submit %>
  </div>
<% end %>
```

After you've added that file field to your form, **do not** forget about the strong params.

```
# articles_controller.rb
  def article_params
    params.require(:article).permit(:title, :content, :photo)
  end
```

The very last step is to just display the image. If you want to display the originally uploaded image, you can use this code.

```erb
<%= image_tag @article.photo.url %>
```

But to actually display the thumbnail...

```erb
<%= image_tag @article.photo.url(:thumb) %>
```

Now you're ready to go!

```
rails s
```

### 3. The horrible part - S3 integration

This is where things can go wrong really easily. Honestly I would actually suggest using [fog](https://github.com/fog/fog) here, because then you're not tied into one particular vendor as much, plus you can mock this stuff out easier in your tests. For this tutorial though, we'll be using the [aws-sdk](https://github.com/aws/aws-sdk-ruby) gem.

```ruby
gem 'aws-sdk'
```

Run `bundle` to install that gem, and now we can get into making sure we have everything we need before we get into coding.

I personally prefer to **not** use S3 in my development and test environments, and only use it in production, so we're going to be following that path. Before we code we need to set some stuff up through the AWS management panel.

Go to [aws.amazon.com](http://aws.amazon.com/) and then sign in by hovering over "My Account" and clicking on "AWS Management Console", then sign in with your Amazon account.

The first thing we have to do is create a bucket, so click on S3 in the massive list of services. Once that page loads, click "Create Bucket", and fill in the form. Once the bucket has been created, click on "Properties".

In the "Properties" panel is a "Permissions" section. You need to add a new permision, choose "Everyone" from the dropdown, and check "List". After making that change, click "Save". What this has done is allow anyone on the internet to access this file if they have the link, which they will eventually because we'll be putting that into our HTML to display our images.

![Permission panel](/images/figure-3.1.png)

Next is creating the AWS user that will have permissions to write files to S3.

Under your name in the navigation bar, when you hover over it there is a menu option for "Security Credentials", click on that. Once that page has come up, click on "Users" in the left sidebar.

Create a new user, enter a username for it (it really doesn't matter what it is) and make sure "Generate an access key for each user" is checked.

After you create that user, a new page will come up, and there is a link "Show User Security Credentials", click on it and take note of the "Access Key ID" and "Secret Access Key", you will need these to set up S3 integration, downloading them is heavily suggested as you won't have access to them again.

The user has been created but it currently has access to absolutely none of your Amazon services. To give a user access to a service, you need to attach a security policy. Click "close" and then when the list of users comes back up, click on the one you created.

Scroll down to "Permissions" and click the big blue "Attach User Policy" button. You will be shown a massive list in a box of all the different kinds of permissions you can give the user, you want to find "Amazon S3 Full Access" and press"Select" on it. It will preview the policy, just click the "Apply Policy" button. Now this user has access to write to S3.

![AWS User policy](/images/figure-3.2.png)

That's everything we need to set up, now we need to configure our app.

In the `production.rb` file we need to set up some configuration for paperclip.

```ruby
config.paperclip_defaults = {
  storage: :s3,
  s3_credentials: {
    bucket: ENV["AWS_BUCKET"],
    access_key_id: ENV["AWS_ACCESS_KEY_ID"],
    secret_access_key: ENV["AWS_SECRET_ACCESS_KEY"]
  }
}
```

This is going to load environment variables to set the settings for the bucket, access key id, and secret access key. **DO NOT EVER PUT THE VALUES IN DIRECTLY IN THE CODE!!!**

With this configuration done, you now need to set these environment variables on heroku, to do that:

```
heroku config:add AWS_BUCKET="XXXXXXX"
heroku config:add AWS_ACCESS_KEY_ID="XXXXXX"
heroku config:add AWS_SECRET_ACCESS_KEY="XXXXXXX"
```

Don't just copy and paste these and press enter, instead copy them and replace the "XXXXXXX" values with the bucket, access key id, and secret access key respectively.

If you want to set these locally too, you should add this code to your `~/.bash_profile` or `~/.zshrc` or similar file.

```
export AWS_BUCKET="XXXXXXX"
export AWS_ACCESS_KEY_ID="XXXXXX"
export AWS_SECRET_ACCESS_KEY="XXXXXXX"
```

You also need to change the settings for your `development.rb` file a little too by adding the `path` and `url` options.

```ruby
config.paperclip_defaults = {
  storage: :s3,
  s3_credentials: {
    bucket: ENV["AWS_BUCKET"],
    access_key_id: ENV["AWS_ACCESS_KEY_ID"],
    secret_access_key: ENV["AWS_SECRET_ACCESS_KEY"]
  },
  url: ":s3_domain_url",
  path: "/:class/:attachment/:id_partition/:style/:filename"
}
```

With those configuration variables created on Heroku now...

```
git add .
git commit -m "Added paperclip"
git push heroku master
heroku run rake db:migrate
heroku open
```

This will commit, push, and migrate heroku, and then open up the browser with the Heroku app URL, which you should add '/articles' to. Once you've done all of this, you should be able to create a article with an image. Congratulations!

Got issues? I'm interested to hear: [info@fluffyjack.com](mailto:info@fluffyjack.com)

### Missing RailsCasts?

I'm launching RailsInMotion, a new weekly Ruby on Rails screencast in March/April 2015, go and follow [@FluffyJack](https://twitter.com/FluffyJack) and [@RailsInMotion](https://twitter.com/RailsInMotion) on Twitter for updates.
