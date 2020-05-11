# Sequel::Packer

A Ruby serialization library to be used with the Sequel ORM.

## Installation

Add this line to your application's Gemfile:

```ruby
gem 'sequel-packer'
```

And then execute:

    $ bundle install

Or install it yourself as:

    $ gem install sequel-packer

## Usage

Suppose we have the following basic database schema:

```ruby
DB.create_table(:users) do
  primary_key :id
  String :name
end

DB.create_table(:posts) do
  primary_key :id
  foreign_key :author_id, :users
  String :title
  String :content
end

DB.create_table(:comments) do
  primary_key :id
  foreign_key :author_id, :users
  foreign_key :post_id, :posts
  String :content
end

class User < Sequel::Model(:users); end
class Post < Sequel::Model(:posts); end
class Comment < Sequel::Model(:comments)
  many_to_one :author, key: :author_id, class: :User
end
```

Suppose an endpoint wants to fetch all the ten most recent comments by a user.
After validating the user id, we end up with the Sequel dataset represting the
data we want to return:

```ruby
recent_comments = Comment
  .where(author_id: user_id)
  .order(:id.desc)
  .limit(10)
```

We can define a Packer class to serialize just fields we want to, using a
custom DSL:

```ruby
class CommentPacker < Sequel::Packer
  model Comment

  field :id
  field :content
end
```

This can then be used as follows:

```ruby
CommentPacker.new.pack(recent_comments)
=> [
  {id: 536, "Great post, man!"},
  {id: 436, "lol"},
  {id: 413, "What a story..."},
]
```

In another context, suppose that we want to fetch the comments on a post along
with the authors of those comments.

In this case we could define two separate Packers, one for Users, and another
for Comments:

```ruby
class UserPacker < Sequel::Packer
  model User

  field :id
  field :name
end

class CommentWithAuthorPacker < Sequel::Packer
  model Comment

  field :id
  field :content
  field :author, UserPacker
end
```

When we use this, we get:

```ruby
comments_on_post = Comment.where(post_id: post.id)
CommentWithAuthorPacker.new.pack(comments_on_post)
=> [
  {
    id: 752,
    content: "Great gem",
    author: {id: 382, name: "Jeremy Evans"},
  },
  {
    id: 162,
    content: "Never seen anything like it",
    author: {id: 382, name: "Donald Knuth"},
  },
  ...
]
```

See the API Reference below for the exact API.

## API Reference

Custom packers are written by creating subclasses of `Sequel::Packer`. This
class defines a DSL for declaring how a Sequel Model will be converted into a
plain Ruby hash.

### `self.model(sequel_model_class)`

The beginning of each Packer class must begin with `model MySequelModel`, which
specifies which Sequel Model this Packer class will serialize. This is mostly
to catch certain errors at load time, rather than at run time.

### `self.field(column_name)` (or `self.field(method_name)`)

Defining the shape of the outputted data is done using the `field` method, which
exists in four different variants. This first variant is the simplest. It simply
fetches the value of the column from the model and stores it in the outputted
hash under a key of the same name. Essentially `field :my_column` eventually
results in `hash[:my_column] = model.my_column`.

Sequel Models define accessor methods for each column in the underlying table,
so technically underneath the hood Packer is actually calling the sending the
method `column_name` to the model: `hash[:my_column] = model.send(:my_column)`.

This means that the result of any method can be serialized using
`field :method_name`. For example, suppose a User model has a `first_name` and
`last_name` column, and a helper method `full_name`:

```ruby
class User < Sequel::Model(:users)
  def full_name
    "#{first_name} #{last_name}"
  end
end
```

Then when `User.create(first_name: "Paul", last_name: "Martinez")` gets packed
with `field :full_name` specified, the outputted hash will contain
`full_name: "Paul Martinez"`.

### `self.field(key, &block)`

A block can be passed to `field` to perform arbitrary computation and store the
result under the specified `key`. The block will be passed the model as a single
argument. Use this to call methods on the model that may take additional
arguments, or to "rename" a column.

Examples:

```ruby
class MyPacker < Sequel::Packer
  model MyModel

  field :friendly_public_name do |model|
    model.unfriendly_internal_name
  end

  # Shorthand for above
  field :friendly_public_name, &:unfriendly_internal_name

  field :foo do |model|
    model.bar(baz, quux)
  end
end
```

### `self.field(&block)`

Passing a block but no `key` to `field` allows for arbitrary manipulation of the
packed hash. The block will be passed the model and the partially packed hash.
One potential usage is for dynamic keys that cannot be determined at load time,
but otherwise it's meant as a general escape hatch.

```ruby
field do |model, hash|
  hash[model.compute_dynamic_key] = model.dynamic_value
end
```

### `pack(dataset)`

After creating a new instance of a Packer class, call `packer.pack(dataset)` to
materialize a dataset and convert it to an array of packed Ruby hashes.

## Development

After checking out the repo, run `bin/setup` to install dependencies. Then, run
`rake test` to run the tests. You can also run `bin/console` for an interactive
prompt that will allow you to experiment.

To install this gem onto your local machine, run `bundle exec rake install`. To
release a new version, update the version number in `version.rb`, and then run
`bundle exec rake release`, which will create a git tag for the version, push
git commits and tags, and push the `.gem` file to
[rubygems.org](https://rubygems.org).

## Contributing

Bug reports and pull requests are welcome on GitHub at
https://github.com/PaulJuliusMartinez/sequel-packer.


## License

The gem is available as open source under the terms of the
[MIT License](https://opensource.org/licenses/MIT).
