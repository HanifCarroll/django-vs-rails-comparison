# django-vs-rails-comparison

## About

This document compares the Django and Ruby on Rails webframeworks. The accompanying "API" tools (Django Rest Framework and Rails' "API Mode") are also considered.

The purpose of this documen is to outline how each framework handles common tasks related to creating a web application. These comparisons will hopefully inform the reader on which framework they would prefer to use.

## Scaffolding

### Django

No tool to explicitly generate models, views, controllers, or routes. Generation happens through usage of convenience utilities that Django/DRM offers (e.g. ModelViewSet).

### Rails

`rails generate scaffold books` will create

- A `BooksController`
- A `Book` model
- A new `resources :books` route added to your `config/routes.rb` file
- A set of test files
- View files under `app/views/books` (five in total)
- DB Migration

Rails can also generate specific components.

- `rails g controller Fruit`
- `rails g model Fruit name:string color:string` (creates model + migration)
- `rails g migration CreateBook title:string year:integer` (creates migration only)
- `rails generate controller CreditCards open debit credit close` (creates controller with the following actions)

Full list of generator options:

```
  application_record
  benchmark
  channel
  controller
  generator
  helper
  integration_test
  jbuilder
  job
  mailbox
  mailer
  migration
  model
  resource
  scaffold
  scaffold_controller
  system_test
```

## Models and migrations

### Django

Created as normal Python classes that subclass a Django model class.

DB fields are then added as instance properties.

```python
from django.db import models

class Person(models.Model):
    first_name = models.CharField(max_length=30)
    last_name = models.CharField(max_length=30)
```

Model properties can be seen directly in the model file.

The process of making changes to the DB is:

1. Update the models.
2. Run `django-admin makemigrations`
3. Run `django-admin migrate`

### Rails

Models are generated from the command line, optionally passing in properties:
`rails g model Article title:string body:text`

This creates a migration that can then be modified to add more properties.

```ruby
class CreateArticles < ActiveRecord::Migration[7.0]
  def change
    create_table :articles do |t|
      t.string :title
      t.text :body

      t.timestamps
    end
  end
end
```

Model properties are viewed in the `scheme.rb` file, which shows the current view of the DB entities.

Modifying model fields are done by creating a migration. The rails generator can help with this.

If the migration name is of the form "AddColumnToTable" or "RemoveColumnFromTable" and is followed by a list of column names and types then a migration containing the appropriate [`add_column`](https://api.rubyonrails.org/v7.0.4.2/classes/ActiveRecord/ConnectionAdapters/SchemaStatements.html#method-i-add_column) and [`remove_column`](https://api.rubyonrails.org/v7.0.4.2/classes/ActiveRecord/ConnectionAdapters/SchemaStatements.html#method-i-remove_column)statements will be created.

`bin/rails generate migration AddPartNumberToProducts part_number:string` will generate the following migration:

```ruby
class AddPartNumberToProducts < ActiveRecord::Migration[7.0]
  def change
    add_column :products, :part_number, :string
  end
end

```

Modifying multiple fields at once is possible:
`bin/rails generate migration AddDetailsToProducts part_number:string price:decimal`

```ruby
class AddDetailsToProducts < ActiveRecord::Migration[7.0]
  def change
    add_column :products, :part_number, :string
    add_column :products, :price, :decimal
  end
end
```

Running `bin/rails db:migrate` runs the migration.

## Model Relationships

### Django

```python
from django.db import models

class Manufacturer(models.Model):
    # ...
    pass

class Car(models.Model):
    manufacturer = models.ForeignKey(Manufacturer, on_delete=models.CASCADE)
    # ...
```

### Rails

```ruby
class Author < ApplicationRecord
  has_many :books, dependent: :destroy
end

class Book < ApplicationRecord
  belongs_to :author
end
```

## Custom Model Validators

### Django

Validators are normal functions that attach to the model.

```python
from django.core.exceptions import ValidationError
from django.utils.translation import gettext_lazy as _
from django.db import models

def validate_even(value):
    if value % 2 != 0:
        raise ValidationError(
            _('%(value)s is not an even number'),
            params={'value': value},
        )


class MyModel(models.Model):
    even_field = models.IntegerField(validators=[validate_even])
```

These validators can also be used with form validation.

Django's `ModelForm` class runs the validation or you can call the instance's `full_clean()` method to run them.

### Rails

Validators are classes with a validate method that attach to the model.

```ruby
class MyValidator < ActiveModel::Validator
  def validate(record)
    unless record.name.start_with? 'X'
      record.errors.add :name, "Need a name starting with X please!"
    end
  end
end

class Person
  include ActiveModel::Validations
  validates_with MyValidator
end
```

You can add custom methods as validators that run when you call `instance.valid?` or save he object. These methods should add errors to the `errors` collection.

```ruby
class Invoice < ApplicationRecord
  validate :expiration_date_cannot_be_in_the_past,
    :discount_cannot_be_greater_than_total_value

  def expiration_date_cannot_be_in_the_past
    if expiration_date.present? && expiration_date < Date.today
      errors.add(:expiration_date, "can't be in the past")
    end
  end

  def discount_cannot_be_greater_than_total_value
    if discount > total_value
      errors.add(:discount, "can't be greater than total value")
    end
  end
end
```

You can also run validation on create or update:

```ruby
class Invoice < ApplicationRecord
  validate :active_customer, on: :create

  def active_customer
    errors.add(:customer_id, "is not active") unless customer.active?
  end
end
```

## Returning a model as a response

### Django

Uses Serializers in order to map the object to JSON.

```python
from django.core import serializers
data = serializers.serialize("json", SomeModel.objects.all())
```

### Rails

Able to return JSON directly from the controller method.
`render :json => @product` or `render json: @product`

You can customize the JSON representation by using the `to_json` method.

```ruby
  some_model.to_json(:except => [ :id, :created_at, :age ])
  # => {"name": "Model Name", "other_property": true}
```

For APIs, better to use the Active Model Serializers.

```ruby
class User
  include ActiveModel::Serializers::JSON
  attr_accessor :name, :notes # Emulate has_many :notes
  def attributes
    {'name' => nil}
  end
end

class Note
  include ActiveModel::Serializers::JSON
  attr_accessor :title, :text
  def attributes
    {'title' => nil, 'text' => nil}
  end
end

note = Note.new
note.title = 'Battle of Austerlitz'
note.text = 'Some text here'

user = User.new
user.name = 'Napoleon'
user.notes = [note]

user.serializable_hash
# => {"name" => "Napoleon"}
user.serializable_hash(include: { notes: { only: 'title' }})
# => {"name" => "Napoleon", "notes" => [{"title"=>"Battle of Austerlitz"}]}
```

## UI for interacting with models

### Django

Admin site, part of the framework.

### Rails

Various gems exist that you can add to your project.

## Browsable UI

### Django

Django offers an HTML interface that allows a user to interact with the API through the browser as well as see documentation for it.
[The Browsable API - Django REST framework](https://www.django-rest-framework.org/topics/browsable-api/)

### Rails

Rails offer no such tool, as far as I know.

## Async Code

### Django

```python
import asyncio
from django.http import HttpResponse
from django.views import View

class AsyncView(View):
    async def get(self, request, *args, **kwargs):
        # Perform io-blocking view logic using await, sleep for example.
        await asyncio.sleep(1)
        return HttpResponse("Hello async world!")
```

### Rails

Not available, however there is a gem (Async Ruby) that allows a developer to utilize something like coroutines.

## Websockets

### Django

Handled with an add-on package called Channels, which also handles other types of long-running connections.

### Rails

Handle with ActionCable, which is provided by the framework.

## Router

### Django

Routes are linked to views that are either functions or classes.

```python
from django.http import HttpResponse
from django.views import View
from django.urls import path
from django.shortcuts import render

class MyClassView(View):
    def get(self, request):
        # <view logic>
        return HttpResponse('result')

def my_function_view(request):
    if request.method == 'GET':
        # <view logic>
        return HttpResponse('result')

urlpatterns = [
    path('class-view/', MyClassView.as_view()),
    path('function-view', my_function_view),
]
```

Django Rest Framework allows generation of URLs based on  a corresponding `ViewSet`, which holds the corresponding actions to be run for each URL. Only URLs that have a corresponding action in the `ViewSet` will be created.

```python
from rest_framework import routers

router = routers.SimpleRouter()
router.register(r'users', UserViewSet)
router.register(r'accounts', AccountViewSet)
urlpatterns = router.urls
```

The above generates:

- URL pattern: `^users/$` Name: `'user-list'`
- URL pattern: `^users/{pk}/$` Name: `'user-detail'`
- URL pattern: `^accounts/$` Name: `'account-list'`
- URL pattern: `^accounts/{pk}/$` Name: `'account-detail'`

```python
from django.contrib.auth.models import User
from django.shortcuts import get_object_or_404
from myapps.serializers import UserSerializer
from rest_framework import viewsets
from rest_framework.response import Response

class UserViewSet(viewsets.ViewSet):
    """
    A simple ViewSet for listing or retrieving users.
    """
    def list(self, request):
        queryset = User.objects.all()
        serializer = UserSerializer(queryset, many=True)
        return Response(serializer.data)

    def retrieve(self, request, pk=None):
        queryset = User.objects.all()
        user = get_object_or_404(queryset, pk=pk)
        serializer = UserSerializer(user)
        return Response(serializer.data)
```

The above is an explicit declaration of the logic that should be run when we hit the index (collection) and detail (single item from the collection) routes.

```python
class UserViewSet(viewsets.ModelViewSet):
    """
    A viewset for viewing and editing user instances.
    """
    serializer_class = UserSerializer
    queryset = User.objects.all()
```

The above is a convenience class that Django Rest Framework provides. The `ModelViewSet` provides default implementations for  `.list()`, `.retrieve()`,  `.create()`, `.update()`, `.partial_update()`, and `.destroy()`.  Particular methods can be overridden to provide a custom implementation.

The advantage of using `ViewSet`s is the ability to colocate logic (such as `serializer_class` and `queryset` instead of repeating it throughout various methods.

Other `ViewSet`s exist that generate a subsection of the routes, such as `ReadOnlyModelViewSet` , which only provides actions for `.list()` and `.retrieve()`.

 ### Rails

```ruby
Rails.application.routes.draw do
  resources :brands, only: [:index, :show] do
    resources :products, only: [:index, :show]
  end

  resource :basket, only: [:show, :update, :destroy]

  resolve("Basket") { route_for(:basket) }
end
```

Routes are linked to controller actions. For the `brands` resource, the index action can be found in the `BrandsController` file under the method called `index`.

`resource` generates URLs for `index`, `show`, `new`, `edit`, `create`, `update`, and `destroy`actions, but the `only:` property can be used to generate routes for specific actions.

`resource` also generates a number of convenience methods that return the URLs to particular routes.

- `photos_path` returns `/photos`
- `new_photo_path` returns `/photos/new`
- `edit_photo_path(:id)` returns `/photos/:id/edit` (for instance, `edit_photo_path(10)` returns `/photos/10/edit`)
- `photo_path(:id)` returns `/photos/:id` (for instance, `photo_path(10)` returns `/photos/10`)

Each of these helpers has a corresponding `_url` helper (such as `photos_url`) which returns the same path prefixed with the current host, port, and path prefix.

Singular routes can be created with

``` ruby
get 'profile', to: 'users#show'
```

## Testing

### Django

Includes a module based on Python standard library module `unittest`, which defines tests with a class-based approach.  

```python
from django.test import TestCase
from myapp.models import Animal

class AnimalTestCase(TestCase):
    def setUp(self):
        Animal.objects.create(name="lion", sound="roar")
        Animal.objects.create(name="cat", sound="meow")

    def test_animals_can_speak(self):
        """Animals that can speak are correctly identified"""
        lion = Animal.objects.get(name="lion")
        cat = Animal.objects.get(name="cat")
        self.assertEqual(lion.speak(), 'The lion says "roar"')
        self.assertEqual(cat.speak(), 'The cat says "meow"')
```

Django Rest Framework also offers some tools to make API testing easier.

```python
from django.urls import reverse
from rest_framework import status
from rest_framework.test import APITestCase
from myproject.apps.core.models import Account

class AccountTests(APITestCase):
    def test_create_account(self):
        """
        Ensure we can create a new account object.
        """
        url = reverse('account-list')
        data = {'username': 'lauren'}
        response = self.client.post(url, data, format='json')
        self.assertEqual(response.status_code, status.HTTP_201_CREATED)
  self.assertEqual(response.data, {'id': 1, 'username': 'lauren'})
        self.assertEqual(Account.objects.count(), 1)
        self.assertEqual(Account.objects.get().username, 'lauren')
```

### Rails

Functional tests for controllers test how an action handles a request and the expected result or response.

```ruby
test "should create article" do
  assert_difference("Article.count") do
    post articles_url, params: { article: { body: "Rails is awesome!", title: "Hello Rails" } }
end

  assert_redirected_to article_path(Article.last)

test "should destroy article" do
  article = articles(:one)
  assert_difference("Article.count", -1) do
    delete article_url(article)
  end

  assert_redirected_to articles_path
end
```

Supports end-to-end testing ("system tests") out of the box with Capybara. This is testing in a headless browser or a real browser instance that allows for interacting with things like screen sizes, responsive layouts, and JavaScript interactions.

```ruby
test "should create Article" do
  visit articles_path

  click_on "New Article"

  fill_in "Title", with: "Creating an Article"
  fill_in "Body", with: "Created this article successfully!"

  click_on "Create Article"

  assert_text "Creating an Article"
end
```

Rails also supports integration testing, which is like a system test, but without the browser instance. You can only interact with the HTML result of the page.

Mailers, jobs, and ActionCables can also be tested.

## Background Jobs

### Django

Celery, a third-party package, is used a lot for this. It's a heavier and more powerful package, but there seem to be some lighter-weight alternatives if your needs aren't so complex, such as Django-q, django-background-tasks, and Django APScheduler.

### Rails

Has the `ActiveJob` framework, which is an interface for working with jobs that can work with different queuing backends.

## Sending Emails

### Django

Provides a `mail` module that has some light wrappers over the Python standard library module called `smtplib`.

```python
from django.core.mail import send_mail

send_mail(
    'Subject here',
    'Here is the message.',
    'from@example.com',
    ['to@example.com'],
    fail_silently=False,
)
```

### Rails

Has the `ActionMailer` framework, which can be used with a variety of backends.

```ruby
class UserMailer < ApplicationMailer
  default from: 'notifications@example.com'

  def welcome_email
    @user = params[:user]
    @url  = 'http://example.com/login'
    mail(to: @user.email, subject: 'Welcome to My Awesome Site')
  end
end
```

HTML views can be created

```html
<!-- welcome_email.html.erb in app/views/user_mailer/ -->
<!DOCTYPE html>
<html>
  <head>
    <meta content='text/html; charset=UTF-8' http-equiv='Content-Type' />
  </head>
  <body>
    <h1>Welcome to example.com, <%= @user.name %></h1>
    <p>
      You have successfully signed up to example.com,
      your username is: <%= @user.login %>.<br>
    </p>
    <p>
      To login to the site, just follow this link: <%= @url %>.
    </p>
    <p>Thanks for joining and have a great day!</p>
  </body>
</html>
```

Corresponding text views can be created

```
<!-- welcome_email.text.erb in app/views/user_mailer/ -->
Welcome to example.com, <%= @user.name %>
===============================================

You have successfully signed up to example.com,
your username is: <%= @user.login %>.

To login to the site, just follow this link: <%= @url %>.

Thanks for joining and have a great day!

```

## Middleware

### Django

Middleware can be functions or classes and are added to the `MIDDLEWARE` array in the settings file.

```python
def simple_middleware(get_response):
    # One-time configuration and initialization.

    def middleware(request):
        # Code to be executed for each request before
        # the view (and later middleware) are called.

        response = get_response(request)

        # Code to be executed for each request/response after
        # the view is called.

        return response

    return middleware


class SimpleMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response
        # One-time configuration and initialization.

    def __call__(self, request):
        # Code to be executed for each request before
        # the view (and later middleware) are called.

        response = self.get_response(request)

        # Code to be executed for each request/response after
        # the view is called.

        return response
```

### Rails

Middleware can be added or removed using the `config.middleware` interface.

```ruby
# config/application.rb

# Push Rack::BounceFavicon at the bottom
config.middleware.use Rack::BounceFavicon

# Add Lifo::Cache after ActionDispatch::Executor.
# Pass { page_cache: false } argument to Lifo::Cache.
config.middleware.insert_after ActionDispatch::Executor, Lifo::Cache, page_cache: false
```

```ruby
# lib/middleware/my_middleware.rb

class MyMiddleware
  def initialize app
    @app = app
  end

  def call env
    # do something...
  end
end
```

## Authentication

### Django

The auth system consists of:

- Users
- Permissions: Binary (yes/no) flags designating whether a user may perform a certain task.
- Groups: A generic way of applying labels and permissions to more than one user.
- A configurable password hashing system
- Forms and view tools for logging in users, or restricting content
- A pluggable backend system

Django Rest Framework provides authentication and authorization for views:

```python
from rest_framework.authentication import SessionAuthentication, BasicAuthentication
from rest_framework.authentication import SessionAuthentication, BasicAuthentication
from rest_framework.response import Response
from rest_framework.views import APIView

class ExampleView(APIView):
    authentication_classes = [SessionAuthentication, BasicAuthentication]

    def get(self, request, format=None):
        content = {
            'user': str(request.user),  # `django.contrib.auth.User` instance.
            'auth': str(request.auth),  # None
        }
        return Response(content)
```

### Rails

Includes basic, digest, and token auth, but there doesn't seem to be much documentation on how to use them.

Popular third-party solutions exist, such as devise, although it only handles HTTP basic auth if you're using Rails API mode.

## Authorization

### Django

Built into the framework. Permissions can be set per type of object and per specific object.

Permissions can be set on a user level or on a group level, and users can be be a part of many groups.

### Rails

Popular third-party solutions exist, such as cancancan.
