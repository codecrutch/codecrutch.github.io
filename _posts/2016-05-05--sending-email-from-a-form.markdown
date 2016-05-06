---
layout: post
title:  "Sending Email From a Form in Rails"
date:   2016-05-05 19:36
tags: ruby rails
categories: ruby
---

At some point when developing your rails app, you're going to want to be able to
send e-mail to users or even yourself.  You should find a mail delivery service
that can handle sending/tracking/analyzing e-mails reliably.

In a recent application, I decided to use [SendGrid](https://sendgrid.com) to
handle the sending of e-mails for user/admin creation and password resets.
We'll take a look at how I allowed users to send an e-mail to the sales team
right from a webform.

> SendGrid is a scalable e-mail delivery service that provides an SMTP, Web, and Event
API.

First things first, lets take a look at how to set up SendGrid in Rails. If you
aren't signed up for SendGrid, do that now.  If your using
[heroku](https://www.heroku.com) and the heroku toolbelt, you can apply SendGrid's free plan from the command line with: `heroku addons:create sendgrid:starter`

### Setup SendGrid in Development
{% highlight ruby %}
# config/environments/development.rb

  config.action_mailer.perform_deliveries = true
  config.action_mailer.raise_delivery_errors = true
  config.action_mailer.default_url_options = { host: 'localhost', port: 3000 }
  config.action_mailer.delivery_method = :smtp

  ActionMailer::Base.smtp_settings = {
    :address        => 'smtp.sendgrid.net',
    :port           => '587',
    :authentication => :plain,
    :user_name      => ENV['SENDGRID_USERNAME'],
    :password       => ENV['SENDGRID_PASSWORD'],
    :enable_starttls_auto => true
  }
{% endhighlight %}
> Note:  It's recommended to store your SendGrid API key and password out of
> source control by placing it in environment variables.

Next we'll create the mailer so we'll have it when we link up the controller.

### Create a mailer

We will generate a mailer class with an instance method.  This method will
contain information that we want to pass into our e-mail template.
{% highlight ruby %}
rails generate mailer estimate_mailer new_estimate_notification

      create  app/mailers/estimate_mailer.rb
      create  app/mailers/application_mailer.rb
      invoke  erb
      create    app/views/estimate_mailer
      create    app/views/layouts/new_estimate_notification.text.erb
      create    app/views/layouts/new_estimate_notification.html.erb
      invoke  rspec
      create    spec/mailers/estimate_mailer_spec.rb
      create    spec/mailers/previews/estimate_mailer_preview.rb
{% endhighlight %}

We are going to want our email to use data from the form we will create.  To
do this we will let `#new_estimate_notification` take an argument. Since this
e-mail is being sent to only one person, the email recipient is hardcoded.  This
could be changed easily by passing in the current user.

{% highlight ruby %}
# app/mailers/estimate_mailer.rb

class EstimateMailer < ApplicationMailer
  default :from => "email@example.com"

  def new_estimate_notification(estimate)
    @estimate = estimate
    @greeting = "A new estimate request has been submitted online."

    mail(to: "someone@example.com", subject: "Job Estimate Request")
  end
end
{% endhighlight %}

### Create the Form Model

Here is where we will create the attributes that the form will be able to pass
into our e-mail.  Now we could save each request to the database with an
ActiveRecord model, but we'll keep it simple and use `ActiveModel::Model` so we
can have validations.

{% highlight ruby %}
# app/models/estimate.rb

class Estimate
  include ActiveModel::Model

  validates_presence_of :customer_name, :contact, :inquiry

  attr_accessor :customer_name, :contact, :inquiry
end
{% endhighlight %}

### Create your Email

In the e-mail we will use the estimate instance we passed in and the greeting.
We also add the Time that the email was submitted.

{% highlight erb %}
# app/views/estimate_mailer/new_estimate_notification.html.erb

<!DOCTYPE html>
<html>
  <head>
    <meta content='text/html; charset=UTF-8' http-equiv='Content-Type' />
  </head>
  <body>
    <h1>Free Job Estimate Notification</h1>
    <h2><%= @greeting %></h2>
    <h3>Customer Information</h3>
    <p> Customer Name: <%= @estimate.customer_name %> </p>
    <p> Contact Information: <%= @estimate.contact %> </p>
    <h3>Services Requested</h3>
    <p> <%= @estimate.inquiry %> </p>

    <h3>Submitted on <%= Time.now.strftime("%A, %d %b %Y %l:%M %p") %></h3>
  </body>
</html>
{% endhighlight %}

## Linking the Form

Imagine we have an index page on a homes controller (`homes#index`). The first
thing to do is to create a link on a page that will route us to to the form.

### Create a link
{% highlight ruby %}
# app/view/homes/index.html.erb

# ... Everything else ommitted

<%= link_to "Request a Free Estimate!", new_estimate_path %>

{% endhighlight %}

Now we need to setup the route so `new_estimate_path` will work.

### Setup the route

A `get` route is used because I want the route to the form to match:
`http://www.example.com/estimate`

{% highlight ruby %}
# config/routes.rb
Rails.application.routes.draw do
  root "homes#index"

  get 'estimate', to: 'estimates#new', as: 'new_estimate'
  resources :estimates, only: [:create]

  # Other routes ommitted
end
{% endhighlight %}

### Create the controller

{% highlight ruby %}
# app/controllers/estimates_controller.rb

class EstimatesController < ApplicationController
  def new
    @estimate = Estimate.new
  end

  def create
    @estimate = Estimate.new(estimate_params)

    if @estimate.valid?
      EstimateMailer.new_estimate_notification(@estimate).deliver
      redirect_to root_path, notice: 'Request Sent Successfully!'
    else
      render :new
    end
  end

  private

  def estimate_params
    params.require(:estimate).permit(:customer_name, :contact, :inquiry)
  end
end
{% endhighlight %}

### Create the Form

Here I'm using simple form, but you can use the regular form_for helper.
{% highlight erb %}
# app/views/estimates/new.html.erb

<%= simple_form_for(@estimate, html: { class: 'form-group text-left' }) do |f| %>
  <%= f.input :customer_name, label: 'Full Name' %>
  <%= f.input :contact, label: 'Phone or E-mail' %>
  <p>
    <%= f.input :inquiry, label: 'Service(s) Requested?', as: :text, :input_html => {:rows => 7} %>
  </p>
  <%= f.submit 'Send Request', class: 'btn btn-success' %>
  <p></p>
  <%= link_to "Cancel", root_path, class: 'btn btn-danger' %>
<% end %>
{% endhighlight %}


### Finish Line

Thats it!  Your users can now send form data to a specified e-mail.  I recommend
using test driven development(TDD) to develop a feature like this.  With TDD,
the steps will be different than above.  Personally I used Capybara and RSpec
and created a feature spec to guide the process.
