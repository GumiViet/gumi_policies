# Contents
---

[TOC]

---
## Rspec Rule
#### Rule
* Controller: Unit test - test actions in the controller (Ex: index, show, new, create, ...)  
* Model: Test data that stored into database passed validation 
* Feature: Behavior test - run a series of steps to return the expect result
	
	```
	Ex: Login feature. The test runs as follow.
	 	1. Go to login path
	 	2. Fill the email and password
	 	3. Click the login button
	 	4. Expect result show alert login successful 
	``` 
 
#### 1. How to describe your methods
Be clear about what method you are describing. For instance, use the Ruby documentation convention of . (or ::) when referring to a class method's name and # when referring to an instance method's name.

```
# BAD
describe 'the authenticate method for User' do
describe 'if the user is an admin' do

# GOOD
describe '.authenticate' do
describe '#admin?' do
```

#### 2. Use contexts
Contexts are a powerful method to make your tests clear and well organized. In the long term this practice will keep tests easy to read.

```
# BAD
it 'has 200 status code if logged in' do
  expect(response).to respond_with 200
end
it 'has 401 status code if not logged in' do
  expect(response).to respond_with 401
end

# GOOD
context 'when logged in' do
  it { is_expected.to respond_with 200 }
end
context 'when logged out' do
  it { is_expected.to respond_with 401 }
end
```

When describing a context, start its description with "when" or "with".

#### 3. Keep your description short
A spec description should never be longer than 40 characters. If this happens you should split it using a context.

```
# BAD
it 'has 422 status code if an unexpected params will be added' do

# GOOD
context 'when not valid' do
  it { is_expected.to respond_with 422 }
end
```

In the example we removed the description related to the status code, which has been replaced by the expectation it { is_expected.to respond_with 422 }. If you run this test typing rspec filename you will obtain a readable output.

```
# FORMATTED OUTPUT
when not valid
  it should respond with 422
```

#### 4. Single expectation test
The 'one expectation' tip is more broadly expressed as 'each test should make only one assertion'. This helps you on finding possible errors, going directly to the failing test, and to make your code readable.

In isolated unit specs, you want each example to specify one (and only one) behavior. Multiple expectations in the same example are a signal that you may be specifying multiple behaviors.

Anyway, in tests that are not isolated (e.g. ones that integrate with a DB, an external webservice, or end-to-end-tests), you take a massive performance hit to do the same setup over and over again, just to set a different expectation in each test. In these sorts of slower tests, I think it's fine to specify more than one isolated behavior.

```
# GOOD (ISOLATED)
it { is_expected.to respond_with_content_type(:json) }
it { is_expected.to assign_to(:resource) }

# GOOD (NOT ISOLATED)
it 'creates a resource' do
  expect(response).to respond_with_content_type(:json)
  expect(response).to assign_to(:resource)
end
```

#### 5. Test all possible cases
Testing is a good practice, but if you do not test the edge cases, it will not be useful. Test valid, edge and invalid case. For example, consider the following action.

```
# DESTROY ACTION
before_filter :find_owned_resources
before_filter :find_resource

def destroy
  render 'show'
  @consumption.destroy
end
```

The error I usually see lies in testing only whether the resource has been removed. But there are at least two edge cases: when the resource is not found and when it's not owned. As a rule of thumb think of all the possible inputs and test them.

```
# BAD
it 'shows the resource'

# GOOD
describe '#destroy' do

  context 'when resource is found' do
    it 'responds with 200'
    it 'shows the resource'
  end

  context 'when resource is not found' do
    it 'responds with 404'
  end

  context 'when resource is not owned' do
    it 'responds with 404'
  end
end
```

#### 6. Expect vs Should syntax
On new projects always use the expect syntax.

```
# BAD
it 'creates a resource' do
  response.should respond_with_content_type(:json)
end

# GOOD
it 'creates a resource' do
  expect(response).to respond_with_content_type(:json)
end
```

Configure the Rspec to only accept the new syntax on new projects, to avoid having the 2 syntax all over the place.

```
# GOOD
# spec_helper.rb
RSpec.configure do |config|
  # ...
  config.expect_with :rspec do |c|
    c.syntax = :expect
  end
end
```

On one line expectations or with implicit subject we should use is_expected.to.

```
# BAD
context 'when not valid' do
  it { should respond_with 422 }
end

# GOOD
context 'when not valid' do
  it { is_expected.to respond_with 422 }
end
```

On old projects you can use the [transpec](https://github.com/yujinakayama/transpec) to convert them to the new syntax.
More about the new Rspec expectation syntax can be found [here](http://myronmars.to/n/dev-blog/2012/06/rspecs-new-expectation-syntax) and [here](http://myronmars.to/n/dev-blog/2013/07/the-plan-for-rspec-3#what_about_the_old_expectationmock_syntax).

#### 7. Use subject
If you have several tests related to the same subject use subject{} to DRY them up.

```
# BAD
it { expect(assigns('message')).to match /it was born in Belville/ }

# GOOD
subject { assigns('message') }
it { is_expected.to match /it was born in Billville/ }
```

RSpec has also the ability to use a named subject.

```
# GOOD
subject(:hero) { Hero.first }
it "carries a sword" do
  expect(hero.equipment).to include "sword"
end
```

Learn more about [rspec subject](https://www.relishapp.com/rspec/rspec-core/v/2-11/docs/subject).

#### 8. Use let and let!
When you have to assign a variable instead of using a before block to create an instance variable, use let. Using let the variable lazy loads only when it is used the first time in the test and get cached until that specific test is finished. A really good and deep description of what let does can be found in this [stackoverflow answer](http://stackoverflow.com/questions/5359558/when-to-use-rspec-let/5359979#5359979/).

```
# BAD
describe '#type_id' do
  before { @resource = FactoryGirl.create :device }
  before { @type     = Type.find @resource.type_id }

  it 'sets the type_id field' do
    expect(@resource.type_id).to equal(@type.id)
  end
end

# GOOD
describe '#type_id' do
  let(:resource) { FactoryGirl.create :device }
  let(:type)     { Type.find resource.type_id }

  it 'sets the type_id field' do
    expect(resource.type_id).to equal(type.id)
  end
end
```

Use let to initialize actions that are lazy loaded to test your specs.

```
# GOOD
context 'when updates a not existing property value' do
  let(:properties) { { id: Settings.resource_id, value: 'on'} }

  def update
    resource.properties = properties
  end

  it 'raises a not found error' do
    expect { update }.to raise_error Mongoid::Errors::DocumentNotFound
  end
end
```

Use let! if you want to define the variable when the block is defined. This can be useful to populate your database to test queries or scopes.

Here an example of what let actually is.

```
# GOOD
# this:
let(:foo) { Foo.new }

# is very nearly equivalent to this:
def foo
  @foo ||= Foo.new
end
```

Learn more about [rspec let](https://www.relishapp.com/rspec/rspec-core/v/2-11/docs/helper-methods/let-and-let).

#### 9. Mock or not to mock
There's a debate going on. Do not (over)use mocks and test real behavior when possible. Testing real cases are useful when updating your application flow.

```
# GOOD
# simulate a not found resource
context "when not found" do
  before { allow(Resource).to receive(:where).with(created_from: params[:id]).and_return(false) }
  it { is_expected.to respond_with 404 }
end
```

Mocking makes your specs faster but they are difficult to use. You need to understand them well to use them well. Read more [about](http://myronmars.to/n/dev-blog/2012/06/thoughts-on-mocking).

#### 10. Create only the data you need
If you have ever worked in a medium size project (but also in small ones), test suites can be heavy to run. To solve this problem, it's important not to load more data than needed. Also, if you think you need dozens of records, you are probably wrong.

```
# GOOD
describe "User" do
  describe ".top" do
    before { FactoryGirl.create_list(:user, 3) }
    it { expect(User.top(2)).to have(2).item }
  end
end
```

#### 11. Use factories and not fixtures
This is an old topic, but it's still good to remember it. Do not use fixtures because they are difficult to control, use factories instead. Use them to reduce the verbosity on creating new data.

```
# BAD
user = User.create(
  name: 'Genoveffa',
  surname: 'Piccolina',
  city: 'Billyville',
  birth: '17 Agoust 1982',
  active: true
)

# GOOD
user = FactoryGirl.create :user
```

One important note. When talking about unit tests the best practice would be to use neither fixtures or factories. Put as much of your domain logic in libraries that can be tested without needing complex, time consuming setup with either factories or fixtures. Read more in [this article](http://blog.steveklabnik.com/posts/2012-07-14-why-i-don-t-like-factory_girl)

Learn more about [Factory Girl](https://github.com/thoughtbot/factory_girl).

#### 12. Easy to read matcher
Use readable matchers and double check the available [rspec matchers](https://www.relishapp.com/rspec/rspec-expectations/docs/built-in-matchers).

```
# BAD
lambda { model.save! }.to raise_error Mongoid::Errors::DocumentNotFound

# GOOD
expect { model.save! }.to raise_error Mongoid::Errors::DocumentNotFound
```

#### 13. Shared Examples
Making tests is great and you get more confident day after day. But in the end you will start to see code duplication coming up everywhere. Use shared examples to DRY your test suite up.

```
# BAD
describe 'GET /devices' do
  let!(:resource) { FactoryGirl.create :device, created_from: user.id }
  let(:uri) { '/devices' }

  context 'when shows all resources' do
    let!(:not_owned) { FactoryGirl.create factory }

    it 'shows all owned resources' do
      page.driver.get uri
      expect(page.status_code).to be(200)
      contains_owned_resource resource
      does_not_contain_resource not_owned
    end
  end

  describe '?start=:uri' do
    it 'shows the next page' do
      page.driver.get uri, start: resource.uri
      expect(page.status_code).to be(200)
      contains_resource resources.first
      expect(page).to_not have_content resource.id.to_s
    end
  end
end

# GOOD
describe 'GET /devices' do

  let!(:resource) { FactoryGirl.create :device, created_from: user.id }
  let(:uri)       { '/devices' }

  it_behaves_like 'a listable resource'
  it_behaves_like 'a paginable resource'
  it_behaves_like 'a searchable resource'
  it_behaves_like 'a filterable list'
end
```

In our experience, shared examples are used mainly for controllers. Since models are pretty different from each other, they (usually) do not share much logic.

Learn more about [rspec shared examples](https://www.relishapp.com/rspec/rspec-core/v/2-11/docs/example-groups/shared-examples).

#### 14. Test what you see
Deeply test your models and your application behaviour (integration tests). Do not add useless complexity testing controllers.

When I first started testing my apps I was testing controllers, now I don't. Now I only create integration tests using RSpec and Capybara. Why? Because I truly believe that you should test what you see and because testing controllers is an extra step you don't need. You'll find out that most of your tests go into the models and that integration tests can be easily grouped into shared examples, building a clear and readable test suite.

This is an open debate in the Ruby community and both sides have good arguments supporting their idea. People supporting the need of testing controllers will tell you that your integration tests don't cover all use cases and that they are slow.

Both are wrong. You can easily cover all use cases (why shouldn't you?) and you can run single file specs using automated tools like Guard. In this way you will run only the specs you need to test blazing fast without stopping your flow.

#### 15. Don't use should
Do not use should when describing your tests. Use the third person in the present tense. Even better start using the new [expectation](http://myronmars.to/n/dev-blog/2012/06/rspecs-new-expectation-syntax) syntax.

```
# BAD
it 'should not change timings' do
  consumption.occur_at.should == valid.occur_at
end

# GOOD
it 'does not change timings' do
  expect(consumption.occur_at).to equal(valid.occur_at)
end
```

See [the should_not gem](https://github.com/should-not/should_not) for a way to enforce this in RSpec and [the should_clean](https://github.com/siyelo/should_clean) gem for a way to clean up existing RSpec examples that begin with "should."

#### 17. Stubbing HTTP requests
Sometimes you need to access external services. In these cases you can't rely on the real service but you should stub it with solutions like webmock.

```
# GOOD
context "with unauthorized access" do
  let(:uri) { 'http://api.lelylan.com/types' }
  before    { stub_request(:get, uri).to_return(status: 401, body: fixture('401.json')) }
  it "gets a not authorized notification" do
    page.driver.get uri
    expect(page).to have_content 'Access denied'
  end
end
```

Learn more about [webmock](https://github.com/bblimke/webmock) and [VCR](https://github.com/vcr/vcr). Here a [nice presentation](http://marnen.github.com/webmock-presentation/webmock.html) explaining how to mix them together.

