---
layout: post
title: Clean & Simple FactoryGirl Definitions
category: code
description: "Foo"
---

## Clean & Simple FactoryGirl Definitions

FactoryGirl is a test data creation library by
[Thoughtbot](https://thoughtbot.com/) that helps you setup data for your tests
in a structured and repeatable way. When used correctly, it allows to you
succinctly express what types of data your tests need, and build them without
having to repeat yourself.

To start, we need to put our factories somewhere. The creators recommend
sticking all your files in one `factories.rb` file, but some apps will be so
large that it becomes unwieldy. FactoryGirl will autload factories defined in
`spec/factories` and `spec/factories.rb`, and for our small blog app, one file
will be fine.

```ruby
# spec/factories.rb

FactoryGirl.define do
  factory :user do
    first_name "Example"
    last_name "User"
    email "user@example.com"
    joined_on Date.today
  end
end
```

```ruby
# spec/models/user_spec.rb

require 'spec_helper'

RSpec.describe User do
  describe "creating a user" do
    it "can be created" do
      expect(FactoryGirl.create(:user)).to be_a(User)
    end
  end
end
```

This is most simple factory definition, we just assign default values to each
attribute of `User`. If we want to overwrite any of them in our test, you can
pass a hash to `FactoryGirl.create` as the second argument, which will replace
any default attributes with ones that you specify.

### Sequences

This is nice, but most apps get a lot more complicated than that. Let's say we
have a unique constraint on the user's email. If we try to create more than one
`User` with the default values, it'll error on the unique constraint! So rather
than immediately jumping to specifying the user's email in every single test,
FactoryGirl provides 'sequences' for attributes.

```ruby
# spec/factories.rb

FactoryGirl.define do
  factory :user do
    first_name "Example"
    last_name "User"
    sequence :email do |i|
      "user#{i}@example.com"
    end
    joined_on Date.today
  end
end
```

Now when you generate a user in your tests, the email will increment, ensuring
that you'll never hit your unique constraint using default values.

### Traits

Another thing you might find yourself repating in your tests is the different
roles and object can fill. For example, let's say that a User with no
`joined_on` date is considered a guest user, or a user that hasn't signed up
yet. You'll probably find yourself writing a test context for "as a guest
user", wherein you need to create a guest user. FactoryGirl supplies `traits`
just for this scenario.

```ruby
# spec/factories.rb

FactoryGirl.define do
  factory :user do
    first_name "Example"
    last_name "User"
    sequence :email do |i|
      "user#{i}@example.com"
    end
    joined_on Date.today

    trait :guest do
      joined_on nil
    end
  end
end
```

```ruby
# spec/controllers/some_controller_spec.rb

require 'spec_helper'

RSpec.describe SomeController do
  context "as a guest user" do
    describe "GET" do
      guest_user = FactoryGirl.create(:user, :guest)
    end
  end
end
```

Usig traits, with one extra parameter you can specify roles for your objects.
This adds clarity for your tests, keeps things clean, and can be combined in
every way imaginable.

### Associations

Another major part of creating test data is associations, the way that data
relates to other data. In our blog app, a User has many posts, which has many
comments, and a User also has many comments (the comments they leave on other
people's posts). Let's also assume that we named the user association
`belongs_to :author, class: User` on both Post and Comment. Here's how we would
lay this out with FactoryGirl.

```ruby
# spec/factories.rb

FactoryGirl.define do
  factory :user do
    first_name "Example"
    last_name "User"
    sequence :email do |i|
      "user#{i}@example.com"
    end
    joined_on Date.today

    trait :guest do
      joined_on nil
    end
  end

  factory :post do
    title "My First Post"
    content "Posting is really fun"
    posted_on Date.today
    association :author, factory: :user
  end

  factory :comment do
    content "I love this post!"
    association :author, factory: :user
    post
  end
end
```

In the post factory, we define an author association, and because the name of
the association doesn't match a factory, we have to supply a factory name. In
the comment factory, we supply the same author assocation, but we also supply
`post`. Because the name of the association matches a factory, FactoryGirl can
automatically find everything on its own, we just have to tell it to build the
association.

Note: One way we can clean up this example is to define what I call an alias 
factory, which just points to the user factory.

```ruby
factory :author do
  user
end
```

Then, in our post and comment factories, we can just reference `author`,
because now there's a matching factory.

You can read more about FactoryGirl at the following links:

http://www.rubydoc.info/gems/factory_girl/file/GETTING_STARTED.md
