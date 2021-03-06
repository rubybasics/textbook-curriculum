# Unit Testing Models
## Learning Goals
- Get your rails app setup to do testing with minitest-spec
- Write some unit tests using the Rails testing DSL
- Be comforted knowing it's Minitest under the hood, and we know how to Minitest
- Talk about how to test Active Record models
- Get some basic strategies for where to start with testing
- Acknowledge that _fixtures_ are equal parts cool and weird.

## Testing Active Record Models
Figuring out what to test can be really confusing. You'll develop a sense of what needs tested as you gain experience and exposure, but we can at least set you up with guidelines:

- Write at least one test for each _validation_ on a model
- Write at least one test for each _custom method_ on a model
- Write at least one test for each _model relationship_ on a model
- Write at least one test for each _scope_ on a model (we'll talk about scopes next week)

We say _at least one_ test because it often makes sense to test different combinations of information. For example, with _validations_ we like to write a test that verifies what kind of data is valid and a separate test that provides an example of invalid data. We write both a _positive_ and _negative_ test case.  Look to test edge-cases and boundaries between when a model becomes valid and invalid.  For example you should test normal looking data for validity, but also test instances with the minimal number of fields.  For example an Album must have a title and must be linked to an artist, but all other fields are optional.  So test an instance with only an artist and title as well as normal amounts of data.  An Album price should never be negative, so test instances where it's positive, negative and zero.  

### Constructing Test Cases
Let's say our model looks like this:

```ruby
class Album < ActiveRecord::Base
  belongs_to :artist
  validates :title, presence: true, uniqueness: true
end
```

Just in this small bit of code, we see three or four things to test. First are the validations. Writing a _positive_ and _negative_ test case for each validation is four or so tests. Then there's the `belongs_to` relationship. We should test that too. It gets a positive test--an associated `Artist` is defined--and a negative test--it should still be able to save/retrieve/etc. even if an `Artist` isn't associated. If we go over to `test/models`, we should find an `album_test.rb` file.

It was created for us when we generated the model. If the models were created prior to setting up Minitest Spec, it'll look like this:

```ruby
require 'test_helper'

class AlbumTest < ActiveSupport::TestCase
  # test "the truth" do
  #   assert true
  # end
end
```
Otherwise it new models will generate with spec style tests like this:

```ruby
require "test_helper"

describe Album do
  let(:album) { Album.new }

  it "must be valid" do
    value(album).must_be :valid?
  end
end
```

`ActiveSupport::TestCase` is the DSL Rails put on top of Minitest. We can follow the pattern shown in the comment to write our first test. The `it` method takes two parameters. The first is the test name/description, and the second is a block where we'll define our _expectations_.

```ruby
require 'test_helper'

describe Album do
  let (:empty_album) {Album.new}

  it "An album title can't be blank" do
    empty_album.wont_be :valid?
    empty_album.errors.keys.must_include :title, "Title is not in the errors hash"
  end
end
```

Otherwise Rails will generate spec style tests like below.  We can replace the assert-style tests by deleting the original file and regenerating the model, or by replacing the text with describe & it blocks.

```ruby
require "test_helper"

describe Album do
  let(:album) { Album.new }

  it "must be valid" do
    value(album).must_be :valid?
  end
end
```

There's a couple things to note in the above example. First and foremost is that we are invoking our model directly with `Album.new`. This test file has scope to the full application due to how things are structured in the `test_helper` and parent class.

Also check our _expectations_. They look like Minitest _expectations_ because they are. [There's a bunch of available _expectations_](http://ruby-doc.org/stdlib-trunk/libdoc/minitest/spec/rdoc/MiniTest/Expectations.html), but I get most of my testing done with just a small set:

- must\_be(expression, fail_message)
- wont\_be(expression, fail_message)
- must\_equal(expr1, expr2, fail_message)
- must\_include(collection, object, fail_message)
- wont\_be\_nil(expression, fail_message)

__Question: Is the test above a _positive_ or _negative_ test? What would the other test look like?__

## Running tests
[The Rails Guide on testing](http://guides.rubyonrails.org/testing.html#the-rails-test-runner) has a specific section for how to run tests that's definitely worth reading. The short version is that from the project root, we can run all of our tests with `rails test`. If you want to run just the model tests, run `rails test test/models`. The output will probably look pretty familiar by now:

```bash
$ rails test test/models

# Running tests with run options --seed 56340:

.........

Finished tests in 0.097170s, 92.6214 tests/s, 92.6214 assertions/s.


9 tests, 9 assertions, 0 failures, 0 errors, 0 skips
```

## Creating Test Data With _Fixtures_
Writing tests for objects that interact with a database often involves test data. In Rails, we define _fixtures_--temporary data used to populate models in tests--for test data. _Fixtures_ are kept in `test/fixtures` and are defined as [YAML](http://yaml.org/) files.

Each YAML file defines default data for one model. So we'd use `test/fixtures/artists.yml` to create some test data for use when testing `Artist` models. Here's what YAML looks like:

```yml
the_heavy:
  name: The Heavy
the_clash:
  name: The Clash
tmonk:
  name: Thelonious Monk
```

YML is a set of key/value pairs separated by a colon. It's also __white space sensitive__. It knows how to nest key/value pairs based on their indentation, so pay close attention to your formatting. In the example above, we define three sets of data with keys `the_heavy`, `the_clast`, and `tmonk`. Each of those keys has a value which is a second set of key/value pairs (like a hash of hashes). Because our `Artist` model doesn't track much data, the _fixtures_ are pretty small. Let's take a look at another set of _fixtures_, this time for the `Album` model:

```yml
combat_rock:
  title: Combat Rock
  artist: the_clash
  released: 1982
  label: Epic
  label_code: FE 37689
  format: LP, Album
house_that_dirt_built:
  title: The House That Dirt Built
  artist: the_heavy
  released: 2009
  label: Counter Records
  label_code: COUNT028
  format: LP, Album
```

In this example, we define two data sets, each representing one `Album`. Each _fixture_ contains quite a bit of data. Take a look at the values for the `artist` key in each _fixture_. Because we're using Rails, and because Rails knows that an `Album` `belongs_to` an `Artist`, and because we're defining our test data with _fixtures_, we can __reference other _fixture_ data in related models by key__. So `artist: the_heavy` will connect the _fixture_ `house_that_dirt_built` with the `Artist` _fixture_ `the_heavy`. This connection will be critical when we begin testing model relationships.

### Managing The Test Database
So now you've got this test data, what do you do with it? Short answer: put it in the test database. When Rails runs tests, it does so in the __test environment__. This means that we can configure Rails to do things like load different gems or use a different database when running tests. Using a different database is important because we don't want to impact the development database by writing potentially invalid or corrupt data.

The test database is meant to be transient. By default, Rails will reset the test database _between every test_. Data saved to the database in one test won't exist in other tests. The exception is fixture data, which is always available in every test. However, any changes to fixture data in a test will not be preserved into the next test.

Use `rails db:test:prepare` if the test database seems to be stuck in a broken state. It will reset the test database, run any pending migrations, and re-seed the fixture data. Very handy!

## Putting it all together
Ok, let's test a model relationship for the final example. We'll use the fixture data from the previous examples. Here's our models:

```ruby
# app/models/artist.rb
class Artist < ActiveRecord::Base
  has_many :albums
  validates :name, presence: true, uniqueness: true
end

# app/models/albums.rb
class Album < ActiveRecord::Base
  belongs_to :album
  validates :name, presence: true, uniqueness: true
end
```

We want to test that an instance of `Artist` can retrieve the associated `Album` instances. In the test, we'll use the _fixture_ data to confirm that, starting with an `Artist` (that I know should have an associated artist), we can get their associated `Album` instances. We'll use `assert_include` to verify that the `Album` object (from the _fixture_ data) is in the returned collection:

```ruby
# test/models/artist_test.rb
require 'test_helper'

describe Artist
  it "Artists can have albums" do
    artists(:the_clash).albums.must_include albums(:combat_rock)
  end
end
```

## References
- [Minitest Quick Reference](http://www.mattsears.com/articles/2011/12/10/minitest-quick-reference/)
- [Minitest Expectations](http://ruby-doc.org/stdlib-trunk/libdoc/minitest/spec/rdoc/MiniTest/Expectations.html)
- [Adding Color to Minitest Output](http://chriskottom.com/blog/2014/06/dress-up-your-minitest-output/)
- [Ruby on Rails Guide - Model Testing](http://guides.rubyonrails.org/testing.html#model-testing)
-  [Minitest Rails Spec Documentation](http://blowmage.com/minitest-rails/)]
- [Minitest Model Testing for Beginners](http://buildingrails.com/a/rails_unit_testing_with_minitest_for_beginners)
