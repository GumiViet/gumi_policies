# Contents
---

[TOC]

---

## Coding convention

### The Rails Style Guide
- This Rails style guide recommends best practices so that real-world Rails programmers can write code that can be maintained by other real-world Rails programmers. A style guide that reflects real-world usage gets used, and a style guide that holds to an ideal that has been rejected by the people it is supposed to help risks not getting used at all – no matter how good it is.

- The guide is separated into several sections of related rules. I've tried to add the rationale behind the rules (if it's omitted I've assumed it's pretty obvious).

- I didn't come up with all the rules out of nowhere - they are mostly based on my extensive career as a professional software engineer, feedback and suggestions from members of the Rails community and various highly regarded Rails programming resources.

#### 1. Configuration
- Put custom initialization code in `config/initializers`. The code in initializers executes on application startup.

- Keep initialization code for each gem in a separate file with the same name as the gem, for example carrierwave.rb, `active_admin.rb`, etc.

- Adjust accordingly the settings for development, test and production environment (in the corresponding files under `config/environments/`)

	- Mark additional assets for precompilation (if any):
```
# config/environments/production.rb
# Precompile additional assets (application.js, application.css,
#and all non-JS/CSS are already added)
config.assets.precompile += %w( rails_admin/rails_admin.css rails_admin/rails_admin.js )
```
- Keep configuration that's applicable to all environments in the `config/application.rb` file.

- Create an additional `staging` environment that closely resembles the `production` one.

- Keep any additional configuration in YAML files under the `config/` directory.

- Since Rails 4.2 YAML configuration files can be easily loaded with the new `config_for` method:
```
Rails::Application.config_for(:yaml_file)
```

#### 2. Routing
- When you need to add more actions to a RESTful resource (do you really need them at all?) use `member` and `collection` routes.

```
# bad
get 'subscriptions/:id/unsubscribe'
resources :subscriptions

# good
resources :subscriptions do
  get 'unsubscribe', on: :member
end

# bad
get 'photos/search'
resources :photos

# good
resources :photos do
  get 'search', on: :collection
end
```
- If you need to define multiple `member/collection` routes use the alternative block syntax.

```
resources :subscriptions do
  member do
    get 'unsubscribe'
    # more routes
  end
end

resources :photos do
  collection do
    get 'search'
    # more routes
  end
end
```
- Use nested routes to express better the relationship between ActiveRecord models.

```
class Post < ActiveRecord::Base
  has_many :comments
end

class Comments < ActiveRecord::Base
  belongs_to :post
end

# routes.rb
resources :posts do
  resources :comments
end
```
- If you need to nest routes more than 1 level deep then use the `shallow: true` option. This will save user from long urls `posts/1/comments/5/versions/7/edit` and you from long url helpers `edit_post_comment_version`.

```
resources :posts, shallow: true do
  resources :comments do
    resources :versions
  end
end
```
- Use namespaced routes to group related actions. [link]

```
namespace :admin do
  # Directs /admin/products/* to Admin::ProductsController
  # (app/controllers/admin/products_controller.rb)
  resources :products
end
```
- Never use the legacy wild controller route. This route will make all actions in every controller accessible via GET requests. [link]

```
# very bad
match ':controller(/:action(/:id(.:format)))'
```
- Don't use match to define any routes unless there is need to map multiple request types among `[:get, :post, :patch, :put, :delete]` to a single action using :via option.

#### 3. Controllers
- Keep the controllers skinny - they should only retrieve data for the view layer and shouldn't contain any business logic (all the business logic should naturally reside in the model).

- Each controller action should (ideally) invoke only one method other than an initial find or new.

- Share no more than two instance variables between a controller and a view.

#### 4. Rendering
- Prefer using a template over inline rendering.

```
# very bad
class ProductsController < ApplicationController
  def index
    render inline: "<% products.each do |p| %><p><%= p.name %></p><% end %>", type: :erb
  end
end

# good
## app/views/products/index.html.erb
<%= render partial: 'product', collection: products %>

## app/views/products/_product.html.erb
<p><%= product.name %></p>
<p><%= product.price %></p>

## app/controllers/foo_controller.rb
class ProductsController < ApplicationController
  def index
    render :index
  end
end
```
- Prefer `render plain:` over `render text:`.

```
# bad - sets MIME type to `text/html`
...
render text: 'Ruby!'
...

# bad - requires explicit MIME type declaration
...
render text: 'Ruby!', content_type: 'text/plain'
...

# good - short and precise
...
render plain: 'Ruby!'
...
```
- Prefer corresponding symbols to numeric HTTP status codes. They are meaningful and do not look like "magic" numbers for less known HTTP status codes.

```
# bad
...
render status: 500
...

# good
...
render status: :forbidden
...
```

#### 5. Models
- Introduce non-ActiveRecord model classes freely.

- Name the models with meaningful (but short) names without abbreviations.

- If you need model objects that support ActiveRecord behavior (like validation) without the ActiveRecord database functionality use the ActiveAttr gem.

```
class Message
  include ActiveAttr::Model

  attribute :name
  attribute :email
  attribute :content
  attribute :priority

  attr_accessible :name, :email, :content

  validates :name, presence: true
  validates :email, format: { with: /\A[-a-z0-9_+\.]+\@([-a-z0-9]+\.)+[a-z0-9]{2,4}\z/i }
  validates :content, length: { maximum: 500 }
end
```
For a more complete example refer to the RailsCast on the subject.

- Unless they have some meaning in the business domain, do not put methods in your model that just format your data (like code generating HTML). These methods are most likely going to be called from the view layer only, so their place is in helpers. Keep your models for business logic and data-persistence only.

#### 6. ActiveRecord
- Avoid altering ActiveRecord defaults (table names, primary key, etc) unless you have a very good reason (like a database that is not under your control).

```
# bad - do not do this if you can modify the schema
class Transaction < ActiveRecord::Base
  self.table_name = 'order'
  ...
end
```
- Group macro-style methods (`has_many`, `validates`, etc) in the beginning of the class definition.

```
class User < ActiveRecord::Base
  # keep the default scope first (if any)
  default_scope { where(active: true) }

  # constants come up next
  COLORS = %w(red green blue)

  # afterwards we put attr related macros
  attr_accessor :formatted_date_of_birth

  attr_accessible :login, :first_name, :last_name, :email, :password

  # Rails4+ enums after attr macros, prefer the hash syntax
  enum gender: { female: 0, male: 1 }

  # followed by association macros
  belongs_to :country

  has_many :authentications, dependent: :destroy

  # and validation macros
  validates :email, presence: true
  validates :username, presence: true
  validates :username, uniqueness: { case_sensitive: false }
  validates :username, format: { with: /\A[A-Za-z][A-Za-z0-9._-]{2,19}\z/ }
  validates :password, format: { with: /\A\S{8,128}\z/, allow_nil: true }

  # next we have callbacks
  before_save :cook
  before_save :update_username_lower

  # other macros (like devise'ss) should be placed after the callbacks

  ...
end
```

- Prefer `has_many :through` to `has_and_belongs_to_many`. Using `has_many :through` allows additional attributes and validations on the join model.

```
# not so good - using has_and_belongs_to_many
class User < ActiveRecord::Base
  has_and_belongs_to_many :groups
end

class Group < ActiveRecord::Base
  has_and_belongs_to_many :users
end

# preferred way - using has_many :through
class User < ActiveRecord::Base
  has_many :memberships
  has_many :groups, through: :memberships
end

class Membership < ActiveRecord::Base
  belongs_to :user
  belongs_to :group
end

class Group < ActiveRecord::Base
  has_many :memberships
  has_many :users, through: :memberships
end
```

- Prefer `self[:attribute]` over `read_attribute(:attribute)`.

```
# bad
def amount
  read_attribute(:amount) * 100
end

# good
def amount
  self[:amount] * 100
end
```

- Prefer `self[:attribute] = value` over `write_attribute(:attribute, value)`.

```
# bad
def amount
  write_attribute(:amount, 100)
end

# good
def amount
  self[:amount] = 100
end
```

- Always use the new "sexy" validations.

```
# bad
validates_presence_of :email
validates_length_of :email, maximum: 100

# good
validates :email, presence: true, length: { maximum: 100 }
```

- When a custom validation is used more than once or the validation is some regular expression mapping, create a custom validator file.

```
# bad
class Person
  validates :email, format: { with: /\A([^@\s]+)@((?:[-a-z0-9]+\.)+[a-z]{2,})\z/i }
end

# good
class EmailValidator < ActiveModel::EachValidator
  def validate_each(record, attribute, value)
    record.errors[attribute] << (options[:message] || 'is not a valid email') unless value =~ /\A([^@\s]+)@((?:[-a-z0-9]+\.)+[a-z]{2,})\z/i
  end
end

class Person
  validates :email, email: true
end
```

- Keep custom validators under `app/validators`.

- Consider extracting custom validators to a shared gem if you're maintaining several related apps or the validators are generic enough.

- Use named scopes freely.

```
class User < ActiveRecord::Base
  scope :active, -> { where(active: true) }
  scope :inactive, -> { where(active: false) }

  scope :with_orders, -> { joins(:orders).select('distinct(users.id)') }
end
```

- When a named scope defined with a lambda and parameters becomes too complicated, it is preferable to make a class method instead which serves the same purpose of the named scope and returns an `ActiveRecord::Relation` object. Arguably you can define even simpler scopes like this.

```
class User < ActiveRecord::Base
  def self.with_orders
    joins(:orders).select('distinct(users.id)')
  end
end
```

- Beware of the behavior of the `update_attribute` method. It doesn't run the model validations (unlike `update_attributes`) and could easily corrupt the model state.

- Use user-friendly URLs. Show some descriptive attribute of the model in the URL rather than its `id`. There is more than one way to achieve this:

	- Override the `to_param` method of the model. This method is used by Rails for constructing a URL to the object. The default implementation returns the `id` of the record as a String. It could be overridden to include another human-readable attribute.

  ```
class Person
    def to_param
      "#{id} #{name}".parameterize
    end
  end
```

In order to convert this to a URL-friendly value, `parameterize` should be called on the string. The id of the object needs to be at the beginning so that it can be found by the `find` method of ActiveRecord.

	- Use the `friendly_id` gem. It allows creation of human-readable URLs by using some descriptive attribute of the model instead of its `id`.

```
class Person
    extend FriendlyId
    friendly_id :name, use: :slugged
  end
```

Check the gem documentation for more information about its usage.

- Use `find_each` to iterate over a collection of AR objects. Looping through a collection of records from the database (using the `all` method, for example) is very inefficient since it will try to instantiate all the objects at once. In that case, batch processing methods allow you to work with the records in batches, thereby greatly reducing memory consumption.

```
# bad
Person.all.each do |person|
  person.do_awesome_stuff
end

Person.where('age > 21').each do |person|
  person.party_all_night!
end

# good
Person.find_each do |person|
  person.do_awesome_stuff
end

Person.where('age > 21').find_each do |person|
  person.party_all_night!
end
```

- Since Rails creates callbacks for dependent associations, always call `before_destroy` callbacks that perform validation with `prepend: true`.

```
# bad (roles will be deleted automatically even if super_admin? is true)
has_many :roles, dependent: :destroy

before_destroy :ensure_deletable

def ensure_deletable
  fail "Cannot delete super admin." if super_admin?
end

# good
has_many :roles, dependent: :destroy

before_destroy :ensure_deletable, prepend: true

def ensure_deletable
  fail "Cannot delete super admin." if super_admin?
end
```

- Define the `dependent` option to the `has_many` and `has_one` associations.

```
# bad
class Post < ActiveRecord::Base
  has_many :comments
end

# good
class Post < ActiveRecord::Base
  has_many :comments, dependent: :destroy
end
```

- When persisting AR objects always use the exception raising bang! method or handle the method return value. This applies to `create`, `save`, `update`, `destroy`, `first_or_create` and `find_or_create_by`.

```
# bad
user.create(name: 'Bruce')

# bad
user.save

# good
user.create!(name: 'Bruce')
# or
bruce = user.create(name: 'Bruce')
if bruce.persisted?
  ...
else
  ...
end

# good
user.save!
# or
if user.save
  ...
else
  ...
end
```

#### 7. ActiveRecord Queries
- Avoid string interpolation in queries, as it will make your code susceptible to SQL injection attacks.

```
# bad - param will be interpolated unescaped
Client.where("orders_count = #{params[:orders]}")

# good - param will be properly escaped
Client.where('orders_count = ?', params[:orders])
```

- Consider using named placeholders instead of positional placeholders when you have more than 1 placeholder in your query.

```
# okish
Client.where(
  'created_at >= ? AND created_at <= ?',
  params[:start_date], params[:end_date]
)

# good
Client.where(
  'created_at >= :start_date AND created_at <= :end_date',
  start_date: params[:start_date], end_date: params[:end_date]
)
```

- Favor the use of `find` over `where` when you need to retrieve a single record by id.

```
# bad
User.where(id: id).take

# good
User.find(id)
```

- Favor the use of `find_by` over `where` and `find_by_attribute` when you need to retrieve a single record by some attributes.

```
# bad
User.where(first_name: 'Bruce', last_name: 'Wayne').first

# bad
User.find_by_first_name_and_last_name('Bruce', 'Wayne')

# good
User.find_by(first_name: 'Bruce', last_name: 'Wayne')
```

- Favor the use of `where.not` over SQL.

```
# bad
User.where("id != ?", id)

# good
User.where.not(id: id)
```

- When specifying an explicit query in a method such as `find_by_sql`, use heredocs with `squish`. This allows you to legibly format the SQL with line breaks and indentations, while supporting syntax highlighting in many tools (including GitHub, Atom, and RubyMine).

```
User.find_by_sql(<<-SQL.squish)
  SELECT
    users.id, accounts.plan
  FROM
    users
  INNER JOIN
    accounts
  ON
    accounts.user_id = users.id
  # further complexities...
SQL
```

String#squish removes the indentation and newline characters so that your server log shows a fluid string of SQL rather than something like this:

```
SELECT\n    users.id, accounts.plan\n  FROM\n    users\n  INNER JOIN\n    acounts\n  ON\n    accounts.user_id = use
```

#### 8. Views
- Never call the model layer directly from a view.

- Never make complex formatting in the views, export the formatting to a method in the view helper or the model.

- Mitigate code duplication by using partial templates and layouts.

#### 9. Internationalization
- No strings or other locale specific settings should be used in the views, models and controllers. These texts should be moved to the locale files in the `config/locales` directory.

- When the labels of an ActiveRecord model need to be translated, use the activerecord scope:

```
en:
  activerecord:
    models:
      user: Member
    attributes:
      user:
        name: 'Full name'
```

Then `User.model_name.human` will return "Member" and `User.human_attribute_name("name")` will return "Full name". These translations of the attributes will be used as labels in the views.

- Separate the texts used in the views from translations of ActiveRecord attributes. Place the locale files for the models in a folder `locales/models` and the texts used in the views in folder `locales/views`.

	- When organization of the locale files is done with additional directories, these directories must be described in the `application.rb` file in order to be loaded.

```
# config/application.rb
  config.i18n.load_path += Dir[Rails.root.join('config', 'locales', '**', '*.{rb,yml}')]
```
- Place the shared localization options, such as date or currency formats, in files under the root of the `locales` directory.

- Use the short form of the I18n methods: `I18n.t` instead of `I18n.translate` and `I18n.l` instead of `I18n.localize`.

- Use "lazy" lookup for the texts used in views. Let's say we have the following structure:

```
en:
  users:
    show:
      title: 'User details page'
```

The value for `users.show.title` can be looked up in the template `app/views/users/show.html.haml` like this:

```
= t '.title'
```

- Use the dot-separated keys in the controllers and models instead of specifying the `:scope` option. The dot-separated call is easier to read and trace the hierarchy.

```
# bad
I18n.t :record_invalid, scope: [:activerecord, :errors, :messages]

# good
I18n.t 'activerecord.errors.messages.record_invalid'
```

- More detailed information about the Rails I18n can be found in the Rails Guides

#### 10. Assets
Use the assets pipeline to leverage organization within your application.

- Reserve `app/assets` for custom stylesheets, javascripts, or images.

- Use `lib/assets` for your own libraries that don’t really fit into the scope of the application.

- Third party code such as jQuery or bootstrap should be placed in `vendor/assets`.

- When possible, use gemified versions of assets (e.g. jquery-rails, jquery-ui-rails, bootstrap-sass, zurb-foundation).

#### 11. Mailers
- Name the mailers `SomethingMailer`. Without the Mailer suffix it isn't immediately apparent what's a mailer and which views are related to the mailer.

- Provide both HTML and plain-text view templates.

- Enable errors raised on failed mail delivery in your development environment. The errors are disabled by default.

```
# config/environments/development.rb

config.action_mailer.raise_delivery_errors = true
```

- Use a local SMTP server like Mailcatcher in the development environment.

```
# config/environments/development.rb

config.action_mailer.smtp_settings = {
  address: 'localhost',
  port: 1025,
  # more settings
}
```

- Provide default settings for the host name.

```
# config/environments/development.rb
config.action_mailer.default_url_options = { host: "#{local_ip}:3000" }

# config/environments/production.rb
config.action_mailer.default_url_options = { host: 'your_site.com' }

# in your mailer class
default_url_options[:host] = 'your_site.com'
```

- If you need to use a link to your site in an email, always use the `_url`, not `_path` methods. The `_url` methods include the host name and the `_path` methods don't.

```
# bad
You can always find more info about this course
<%= link_to 'here', course_path(@course) %>

# good
You can always find more info about this course
<%= link_to 'here', course_url(@course) %>
```

- Format the from and to addresses properly. Use the following format:

```
# in your mailer class
default from: 'Your Name <info@your_site.com>'
```

- Make sure that the e-mail delivery method for your test environment is set to test:

```
# config/environments/test.rb

config.action_mailer.delivery_method = :test
```

- The delivery method for development and production should be `smtp`:

```
# config/environments/development.rb, config/environments/production.rb

config.action_mailer.delivery_method = :smtp
```

- When sending html emails all styles should be inline, as some mail clients have problems with external styles. This however makes them harder to maintain and leads to code duplication. There are two similar gems that transform the styles and put them in the corresponding html tags: premailer-rails and roadie.

- Sending emails while generating page response should be avoided. It causes delays in loading of the page and request can timeout if multiple email are sent. To overcome this emails can be sent in background process with the help of sidekiq gem.

#### 12. Active Support Core Extensions
- Prefer Ruby 2.3's safe navigation operator `&.` over `ActiveSupport#try!`.

```
# bad
obj.try! :fly

# good
obj&.fly
```

- Prefer Ruby's Standard Library methods over `ActiveSupport` aliases.

```
# bad
'the day'.starts_with? 'th'
'the day'.ends_with? 'ay'

# good
'the day'.start_with? 'th'
'the day'.end_with? 'ay'
```

- Prefer Ruby's Standard Library over uncommon ActiveSupport extensions.

```
# bad
(1..50).to_a.forty_two
1.in? [1, 2]
'day'.in? 'the day'

# good
(1..50).to_a[41]
[1, 2].include? 1
'the day'.include? 'day'
```

- Prefer Ruby's comparison operators over ActiveSupport's `Array#inquiry`, `Numeric#inquiry` and `String#inquiry`.

```
# bad - String#inquiry
ruby = 'two'.inquiry
ruby.two?

# good
ruby = 'two'
ruby == 'two'

# bad - Array#inquiry
pets = %w(cat dog).inquiry
pets.gopher?

# good
pets = %w(cat dog)
pets.include? 'cat'

# bad - Numeric#inquiry
0.positive?
0.negative?

# good
0 > 0
0 < 0
```

#### 13. Time
- Config your timezone accordingly in `application.rb`.

```
config.time_zone = 'Eastern European Time'
# optional - note it can be only :utc or :local (default is :utc)
config.active_record.default_timezone = :local
```

- Don't use `Time.parse`.

```
# bad
Time.parse('2015-03-02 19:05:37') # => Will assume time string given is in the system's time zone.

# good
Time.zone.parse('2015-03-02 19:05:37') # => Mon, 02 Mar 2015 19:05:37 EET +02:00
```

- Don't use `Time.now`.

```
# bad
Time.now # => Returns system time and ignores your configured time zone.

# good
Time.zone.now # => Fri, 12 Mar 2014 22:04:47 EET +02:00
Time.current # Same thing but shorter.
```

#### 14. Bundler
- Put gems used only for development or testing in the appropriate group in the Gemfile.

- Use only established gems in your projects. If you're contemplating on including some little-known gem you should do a careful review of its source code first.

- OS-specific gems will by default result in a constantly changing `Gemfile.lock` for projects with multiple developers using different operating systems. Add all OS X specific gems to a `darwin` group in the Gemfile, and all Linux specific gems to a `linux` group:

```
# Gemfile
group :darwin do
  gem 'rb-fsevent'
  gem 'growl'
end

group :linux do
  gem 'rb-inotify'
end
```
To require the appropriate gems in the right environment, add the following to `config/application.rb`:

```
platform = RUBY_PLATFORM.match(/(linux|darwin)/)[0].to_sym
Bundler.require(platform)
```

- Do not remove the `Gemfile.lock` from version control. This is not some randomly generated file - it makes sure that all of your team members get the same gem versions when they do a `bundle install`.

