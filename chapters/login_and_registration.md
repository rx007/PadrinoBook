## Login and Registration

In traditional frameworks you would generate a user with a `user` model and a `users_controller` with the actions `new`, `create`, `update`, and `delete`. And you can't forget about security these days it would be nice to to have something at hand to save email the end we would need to find a method of safely storing the password for the user.


You don't have to reinvent the wheel you can use Padrino's beautiful [Admin interface](http://www.padrinorb.com/guides/padrino-admin "Padrino admin") for your user authentication to prevent us from reinventing the wheel. But with that we won't learn the basics and as you will see in this chapter you can make a lot of mistakes. Step into the part of creating users, sending confirmation mails, and understanding how sessions are managed in Padrino.


### Extending the User Model

Before we are going to build the controller and the sign-up form for our application we need to specify the data each user has.


```sh
Name: String
Email: String
Password: String
```


Recording from chapter ~\ref{sec:user_model} we only need to add the `Password` fields to the user table:


Let's create the migration:


```sh
$ padrino-gen migration AddRegistrationFieldsToUsers
  apply  orms/activerecord
  create  db/migrate/004_add_registration_fields_to_users.rb
```


And write the fields into the migration file:


```ruby
# db/migrate/004_add_registration_fields_to_users.rb

class AddRegistrationFieldsToUsers < ActiveRecord::Migration

  @fields = [:password]

  def self.up
    change_table :users do |t|
      @fields.each { |field| t.string field}
    end
  end

  def self.down
    change_table :users do |t|
      @fields.each { |field| remove_column field}
    end
  end
end
```


Run the migrations:


```sh
$ padrino rake ar:migrate
```


### Validating attributes

Before we are going to implement what we think, we are going to write **pending** specs:


```ruby
# spec/app/models/user_spec.rb

require 'spec_helper'

RSpec.describe "User Model" do
  ...

  pending('no blank name')
  pending('no blank email')

  describe "passwords" do
    pending('no blank password')
    pending('no blank password_confirmation')
  end

  describe "when name is already used" do
    pending('should not be saved')
  end

  describe "email address" do
    pending('valid')
    pending('not valid')
  end
end
```


(The *pending* word is optional. It is enough to write pending tests only in the form `it "test this"` and leaving the do/end block away).


Before writing code to pass these specs, we need to add the `password` field to our factory:


```ruby
# spec/factories.rb

# encoding: utf-8
FactoryGirl.define do
...
  factory :user do
    name  "Matthias Günther"
    email "matthias@padrinobook.com"
    password "octocat"
  end
end
```


Use the encoding property to allow special symbols from Germany - you have to add them in your files where they occur. Let's implement the first pending test that a user can't have an empty name:


```ruby
# spec/app/models/user_spec.rb
...

it 'have no blank name' do
  user.name = ""
  expect(user.valid?).to be_falsey
end
...
```


If we run the test, we get the following error:


```sh
$ rspec spec

Failures:

  1) User Model have no blank name
     Failure/Error: expect(user.valid?).to be_falsey
       expected: falsey value
            got: true
     # ./spec/app/models/user_spec.rb:20:in `block (2 levels)
     # in <top (required)>'

Finished in 0.42945 seconds
10 examples, 1 failure, 5 pending

Failed examples:

rspec ./spec/app/models/user_spec.rb:18 # User Model have no blank name
```


To make this test pass we need to validate the `email` property in our user model with the help of the [presence validation](http://guides.rubyonrails.org/active_record_validations_callbacks.html#presence "presence validation"):


```ruby
# app/models/user.rb

class User < ActiveRecord::Base
  validates :name, :presence => true

  has_many :job_offers
end
```


As an exercise, Please write the validates for `email` and `password` on your own. Please consider that the `password_confirmation` attribute can be create with the `:confirmation => true` option to the validates `:password` setting.


We don't want to have duplicated names in our application. For testing this, we need as second user with the same name. In order to create a second user with we need to have another mail address. In order to write the test for it, we need to extend or factory with the [sequence function](https://github.com/thoughtbot/factory_girl/wiki/Usage#sequences-and-associations "sequence functions factory girl"):


```ruby
# spec/factories

FactoryGirl.define do
  sequence(:email){ |n| "matthias.guenther#{n}@padrinobook.com"}

  factory :user do
    name  "Matthias Günther"
    email
    password "foo"
  end
  ...
end
```


Whenever you build a new `user` fixture, the value `email_number` is incremented and gives you a fresh user with a unique email address. You can write a test for this ability in the following way:


```ruby
# spec/app/models/user_spec.rb


RSpec.describe "User Model" do
  let(:user) { build(:user) }
  let(:user_second) { build(:user)}
  ...


  describe "when name is already used" do
    it 'should not be saved' do
      User.destroy_all
      user.save
      user_second.name = user.name
      user_second.save
      expect(user_second.valid?).to be_falsey
    end
  end
end
```


To make the test green you have to use the [uniqueness validation](http://guides.rubyonrails.org/active_record_validations_callbacks.html#uniqueness "uniqueness validation") (Note that we the [destroy_all](http://www.rubydoc.info/docs/rails/4.1.7/ActiveRecord%2FAssociations%2FCollectionProxy%3Adestroy_all "destroy_all") to destroy all users in the database and there associations). All what it does is to validates that the attribute's value is unique before it gets saved.


```ruby
# app/models/user.rb

class User < ActiveRecord::Base
  validates :name, :email, :password, :presence => true
  validates :name, :uniqueness => true

  has_many :job_offers
end
```


Now this test is fixed. Next we are going to implement the validation for the email field:


```ruby
# spec/app/models/user_spec.rb

...

describe "email address" do
  it 'valid' do
    adresses = %w[thor@marvel.de hero@movie.com]
    adresses.each do |email|
      user.email = email
      user_second.email= email
      expect(user_second.valid?).to be_truthy
    end
  end

  it 'not valid' do
    adresses = %w[spamspamspam.de heman,test.com]
    adresses.each do |email|
      user_second.email= email
      expect(user_second.valid?).to be_falsey
    end
  end
end

```


We can test the correctness of the `email` field with a regular expression. First we are going to define a regular expression and use the [format validation](http://guides.rubyonrails.org/active_record_validations_callbacks.html#format "format validation") which takes our regular expression against which the field will be tested.


```ruby
# app/models/user.rb

class User < ActiveRecord::Base
  ...
  VALID_EMAIL_REGEX = /\A[\w+\-.]+@[a-z\d\-.]+\.[a-z]+\z/i
  validates :email, :format => { with: VALID_EMAIL_REGEX }
  ...
end
```


\begin{aside}
\heading{Regular Expressions}

[Regular expressions](http://en.wikipedia.org/wiki/Regular_expression "Regular expressions") are your first tool when you need to match certain parts (or whole) strings against a predefined pattern. The drawback of using them is that you have to learn a formal language to define your patterns. I can highly recommend you the [Rubular tool](http://rubular.com "Rubular tool") for training and trying out the expression you want to use.

\end{aside}


### Users Controller

Since we already have a model for potential users of our platform, it's time to create a controller for them. We are creating in a first step our users controller four our sign up form with only one action:


```sh
$ padrino-gen controller Users get:new
  create  app/controllers/users.rb
  create  app/helpers/users_helper.rb
  create  app/views/users
   apply  tests/rspec
  create  spec/app/controllers/users_controller_spec.rb
```


The new thing about the controller command above is the `get:new` option. This will create an URL route `:new` to `users/new`.


### Sign Up Form

The stage is set: We have the model with the tested constraints, and a controller for the user which handles the action.  Time to create a sign up form for getting new users on our platform. For this case we can use the `form_for` helper.  This method takes an object as its input and creates a form using the attributes of the object. We need this to save/edit the attributes of the model in our controller. Create a new erb file under the users view:


```erb
<%# app/views/users/new.erb %>

<h1>Registration</h1>

<% form_for(@user, '/users/create') do |f| %>
  <%= f.label :name %>
  <%= f.text_field :name %>
  <%= f.label :email %>
  <%= f.text_field :email %>
  <%= f.label :password %>
  <%= f.password_field :password %>
  <%= f.label :password_confirmation %>
  <%= f.password_field :password_confirmation %>
  <p>
  <%= f.submit "Create", :class => "btn btn-primary" %>
  </p>
<% end %>
```


- `form_for`: Is part of [Padrino's Form Builders](http://www.padrinorb.com/guides/application-helpers#formbuilders "Padrino's Form Builders") and allows you to create standard input fields based on a model. The first argument to the function is an object (mostly a model), the second argument is an string (the action to which the form should be sent after a submit), and the third parameter are settings in form of an hash which aren't used in this example. The part `action="/users/create"` says, that we want to use the `create` action to the `users` controller with the `create` action.
- `f.label` and `f.text`: Will a label and text field for the attributes of your model.
- `f.password_field`: Constructs a password input, where the input is marked with stars, from the given attribute of the form.
- `f.submit`: Take an string as an caption for the submit button and options as hashes for additional parameter (for `example :class => 'long'`).


The form will be rendered as the following HTML:


```html
<form method="post" action="/users/create" accept-charset="UTF-8">
  <label for="user_name">Name: </label>
  <input id="user_name" name="user[name]" type="text" />

  <label for="user_email">Email: </label>
  <input id="user_email" name="user[email]" type="text" />

  <label for="user_password">Password: </label>
  <input id="user_password" name="user[password]" type="password" />

  <label for="user_password_confirmation">Password confirmation: </label>
  <input id="user_password_confirmation" name="user[password_confirmation]"
    type="password" />

  <p>
  <input class="btn btn-primary" value="Create" type="submit" />
  </p>
</form>
```



### User Controller Sign up Actions

We need to make sure to have the right mappings for the `/login` route in the actions in our controller:


```ruby
# app/controllers/users.rb

JobVacancy::App.controllers :users do
  get :new, :map => "/login" do
    @user = User.new
    render 'new'
  end
end
```


Feel free to visit <http://localhost:3000/login>. Until now we are not saving the inputs of the user. And what about the mistakes a user makes during his input? How can we display any mistakes a user is making and preserve the things he already typed in? Before doing two steps at a time let's code the `create` action which saves the new registered user without going into error validation.


```ruby
# app/controllers/users.rb

JobVacancy::App.controllers :users do
  ...
  post :create do
    @user = User.new(params[:user])
    @user.save
    redirect('/')
  end
  ...
end
```


Let's go through the new parts:

- `User.new(params[:users])`: Will create a new user object based on the information of the form attributes of the `@user` model which which were part of the form from the views/users/new.erb page.
- `@user.save`: Will save the user in the database.
- `redirect`: Will redirect the user to the root directory of our app.


If you send the form without any inputs, you will see that you are redirected into the root of your app. You can't figure out what's wrong, but luckily we have logs:


```sh
DEBUG -  (0.1ms)  begin transaction
DEBUG - User Exists (0.3ms)  SELECT 1 AS one FROM "users" WHERE
  "users"."name" = '' LIMIT 1
DEBUG - User Exists (0.2ms)  SELECT 1 AS one FROM "users" WHERE
  "users"."email" = '' LIMIT 1
DEBUG -  (0.2ms)  rollback transaction
DEBUG -     POST (0.0162ms) /users/create - 303 See Other
DEBUG - TEMPLATE (0.0004ms) /page/home
DEBUG - TEMPLATE (0.0002ms) /application
DEBUG -      GET (0.0057ms) / - 200 OK
DEBUG -      GET (0.0005ms) application.css?1365616902 - 200 OK
DEBUG -      GET (0.0003ms) application.js?1365616902 - 200 OK
DEBUG -      GET (0.0017ms) /favicon.ico - 404 Not Found
```


The part with the `rollback transaction` means, that user was not saved. Why? Because he violated the validation of our user model. Try to create an `User.new` model in the console and call the `.errors` method on. You should see something like:


```ruby
>> user = User.new
=> #<User id: nil, name: nil, email: nil, created_at: nil, password: nil>
>> user.save
  DEBUG -   (0.1ms)  begin transaction
  DEBUG -  User Exists (0.1ms)  SELECT 1 AS one FROM "users" WHERE "users"."name" IS NULL LIMIT 1
  DEBUG -  User Exists (0.1ms)  SELECT 1 AS one FROM "users" WHERE "users"."email" IS NULL LIMIT 1
  DEBUG -   (0.0ms)  rollback transaction
=> false
>> user.errors
=> #<ActiveModel::Errors:0x9dea518 @base=#<User id: nil, name: nil, email: nil,
   #created_at: nil,
    updated_at: nil, password: nil>, messages{:name=>["can't be blank"],
    :password=>["is too short (minimum is 5 characters)", "can't be blank"],
    :email=>["can't be blank", "is invalid"]}
```


We can use this information to display the errors in our form for the user to let him know what what went wrong. In a first try, we use [error\_messages\_for method](http://www.padrinorb.com/api/Padrino/Helpers/FormHelpers.html#error_messages_for-instance_method "error messages for method"):


```erb
<%# views/users/new.erb %>

<% form_for(@user, '/users/create') do |f| %>
  ...
  <%= error_messages_for @user %>
  ...
<% end %>
```


It counts the number of errors `@user.errors.count` and is looping through all field with their error messages. But this will result in a big box with a bunch of error messages like the following one:


```text
5 errors prohibited this User from being saved
There were problems with the following fields:

Name can't be blank
Password is too short (minimum is 5 characters)
Password can't be blank
Email can't be blank
Email is invalid
```


This isn't something we want to ship to our customers. Let's change this by using [error\_message\_on](http://www.padrinorb.com/api/Padrino/Helpers/FormHelpers.html#error_message_on-instance_method "error message on") which returns a string containing the error message attached to the method on the object:[^error_message_on]


```erb
<%# views/users/new.erb %>

<% form_for(@user, '/users/create') do |f| %>
  <%= f.label :name %>
  <%= f.text_field :name %>
  <%= error_message_on @user, :name %>

  <%= f.label :email %>
  <%= f.text_field :email %>
  <%= error_message_on @user, :email %>

  <%= f.label :password %>
  <%= f.password_field :password %>
  <%= error_message_on @user, :password %>

  <%= f.label :password_confirmation %>
  <%= f.password_field :password_confirmation %>
  <%= error_message_on @user, :password_confirmation %>

  <p>
  <%= f.submit "Create", :class => "btn btn-primary" %>
  </p>
<% end %>
```

[^error_message_on]: Instead of writing `@user` for the `error_message_on` you can also use the symbol notation `:user`.


We can do better and make the error text red. Let's add the `:class` at the of the `error_message_on` method with the help of the [text-error class from bootstrap](http://twitter.github.io/bootstrap/base-css.html#forms "text-error class from bootstrap") and using the `:prepend` option which add text to before displaying the field error:


```erb
<%# views/users/new.erb %>

<% form_for(@user, '/users/create') do |f| %>
  <%= f.label :name %>
  <%= f.text_field :name %>
  <%= error_message_on @user, :name, :class => "text-error",
    :prepend => "The name " %>

  <%= f.label :email %>
  <%= f.text_field :email %>
  <%= error_message_on @user, :email, :class => "text-error",
    :prepend => "The email " %>

  <%= f.label :password %>
  <%= f.password_field :password %>
  <%= error_message_on @user, :password, :class => "text-error",
    :prepend => "The password "%>

  <%= f.label :password_confirmation %>
  <%= f.password_field :password_confirmation %>
  <%= error_message_on @user, :password_confirmation, :class => "text-error" %>

  <p>
  <%= f.submit "Create", :class => "btn btn-primary" %>
  </p>
<% end %>
```


If you fill out the form with complete valid parameters and watch your log again, you can see the following log:


```sh
DEBUG -  (0.2ms)  begin transaction
DEBUG - User Exists (0.3ms)  SELECT 1 AS one FROM "users" WHERE
  "users"."name" = 'Testuser' LIMIT 1
DEBUG - User Exists (0.2ms)  SELECT 1 AS one FROM "users" WHERE
  "users"."email" = 'admin@job-vacancy.de' LIMIT 1
DEBUG - SQL (0.2ms)  INSERT INTO "users"
("created_at", "email", "name", "password", "updated_at") VALUES
(?, ?, ?, ?, ?)  [["created_at", 2013-04-10 20:09:10 +0200],
["email", "admin@job-vacancy.de"],
["name", "Testuser"], ["password", "example"],
["updated_at", 2013-04-10 20:09:10 +0200]]
DEBUG -  (174.1ms)  commit transaction
DEBUG -     POST (0.1854ms) /users/create - 303 See Other
DEBUG - TEMPLATE (0.0004ms) /page/home
DEBUG - TEMPLATE (0.0002ms) /application
DEBUG -      GET (0.0058ms) / - 200 OK
```


Remember that have an eye on your logs can help you to see what's going on in your back-end when you can't see it in the front-end of your app.


\begin{aside}
\heading{What are VALUES (?, ?, ?, ?, ?) in a SQL insert query?}

These form of inserting data in your database is known as parameterized queries. A parameterized query is a query in which placeholders are used for parameters and the parameter values are supplied at execution time. The most important reason to use parameterized queries is to avoid [SQL injection](http://en.wikipedia.org/wiki/SQL_injection "SQL injection") attacks. SQL injection means that SQL statements are injected into input fields in order to drop tables or getting access on user related data.

\end{aside}


### Emails

Padrino uses the [Padrino Mailer gem](https://rubygems.org/gems/padrino-mailer "Padrino Mailer gem") for sending mail. For simplification, we are using SMTP with Gmail. First of all we need to give our application the settings for setting mails in the main configuration file of our application `app.rb`:


```ruby
# app/app.rb

module JobVacancy
  class App < Padrino::Application
    ...
    set :delivery_method, :smtp => {
      :address => 'smtp.gmail.com',
      :port => 587,
      :user_name => '<your-gmail-account-address>',
      :password => '<secret>',
      :authentication => :plain,
    }
    ...
  end
end
```


Let's get through all the different options:


- `:delivery_method`: Defines the delivery method. Possible values are [:smtp](http://en.wikipedia.org/wiki/Simple_Mail_Transfer_Protocol "Simple Mail Transfer Protocol") (default), [:sendmail](http://en.wikipedia.org/wiki/Sendmail "Sendmail"), `:test` (no mails will be send), and [:file](http://edgeguides.rubyonrails.org/action_mailer_basics.html#action-mailer-configuration "Action Mailer configuration") (will write the contents of the email in a file).
- `:address`: The SMTP mail address.
- `:port`: The port of the mail address.
- `:user_name`: The name of the SMTP address.
- `:password`: The password of your SMTP address.
- `:authentication`: Specify if your mail server requires authentication. The default setting is plain meaning that the password is not encrypted. `:login` will send the password [Base64 encoded](https://en.wikipedia.org/wiki/Base64 "Base64 encoded"), [:cram_md5](http://en.wikipedia.org/wiki/CRAM-MD5 "CRAM-MD5") is a challenge/response authentication mechanism .
- `:domain`: This key is set up for [HELO checking](http://en.wikipedia.org/wiki/Anti-spam_techniques#HELO.2FEHLO_checking "HELO checking").


Prior Padrino *0.10.7* the `:enable_starttls_auto => true` was changeable. This is option is now always on true in *Padrino >= 0.11.1* so we leave it out.


This is now the default delivery address unless it is overwritten in an individual mail definition. We won't test the email functionality to this point because the *Mailer gem* is already tested.


#### Quick Mail Usage

To send a first simple "Hallo" message we create an [email block](https://github.com/padrino/padrino-framework/blob/master/padrino-mailer/lib/padrino-mailer/base.rb "email block") directly in our user controller:


```ruby
# app/controllers/users.rb

JobVacancy::App.controllers :users do
  ...
  post :create do
    @user = User.new(params[:user])
    if @user.save
      email do
        from "admin@job-vacancy.de"
        to "lordmatze@gmail.com"
        subject "Welcome!"
        body "hallo"
      end
      redirect('/')
    else
      render 'new'
    end
  end
end
```


Now start the app, go to the URL <http://localhost:3000/login>, and register a fresh user. You can check the log if the mail was send or you "feel" a slow down in your application because it takes a while before the mail is send:


```text
DEBUG - Sending email to: lordmatze@gmail.com
Date: Sun, 14 Apr 2013 09:17:38 +0200
From: admin@job-vacancy.de
To: lordmatze@gmail.com
Message-ID: <516a581243fb3_498a446f81c295e3@mg.mail>
Subject: Welcome!
Mime-Version: 1.0
Content-Type: text/plain;
charset=UTF-8
Content-Transfer-Encoding: 7bit

Hallo
```


#### Mailer

We could go on and parametrized our email example above, but this would mean that we have email code directly into our controller code. We can do better by wrapping up the logic into an object and let it handle the action.


[Padrino mailer](http://www.padrinorb.com/api/Padrino/Mailer.html "Padrino mailer") has the `mailer` command to create customized mailer for every purpose we want to use. Let's create the registration mailer:


```sh
$ padrino-gen mailer Registration registration_email
  create  app/mailers/registration.rb
  create  app/views/mailers/registration
```


Let's break it down:


- `mailer`: The command to create a custom mailer. Inside a mailer you can define the name of your mailer object and it's different templates. The name of our first email is `registration_email`.
- `Registration`: Name of the mailer.


Now we let's look into the `registration.rb` file:


```ruby
# app/mailers/registration.rb

JobVacancy:App.mailer :registration do
  email :registration_email do
    # Your mailer goes here
  end
end
```


The generated comment `# Your mailer goes here` says what you have to do. Let's remove the code from our `users` controller and move it to this place.


```ruby
# app/mailers/registration.rb

JobVacancy::App.mailer :registration do
  email :registration_email do
    from "admin@job-vacancy.de"
    to "lordmatze@gmail.com"
    subject "Welcome!"
    body "Hallo"
  end
end
```


Now we can use the `deliver` method to call our `:registration` mailer with it's template `:registration_email`:


```ruby
# app/controller/users.rb

...
post :create do
  @user = User.new(params[:user])
  if @user.save
    deliver(:registration, :registration_email)
    redirect('/')
  else
    render 'new'
  end
end
...
```


\begin{aside}
\heading{Difference between Padrino's Mailer methods email and deliver}

The [email](http://www.padrinorb.com/api/Padrino/Mailer/Helpers/ClassMethods.html#email-instance_method "email") method is has the parameters `mail_attributes = {}, &block`. That means the you write emails directly `JobVacancy.email(:to => '...', :from => '...', :subject => '...', :body => '...')` or use the block syntax `JobVacancy.email do ... end`. In comparison to this is the [deliver](http://www.padrinorb.com/api/Padrino/Mailer/Helpers/ClassMethods.html#deliver-instance_method "deliver method") method. It has `mailer_name, message_name, *attributes` as attributes. In order to use this you always to create a Mailer for them. If you want to use very simple mails in your application, prefer to use the email method. But if you have templates with a much more complex layout in different formats (plain, HTML), the deliver method is the best fit.

\end{aside}


Instead of writing only a simple "Hallo" in our email we would like to give more input. First we need to write an template and then use the `render` method in our registration mailer. Let's define the registration template:


```erb
<%# app/views/mailers/registration/registration_email.plain.erb %>

Hi ...,

we are glad to have you on our platform. Feel free to post jobs and find the
right people for your application.

Your Job Vacancy!
```


And now we make sure that we are rendering this template in our registration mailer:


```ruby
# app/mailers/registration.rb

JobVacancy::App.mailer :registration do
  email :registration_email do
    from "admin@job-vacancy.de"
    to "lordmatze@gmail.com"
    subject "Welcome!"
    render 'registration_email'
    content_type :plain
  end
end
```


If you are sure that you only want to send plain text mail, you can leave the `plain` extension away but making it explicit will make it clear what you want to do.


To make our email more personal we want to add the name of our freshly registered user to our email template. In order to do this we need to use enable the `locals` option.


```ruby
# app/mailers/registration.rb

JobVacancy::App.mailer :registration do
  email :registration_email do |name, email|
    from "admin@job-vacancy.de"
    to email
    subject "Welcome!"
    locals :name => name
    render 'registration_email'
    content_type :plain
  end
end
```


This options enables a hash which we be used in the email template. Now we need to pass the name to the call of our method in our `users` controller:


```ruby
# app/controllers/users.rb
...

post :create do
  @user = User.new(params[:user])

  if @user.save
    deliver(:registration, :registration_email, @user.name)
    redirect('/')
  else
    render 'new'
  end
end
```


And update our template with the name variable:


```erb
<%# app/views/mailers/registration/registration_email.plain.erb %>

Hi <%= name %>,

we are glad to have you on our platform. Feel free to post jobs and find the
right people for your application.

Your Job Vacancy!
```


Next we want to add a PDF which explains the main business needs to our page. For this purpose we create add the `welcome.pdf` into the `/app/assets/pdf` folder. We can attach files (images, PDF, video) with the [add_file method](https://github.com/mikel/mail/blob/master/lib/mail/message.rb "add file method of action mailer") which takes a filename and the content as hash elements as arguments.


```ruby
# app/mailers/registration.rb
...

email :registration_email do |name, email|
  from "admin@job-vacancy.de"
  to email
  subject "Welcome!"
  locals :name => name, :email=> email
  render 'registration_email'
  add_file :filename => 'welcome.pdf', :content =>
    File.open("#{Padrino.root}/app/assets/pdf/welcome.pdf") { |f| f.read }
end
```


Please correct me if there is a better way to get to the asset folder but that is all of what I've found.


During writing this chapter I experiment with the `content_type` option. If you set the `content_type` to plain you will get the attachment based as binary code directly into your mail. Please put the `content_type :plain` into the `registration` mailer. If the mail will be send you can see something like this in your logs:


```text
  DEBUG - Sending email to: lordmatze@gmail.com
Date: Thu, 18 Apr 2013 18:34:15 +0200
From: admin@job-vacancy.de
To: lordmatze@gmail.com
Message-ID: <5170208754967_70f748e80108323a@mg.mail>
Subject: Welcome!
Mime-Version: 1.0
Content-Type: text/plain
Content-Transfer-Encoding: 7bit

----==_mimepart_517020874676e_70f748e8010829d8
Date: Thu, 18 Apr 2013 18:34:15 +0200
Mime-Version: 1.0
Content-Type: text/plain;
 charset=UTF-8
Content-Transfer-Encoding: 7bit
Content-ID: <5170208753fd5_70f748e8010831c1@mg.mail>

Hi Bob,

we are glad to have you on our platform. Feel free to post jobs and find the
right people for your application.

Your Job Vacancy!

----==_mimepart_517020874676e_70f748e8010829d8
Date: Thu, 18 Apr 2013 18:34:15 +0200
Mime-Version: 1.0
Content-Type: application/pdf;
 charset=UTF-8;
 filename=welcome2.pdf
Content-Transfer-Encoding: base64
Content-Disposition: attachment;
 filename=welcome2.pdf
Content-ID: <517020874ba76_70f748e80108301@mg.mail>

JVBERi0xLjQKJcOkw7zDtsOfCjIgMCBvYmoKPDwvTGVuZ3RoIDMgMCBSL0Zp
bHRlci9GbGF0ZURlY29kZT4+CnN0cmVhbQp4nDPQM1Qo5ypUMABCM0MjBXNL
...
# You don't want to read four pages of strange characters ...
...

----==_mimepart_517020874676e_70f748e8010829d8--
```


\begin{aside}
\heading{MIME?}

MIME stands for "Multipurpose Internet Mail Extensions" and they specify additional attributes to email headers like the content type and define transfer encodings which can be used to present a higher encoded file (e.g. 8-bit) with the 7-bit ASCII character set. This makes it possible to put non-English characters in the message header like the subject. The goal of the MIME definition was that existing email servers had nothing to change in order to use MIME types. This means that MIME headers are optional for plain text emails and even none MIME messages can be read correctly by a clients being able to read MIME encoded messages.

\end{aside}


### Sending Email with Confirmation Link

The basic steps for implementing the logic of email confirmation are the following:


- We need to add the *confirmation_code* and *confirmation* attributes in our user model.
- Create a controller method for our user model that expects a user id and confirmation code, looks up the user, checks the code in the parameter matches the code saved in our database and clears the code after confirmation.
- Create an action that maps to our new controller method (e.g. `/confirm/<user-id>/<code>`).
- Create an mailer template which takes the user as a parameter and use the *confirmation code* of the user to send a mail containing a link to the new route in our controller.
- If the email of the user needs to be modified or a new User record is created we need to create a confirmation code, set it in the model and clear the confirmation flag
- Create a helper method which allows views to check if the current user is confirmed.
- Protect our controller methods and views to prevent security issues.


\begin{aside}
\heading{Why Confirmation Mail?}

Check that the user actually signed up for the account and actually wants it. This also helps you from spamming your platform is going to be floated with billions of users. Another usage of this information is to give your users a chance to change their password and/or stay in contact with them to inform them about updates.

\end{aside}


### Add Confirmation Code and Confirmation Attributes for the User Model

Create a good migration which fits to the task we want to do:


```sh
$ padrino-gen migration AddConfirmationCodeAndConfirmationToUsers
    confirmation_code:string confirmation:boolean
   apply  orms/activerecord
  create  db/migrate/005_add_confirmation_code_and_confirmation_to_users.rb
```


Now let's add the fields to a migration:


```ruby
# db/migrate/005_add_confirmation_code_and_confirmation_to_users.rb

class addconfirmationcodeandconfirmationtousers < activerecord::migration
  def self.up
    change_table :users do |t|
      t.string :confirmation_code
      t.boolean :confirmation, default => false
    end
  end

  def self.down
    change_table :users do
      remove_column :confirmation_code, :confirmation
    end
  end
end
```


We added the `:default` option which sets the confirmation for every user to false if a new one is registered. Now let's migrate our production and test database to this new event:


```sh
$ padrino rake ar:migrate
$ padrino rake ar:migrate -e test
```


### My Tests are Slow ...

During writing this book I discovered various strange behavior for my tests because I was writing data into my test database. The tests weren't really reliable because some worked only when the database is fresh with no preexisting entries. One solution would be to clean up the database before each run:


```sh
$ sqlite3 db/job_vacancy_test.db
  SQLite version 3.7.13 2012-06-11 02:05:22
  Enter ".help" for instructions
  Enter SQL statements terminated with a ";"
  sqlite> DELETE FROM users;
  sqlite> .quit
```


But after this my tests were running very slow:


```sh
$ rspec spec
...

Finished in 1.61 seconds
25 examples, 0 failures
```


Running them again make them a little bit faster:


```sh
$ rspec spec
...

Finished in 0.77209 seconds
25 examples, 0 failures
```


Why? Because we are hitting the database and our tests slow. Please consider code like the following:


```ruby
# spec/app/models/user_spec.rb


describe "when name is already used" do
  it 'should not be saved' do
    User.destroy_all
    user.save
    user_second.name = user.name
    user_second.save
    expect(user_second.valid?).to be_falsey
  end
end

describe "when email address is already used" do
  it 'should not save an user with an existing address' do
    user.save
    user_second.email = user.email
    user_second.save
    expect(user_second.valid?).to be_falsey
  end
end
```


We can use mocks the saving operation away. The benefit of mocks are that you create the environment you want to test and don't care about all the preconditions to make this test possible.


Consider the following code example:


```ruby
# app/controllers/users.rb

post :create do
  user = User.find_by_email(params[:email])

  if user && user.confirmation && user.password == params[:password]
    redirect '/'
  else
    render 'new'
  end
end
```


In order to test the condition `if user && user.confirmation && user.password == params[:password]` to return the redirect we need find a User by email out of our database. A normal test would be in need to create a user, save it and giving the object the right attributes to pass it. We can use mocks to simulate this environment by creating a user out of our users factory, setting the attributes of this object and cheating our `find-by-email` method to return our factory user with the right params without actually saving our object to the database:


```ruby
# spec/app/controllers/users_controller_spec.rb

it "should redirect if user is correct" do
  user.confirmation = true
  User.should_receive(:find_by_email).and_return(user)
  post "sessions/create", user.attributes

  last_response.should be_redirect
end
```


The magic behind mocking is to use the [should_receive](https://github.com/rspec/rspec-mocks#message-expectations "should receive from RSpec") and [and_return](https://github.com/rspec/rspec-mocks#consecutive-return-values "and_return from RSpec") flow. `Should_receive` says which method should be called and `and_return` what should be returned when the specified method is called. The line size of our tests will remain the same - you only have to write more characters :) but this will speed up your tests in the long term. With the help of mocks you keep your tests fast and robust.


Even if this will be the first application you write, when you've learned something new and this will make your life easier, go back and take your time to enhance the style and design of your application.


### Controller Method and Action For Password Confirmation
\label{sec:controller_method_and_action_for_password_confirmation}

When we are going to register a new user, we need to create a confirmation code like in the example above. Since this is business logic, we will put this method inside our users model. First we will write a failing test:


```ruby
# spec/app/models/user_spec.rb
...


  describe "confirmation code" do
    let(:user_confirmation) { build(:user) }

    it 'should not be blank' do
      user_confirmation.confirmation_code = ""
      expect(user_confirmation.valid?).to be_falsey
    end
  end
...
```


To make this test pass we add the validates presence of ability in our user model:


```ruby
# app/models/user.rb

class User < ActiveRecord::Base
  ...
  validates :confirmation_code, :presence => true
  ...
end
```


Next we need think of how we can set the `confirmation_code` information to our freshly created user. Instead of creating a confirmation code on our own, we want to encrypt the password by some mechanism. Luckily, we can use [bcrypt](https://github.com/codahale/bcrypt-ruby "bcrypt gem") to create our confirmation code. It is a Ruby binding for the [OpenBSD bcrypt](http://en.wikipedia.org/wiki/OpenBSD_security_features "OpenBSD bcrypt") password hashing algorithm. In order to use this in our app we need to add it to our `Gemfile`:


```ruby
# Gemfile
...

# Security
gem 'bcrypt-ruby', '~> 3.1', :require => 'bcrypt'
```


Now let's open the console and play around with this Gem:


```sh
$ padrino c
  => Loading development console (Padrino v.0.12.5)
  => Loading Application JobVacancy
  >> password = "Test11111134543"
  => "Test11111134543"
  >> salt = "$2a$05$CCCCCCCCCCCCCCCCCCCCC.E5YPO9kmyuRGyh0XouQYb4YMJKvyOeW"
  => "$2a$05$CCCCCCCCCCCCCCCCCCCCC.E5YPO9kmyuRGyh0XouQYb4YMJKvyOeW"
  >> BCrypt::Engine.hash_secret(password, salt)
  => "$2a$05$CCCCCCCCCCCCCCCCCCCCC.9APD.dklRtXYdki/E3XrHiCWd/rfAFu"
```


\begin{aside}
\heading{What is a Salt?}

Salts are used in cryptography as random data to be put as addition to normal password to create a encrypted with the help of a one-way function. A one-way function output by some input string very easily but the other way round is very difficult for the computer to compute the original string from the output.  Salts make it more difficult for hackers to get the password via rainbow tables attacks. Rainbow tables are a huge list of precomputed hashes for widely used password. If a hacker gets access to a password hash he then compare this hash with the entries. If he finds after which he was searching he got the password for the user.

\end{aside}


We could add these methods in the users controller but that isn't something a controller should do. We better use a [callback](http://guides.rubyonrails.org/active_record_validations_callbacks.html#callbacks-overview "callbacks for ActiveRecord validations"). **Callbacks** are methods to run on a certain stage or life cycle of an object. Perfect, that's what we want let's create code for it:


```ruby
# app/models/user.rb

class User < ActiveRecord::Base
  ... # The other validations

  before_save :encrypt_confirmation_code,
    :if => :registered? # our callback with if condition

  private
  def encrypt_confirmation_code
    self.confirmation_code = set_confirmation_code
  end

  def set_confirmation_code
    require 'bcrypt'
    salt = BCrypt::Engine.generate_salt
    confirmation_code = BCrypt::Engine.hash_secret(self.password, salt)
    normalize_confirmation_code(confirmation_code)
  end

  def normalize_confirmation_code(confirmation_code)
    confirmation_code.gsub("/", "")
  end

  def registered?
    self.new_record?
  end
end
```


We won't test the methods under the private keyword, there is no customized business logic inside these methods. We will even not test the difficult looking `set_confirmation_code` method because there is no customized business logic inside, and BCrypt is well tested.


\begin{aside}
\heading{Why private callbacks?}

It is good practice to make your callbacks private that they can called only from inside the model and no other object can use these methods. Our `confirmation_code` method is public available but that is no problem, because it generates a random string.

\end{aside}


After creating the confirmation code mechanism for our user, we need to implement a authenticate which takes the confirmation code as an input and mark our user as *confirmed*. As always, let's begin with failing tests first:


```ruby
# spec/app/models/user_spec.rb
...

describe "confirmation code" do
  let(:user_confirmation) { build(:user) }


  it 'should authenticate user with correct confirmation code and should be confirmed' do
    user_confirmation.save
    confirmation_of_saved_user = User.find_by_id(user_confirmation.id)
    user_confirmation.confirmation_code =
      confirmation_of_saved_user.confirmation_code
    expect(user_confirmation.authenticate(user_confirmation.confirmation_code)).to be_truthy
    expect(user_confirmation.confirmation).to be_truthy
  end

  it 'should not authenticate user with incorrect confirmation code' do
    expect(user_confirmation.authenticate("wrong")).to be_falsey
  end

  it 'should not authenticate user with incorrect confirmation code' do
    user_confirmation.authenticate("wrong").should be false
  end
end
```


\begin{aside}
\heading{Take care of your names!?}

During writing this chapter I wasted a lot of time hour because I had method with the same name as the `confirmation_code` field. When I wanted to check `@user.confirmation_code` it always called the `confirmation_code` method which return a new confirmation code. I was thinking for a long time that it returned the attribute and was wondering what's going on. A couple of [pry](http://pryrepl.org "pry") sessions showed me nothing since I'm expected to be right. After I went to the toilet I started another pry session and out of sudden I discovered my naming problem.

Lesson learned: Breaks are great!

\end{aside}


Before going on we need to update our `factory` for the test with the confirmation code field::


```ruby
# spec/factories.rb

# encoding: utf-8
FactoryGirl.define do
  ...
  sequence(:confirmation_code){ "1" }
  sequence(:id){ |n| n }

  factory :user do
    id
    name
    email
    password "octocat"
    confirmation_code
  end
  ...
end
```


We are making the `confirmation_code` with the value of 1 static because it for the tests.  Here is now the code that makes our tests green:


```ruby
# app/models/user.rb

class User < ActiveRecord::Base
  ...

  def authenticate(confirmation_code)
    return false unless @user = User.find_by_id(self.id)

    if @user.confirmation_code == self.confirmation_code
      self.confirmation = true
      self.save
      true
    else
      false
    end
  end
  ...
end
```


Since our tests of our user model are now green, let's write tests for our `/confirm` route:


```ruby
# spec/app/controllers/users_controller.rb

describe "GET confirm" do
  let(:user) { build(:user) }

  it "render the '/confirm' page if user has confirmation code" do
    user.save
    user_confirmed = User.find_by_id(user.id)
    get "/confirm/#{user_confirmed.id}/#{user_confirmed.confirmation_code.to_s}"
    expect(last_response).to be_ok
  end


  it "redirect to :confirm if user id is wrong" do
    get "/confirm/test/#{user.confirmation_code.to_s}"
    expect(last_response).to be_redirect
  end


  it "redirect to :confirm if user id is wrong" do
    get "/confirm/test/#{user.confirmation_code.to_s}"
    expect(last_response).to be_redirect
  end
end
```


To make this pass, we implement the following code:


```ruby
# app/controllers/users.rb
...

get :confirm, :map => "/confirm/:id/:code" do
  redirect('/') unless @user = User.find_by_id(params[:id])
  redirect('/') unless @user.authenticate(params[:code])
  render 'confirm'
end
```


### Mailer Template for Confirmation Email

If we are lazy we could add our confirmation email into the registration mailer. These are two things that have nothing to do with each other, let's create another mailer:


```sh
$ padrino-gen mailer Confirmation confirmation_email
  create  app/mailers/confirmation.rb
  create  app/views/mailers/confirmation
```


Now fill out the confirmation mailer:


```ruby
# app/mailers/confirmation.rb

JobVacancy::App.mailer :confirmation do
  CONFIRMATION_URL = "http://localhost:3000/confirm"

  email :confirmation_email do |name, email, id, link|
    from "admin@job-vacancy.de"
    subject "Please confirm your account"
    to email
    locals :name => name, :confirmation_link =>
      "#{CONFIRMATION_URL}/#{id}/#{link}"
    render 'confirmation_email'
  end
end
```


Fill the email template with "confirmation-link-life":


```erb
<%# app/views/mailers/confirmation/confirmation_email.plain.erb %>

Hi <%= name %>,

to take fully advantage of our platform you have to follow the following link:

<%= confirmation_link %>

Enjoy the possibility to find the right people for your jobs.
```


And call this method to our users controller:


```ruby
# app/controllers/users.rb

post :create do
  @user = User.new(params[:user])

  if @user.save
    deliver(:registration, :registration_email, @user.name, @user.email)
    deliver(:confirmation, :confirmation_email, @user.name,
            @user.email,
            @user.id,
            @user.confirmation_code)
    redirect('/')
  else
    render 'new'
  end
end
```


### Registration and Confirmation e-mails

The code is working but we have flaws in our design:


1. The controller is sending mails but this is not the responsibility of it.
2. Our user model is blown up with authentication code.


\begin{aside}
\heading{Observers vs. Callbacks vs. POROs}

[Observers](http://en.wikipedia.org/wiki/Observer_pattern "Observers") are a design pattern where an object has a list of its dependents called observers, and notifies them automatically if its state has changed by calling one of their methods. Observers means to be decoupling responsibility. They can serve as a connection point between your models and some other functionality of another subsystem. Observers "lives" longer in your application and can be attached/detached at any time. [Callbacks](http://guides.rubyonrails.org/active_record_callbacks.html "Callbacks") life shorter - you pass it to a function to be called only once. *Rule of the thumb*: When you use callbacks with code that isn't directly related to your model, you better put this into an observer.


The Observer pattern decouples event producers from event consumers but tightly couples models to them - and that make
it hard to test them and you always have to take them with you.  Besides they add a kind of hidden magic to your code,
you may forget when you that they are always around you. Better way is to make those calls explicit in your controller.
That where **Plain Old Ruby Objects** ([PORO](http://blog.steveklabnik.com/posts/2011-09-06-the-secret-to-rails-oo-design "PORO"))
jump in. They make magic calls explicit and are easier to test.
\end{aside}


We want to have a class which sends the registration and confirmation email. To see what we can move out of the user
model let's have a look inside this model:


```ruby
# app/models/user.rb

User < ActiveRecord::Base
  ...
  before_save :encrypt_confirmation_code, :if => :registered?

  private
  def encrypt_confirmation_code
    self.confirmation_code = set_confirmation_code
  end

  def set_confirmation_code
    require 'bcrypt'
    salt = BCrypt::Engine.generate_salt
    confirmation_code = BCrypt::Engine.hash_secret(self.password, salt)
    normalize_confirmation_code(confirmation_code)
  end

  def registered?
    self.new_record?
  end

  def normalize_confirmation_code(confirmation_code)
    confirmation_code.gsub("/", "")
  end
end
```


And refactor the code above into our `UserCompletion` class:


```ruby
# lib/user_completion.rb

class UserCompletion
  require 'bcrypt'

  attr_accessor :user, :app

  def initialize(user, app = JobVacancy::App)
    @user = user
    @app ||= app
  end

  def send_registration_mail
    self.app.deliver(:registration, :registration_email, self.user.name,
      self.user.email)
  end

  def send_confirmation_mail
    self.app.deliver(:confirmation, :confirmation_email, user.name,
      user.email,
      user.id,
      user.confirmation_code)
  end

  def encrypt_confirmation_code
    salt = BCrypt::Engine.generate_salt
    confirmation_code = BCrypt::Engine.hash_secret(self.user.password, salt)
    self.user.confirmation_code = normalize(confirmation_code)
  end

  private
  def normalize(confirmation_code)
    confirmation_code.gsub("/", "")
  end
end
```


We are not using the single `deliver` method here because our file does not have access to this it. Instead we have
to take `JobVacancy::App.deliver` way to access the mail (it is not documented as I
[found out](https://github.com/padrino/padrino-framework/issues/1770)).


We need to remove the callback `before_save :encrypt_confirmation_code, :if => :registered?` and we need also to
transfer this logic into the controller:


```ruby
# app/controllers/users.rb

JobVacancy::App.controllers :users do
  ...
  post :create do
    @user = User.new(params[:user])
    user_completion = UserCompletion.new(@user)
    user_completion.encrypt_confirmation_code

    if @user && @user.save
      user_completion.send_registration_mail
      user_completion.send_confirmation_mail
      redirect '/', flash[:notice] = "You have been registered.  Please confirm
        with the mail we've send you recently."
    else
      render 'new'
    end
  end
end
```


If we have a fresh registered user we create an confirmation code and send him an welcome mail right after the confirmation email.


And the tests for the `UserCompletion` class:


```ruby
# spec/lib/user_completion_spec.rb

require 'spec_helper'

RSpec.describe "UserCompletion" do
  describe "user new record" do
    let(:user) { build(:user)}

    it "encrypts the confirmation code of the user" do
      salt = "$2a$10$y0Stx1HaYV.sZHuxYLb25."
      confirmation_code = "$2a$10$y0Stx1HaYV.sZHuxYLb25.zAi0tu1C5N.oKMoPT6NbjtD
        /.3cg7Au"
      expected_confirmation_code = "$2a$10$y0Stx1HaYV.sZHuxYLb25.zAi0tu1C5N.oKM
        oPT6NbjtD.3cg7Au"
      expect(BCrypt::Engine).to receive(:generate_salt).and_return(salt)
      expect(BCrypt::Engine).to receive(:hash_secret).with(user.password, salt)
        .and_return(expected_confirmation_code)
      @user_completion = UserCompletion.new(user, app(JobVacancy::App))
      @user_completion.encrypt_confirmation_code

      expect(@user_completion.user.confirmation_code).to eq expected_confirmation_code
    end

    it "sends registration mail" do
      expect(app).to receive(:deliver).with(:registration, :registration_email,
        user.name, user.email)

      @user_completion = UserCompletion.new(user, app)
      @user_completion.send_registration_mail
    end

    it "sends confirmation mail" do
      expect(app).to receive(:deliver).with(:confirmation, :confirmation_email,
        user.name, user.email, user.id, user.confirmation_code)

      @user_completion = UserCompletion.new(user, app)
      @user_completion.send_confirmation_mail
    end
  end
end
```


And the tests for the controller:


```ruby
# spec/app/controllers/users_controller_spec.rb

require 'spec_helper'

RSpec.describe "UsersController" do
  ...

  describe "POST /users/create" do
    let(:user) { build(:user) }
    before do
      @user_completion = double(UserCompletion)
      expect(User).to receive(:new).and_return(user)
      expect(@user_completion).to receive(:encrypt_confirmation_code)
    end

    it "redirects to home if user can be saved", :current do
      expect(user).to receive(:save).and_return(true)
      expect(UserCompletion).to receive(:new).with(user).and_return(@user_completion)
      expect(@user_completion).to receive(:send_registration_mail)
      expect(@user_completion).to receive(:send_confirmation_mail)
      post "/users/create"
      expect(last_response).to be_redirect
      expect(last_response.body).to eq "You have been registered. Please confirm
        with the mail we've send you recently."
    end

    it "renders registration page if user cannot be saved" do
      expect(UserCompletion).to receive(:new).with(user).and_return(@user_completion)
      expect(user).to receive(:save).and_return(false)
      post "/users/create"
      expect(last_response).to be_ok
      expect(last_response.body).to include 'Registration'
    end
  end
end
```

