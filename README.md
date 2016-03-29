# Hanami::Validations

Validations mixins for objects

## Status

[![Gem Version](http://img.shields.io/gem/v/hanami-validations.svg)](https://badge.fury.io/rb/hanami-validations)
[![Build Status](http://img.shields.io/travis/hanami/validations/master.svg)](https://travis-ci.org/hanami/validations?branch=master)
[![Coverage](http://img.shields.io/coveralls/hanami/validations/master.svg)](https://coveralls.io/r/hanami/validations)
[![Code Climate](http://img.shields.io/codeclimate/github/hanami/validations.svg)](https://codeclimate.com/github/hanami/validations)
[![Dependencies](http://img.shields.io/gemnasium/hanami/validations.svg)](https://gemnasium.com/hanami/validations)
[![Inline Docs](http://inch-ci.org/github/hanami/validations.svg)](http://inch-ci.org/github/hanami/validations)

## Contact

* Home page: http://hanamirb.org
* Mailing List: http://hanamirb.org/mailing-list
* API Doc: http://rdoc.info/gems/hanami-validations
* Bugs/Issues: https://github.com/hanami/validations/issues
* Support: http://stackoverflow.com/questions/tagged/hanami
* Chat: http://chat.hanamirb.org

## Rubies

__Hanami::Validations__ supports Ruby (MRI) 2+, JRuby 9k+

## Installation

Add this line to your application's Gemfile:

```ruby
gem 'hanami-validations'
```

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install hanami-validations

## Usage

`Hanami::Validations` is a set of lightweight validations for Ruby objects.

### Attributes

The framework allows you to define attributes for each object.

It defines an initializer, whose attributes can be passed as a hash.
All unknown values are ignored, which is useful for whitelisting attributes.

```ruby
require 'hanami/validations'

class Person
  include Hanami::Validations

  attribute :name,  presence: true
  attribute :email, presence: true
end

person = Person.new(name: 'Luca', email: 'me@example.org', age: 32)
person.name  # => "Luca"
person.email # => "me@example.org"
person.age   # => raises NoMethodError because `:age` wasn't defined as attribute.
```

#### Blank Values

The framework will treat as valid any blank attributes, __without__ `presence`, for both `format` and `size` predicates.

```ruby
require 'hanami/validations'

class Person
  include Hanami::Validations

  attribute :name,    type: String, size: 5..45
  attribute :email,   type: String, size: 20..80, format: /@/
  attribute :skills,  type: Array,  size: 1..3
  attribute :keys,    type: Hash,   size: 1..3
end

Person.new.valid?                             # < true
Person.new(name: '').valid?                   # < true
Person.new(skills: '').valid?                 # < true
Person.new(skills: ['ruby', 'hanami']).valid?  # < true

Person.new(skills: []).valid?                 # < false
Person.new(keys: {}).valid?                   # < false
Person.new(keys: {a: :b}, skills: []).valid?  # < false
```

If you want to _disable_ this behaviour, please, refer to [presence](https://github.com/hanami/validations#presence).

### Validations

If you prefer Hanami::Validations to **only define validations**, but **not attributes**,
you can use the following alternative syntax.

```ruby
require 'hanami/validations'

class Person
  include Hanami::Validations
  attr_accessor :name, :email

  # Custom initializer
  def initialize(attributes = {})
    @name, @email = attributes.values_at(:name, :email)
  end

  validates :name,  presence: true
  validates :email, presence: true
end

person = Person.new(name: 'Luca', email: 'me@example.org')
person.name  # => "Luca"
person.email # => "me@example.org"
```

This is a bit more verbose, but offers a great level of flexibility for your
Ruby objects. It also allows to use Hanami::Validations in combination with
**other frameworks**.

### Coercions

If a Ruby class is passed to the `:type` option, the given value is coerced, accordingly.

#### Standard coercions

```ruby
require 'hanami/validations'

class Person
  include Hanami::Validations

  attribute :fav_number, type: Integer
end

person = Person.new(fav_number: '23')
person.valid?

person.fav_number # => 23
```

Allowed types are:

  * `Array`
  * `BigDecimal`
  * `Boolean`
  * `Date`
  * `DateTime`
  * `Float`
  * `Hash`
  * `Integer`
  * `Pathname`
  * `Set`
  * `String`
  * `Symbol`
  * `Time`

#### Custom coercions

If a user defined class is specified, it can be freely used for coercion purposes.
The only limitation is that the constructor should have **arity of 1**.

```ruby
require 'hanami/validations'

class FavNumber
  def initialize(number)
    @number = number
  end
end

class BirthDate
end

class Person
  include Hanami::Validations

  attribute :fav_number, type: FavNumber
  attribute :date,       type: BirthDate
end

person = Person.new(fav_number: '23', date: 'Oct 23, 2014')
person.valid?

person.fav_number # => #<FavNumber:0x007ffc644bba00 @number="23">
person.date       # => this raises an error, because BirthDate#initialize doesn't accept any arg
```

### Validations

Each attribute definition can receive a set of options to define one or more
validations.

**Validations are triggered when you invoke `#valid?`.**

#### Acceptance

An attribute is valid if its value is _truthy_.

```ruby
require 'hanami/validations'

class Signup
  include Hanami::Validations

  attribute :terms_of_service, acceptance: true
end

signup = Signup.new(terms_of_service: '1')
signup.valid? # => true

signup = Signup.new(terms_of_service: 'true')
signup.valid? # => true

signup = Signup.new(terms_of_service: '')
signup.valid? # => false

signup = Signup.new(terms_of_service: '0')
signup.valid? # => false
```

#### Confirmation

An attribute is valid if its value and the value of a corresponding attribute
is valid.

By convention, if you have a `password` attribute, the validation looks for `password_confirmation`.

```ruby
require 'hanami/validations'

class Signup
  include Hanami::Validations

  attribute :password, confirmation: true
end

signup = Signup.new(password: 'secret', password_confirmation: 'secret')
signup.valid? # => true

signup = Signup.new(password: 'secret', password_confirmation: 'x')
signup.valid? # => false
```

#### Exclusion

An attribute is valid, if the value isn't excluded from the value described by
the validator.

The validator value can be anything that responds to `#include?`.
In Ruby, this includes most of the core objects: `String`, `Enumerable` (`Array`, `Hash`,
`Range`, `Set`).

See also [Inclusion](#inclusion).

```ruby
require 'hanami/validations'

class Signup
  include Hanami::Validations

  attribute :music, exclusion: ['pop']
end

signup = Signup.new(music: 'rock')
signup.valid? # => true

signup = Signup.new(music: 'pop')
signup.valid? # => false
```

#### Format

An attribute is valid if it matches the given Regular Expression.

```ruby
require 'hanami/validations'

class Signup
  include Hanami::Validations

  attribute :name, format: /\A[a-zA-Z]+\z/
end

signup = Signup.new(name: 'Luca')
signup.valid? # => true

signup = Signup.new(name: '23')
signup.valid? # => false
```

#### Inclusion

An attribute is valid, if the value provided is included in the validator's
value.

The validator value can be anything that responds to `#include?`.
In Ruby, this includes most of the core objects: like `String`, `Enumerable` (`Array`, `Hash`,
`Range`, `Set`).

See also [Exclusion](#exclusion).

```ruby
require 'prime'
require 'hanami/validations'

class PrimeNumbers
  def initialize(limit)
    @numbers = Prime.each(limit).to_a
  end

  def include?(number)
    @numbers.include?(number)
  end
end

class Signup
  include Hanami::Validations

  attribute :age,        inclusion: 18..99
  attribute :fav_number, inclusion: PrimeNumbers.new(100)
end

signup = Signup.new(age: 32)
signup.valid? # => true

signup = Signup.new(age: 17)
signup.valid? # => false

signup = Signup.new(fav_number: 23)
signup.valid? # => true

signup = Signup.new(fav_number: 8)
signup.valid? # => false
```

#### Presence

An attribute is valid if present.

```ruby
require 'hanami/validations'

class Signup
  include Hanami::Validations

  attribute :name, presence: true
end

signup = Signup.new(name: 'Luca')
signup.valid? # => true

signup = Signup.new(name: '')
signup.valid? # => false

signup = Signup.new(name: nil)
signup.valid? # => false
```

#### Size

An attribute is valid if its `#size` falls within the described value.

```ruby
require 'hanami/validations'

class Signup
  MEGABYTE = 1024 ** 2
  include Hanami::Validations

  attribute :ssn,      size: 11    # exact match
  attribute :password, size: 8..64 # range
  attribute :avatar,   size: 1..(5 * MEGABYTE)
end

signup = Signup.new(password: 'a-very-long-password')
signup.valid? # => true

signup = Signup.new(password: 'short')
signup.valid? # => false
```

**Note that in the example above you are able to validate the weight of the file,
because Ruby's `File` and `Tempfile` both respond to `#size`.**

#### Uniqueness

Uniqueness validations aren't implemented because this library doesn't deal with persistence.
The other reason is that this isn't an effective way to ensure uniqueness of a value in a database.

Please read more at: [The Perils of Uniqueness Validations](http://robots.thoughtbot.com/the-perils-of-uniqueness-validations).

### Custom validations

You can add custom validation in several ways

#### Using `Hanami::Validations::Validation`

Hanami provides the `Hanami::Validations::Validation` mixin you can include in any class to validate attributes.

To use it, you must define a `#validate` method.

The validation name will be taken from its declaration, for instance, for `validate_address_existence_with:` the validation name will be `:address_existence`.

Example:

```ruby
require 'hanami/validations'

class Person
  include Hanami::Validations

  attribute :name,    presence: true
  attribute :email,   presence: true
  attribute :address, presence: true,
                      validate_address_existence_with: AddressExistenceValidator
end

class AddressExistenceValidator
  include Hanami::Validations::Validation

  def validate
    add_error unless address_exists?
  end

  def address
    value
  end

  def address_exists?
    query_address.matches_count == 1
  end

  def query_address
    AddressQueryService.new.query_address(address)
  end
end
```

By including `include Hanami::Validations::Validation` in your class, you have access to the following protocol:

```ruby
# Get the value of the attribute being validated
value

# Get the value of any attribute
value_of('address.street')

# Get the validation name
validation_name

# Override the validation name
validation_name :location_existance

# Get the name of the attribute being validated
attribute_name

# Get the namespace of the attribute being validated
namespace

# Answer whether the attribute being validated is blank or not
blank_value?

# Answer whether any value is blank or not
blank_value?(value_of 'address.street')

# Answer whether a previous validation failed for the attribute being validated or not
validation_failed_for? :size

# Answer whether a previous validation failed for another attribute or not
validation_failed_for? :size, on: 'address.street'

# Answer whether a previous validation failed for the attribute being validated or not
any_validation_failed?

# Answer whether any previous validation failed for another attribute or not
any_validation_failed? on: 'address.street'

# Add a default error for the attribute being validated
add_error

# Add an error for the attribute being validated overriding any of these parameters:
# :validation_name, :expected_value, :actual_value, :namespace
add_error expected_value: 'An existent address'

# Add an error for any attribute defining all of these parameters:
# :validation_name, :expected_value, :actual_value, :namespace
add_error_for 'street',
  namespace: 'address', validation_name: validation_name,
  expected_value: true, actual_value: value_of('address.street')

# Validate any attribute with any validation
validate_attribute 'address.street', on: :format, with: /abc/
```

#### Using an `Hanami::Validations::Validation` instance

If you need to initialize or configure you validator in a certain way, you can give a validator instance instead of a class.

Example:

```ruby
require 'hanami/validations'

class Person
  include Hanami::Validations

  attribute :name,    presence: true
  attribute :email,   presence: true
  attribute :address, presence: true, 
                      validate_address_existence_with: AddressExistenceValidator.new(in_city: 'Buenos Aires')
end
```

The instance will be `#dup` every time the validation is invoked.

#### Using a block

If the validation is simple enough, you can give a block.

Example:

```ruby
require 'hanami/validations'

class Person
  include Hanami::Validations

  attribute :name,    presence: true
  attribute :email,   presence: true
  attribute :age,     type: Integer,  presence: true,
                      validate_adult_with: proc{ add_error unless value >= 21 }
end
```

#### Using many custom validations on the same attribute

You can declare as many custom validations as you wish for each attribute.

Example:

```ruby
require 'hanami/validations'

class Person
  include Hanami::Validations

  attribute :name,    presence: true
  attribute :email,   presence: true
  attribute :address, presence: true, 
                      validate_address_format_with: AddressComplexFormatValidator,
                      validate_address_existence_with: AddressExistanceValidator
end
```

The validations, both built-in and custom, for all the attributes will be evaluated in the same order as they were declared. This is useful if you want to perform a validation only if a previous validation did not fail.

#### Conditional validations

Instead of declaring conditions for a specific validation, you can ask for the value of any attribute and then decide whether to validate an attribute or not:

Example:

```ruby
require 'hanami/validations'

class Person
  include Hanami::Validations

  attribute :name,          presence: true
  attribute :email,         presence: true
  attribute :document_type, presence: true, inclusion: ['passport', 'id']
  attribute :document_id,   presence: true, validate_document_id_with: DocumentIdValidator
end

class DocumentIdValidator
  include Hanami::Validations::Validation

  def validate
    return if preconditions_failed?

    is_passport? ? validate_passport : validate_id
  end

  def preconditions_failed?
    validation_failed_for? :presence, on: 'document_type'   ||
    validation_failed_for? :inclusion, on: 'document_type'  ||
    validation_failed_for? :presence
  end

  def is_passport?
    value_of('document_type') == 'passport'
  end

  def validate_passport
    validate_attribute 'document_id', on: :format, with: /.../
  end

  def validate_id
    validate_attribute 'document_id', on: :format, with: /.../
  end
end
```

### Nested validations

Nested validations are handled with a nested block syntax.

```ruby
class ShippingDetails
  include Hanami::Validations

  attribute :full_name, presence: true

  attribute :address do
    attribute :street,      presence: true
    attribute :city,        presence: true
    attribute :country,     presence: true
    attribute :postal_code, presence: true, format: /.../
  end
end

validator = ShippingDetails.new
validator.valid? # => false
```

Bulk operations on errors are guaranteed by `#each`.
This method yields a **flattened collection of errors**.

```ruby
validator.errors.each do |error|
  error.name
    # => on the first iteration it returns "full_name"
    # => the second time it returns "address.street" and so on..
end
```

Errors for a specific attribute can be accessed via `#for`.

```ruby
error = validator.errors.for('full_name').first
error.name           # => "full_name"
error.attribute_name # => "full_name"

error = validator.errors.for('address.street').first
error.name           # => "address.street"
error.attribute_name # => "street"
```

### Composable validations

Validations can be reused via composition:

```ruby
require 'hanami/validations'

module NameValidations
  include Hanami::Validations

  attribute :name, presence: true
end

module EmailValidations
  include Hanami::Validations

  attribute :email, presence: true, format: /.../
end

module PasswordValidations
  include Hanami::Validations

  # We validate only the presence here
  attribute :password, presence: true
end

module CommonValidations
  include EmailValidations
  include PasswordValidations
end

# A valid signup requires:
#   * name (presence)
#   * email (presence and format)
#   * password (presence and confirmation)
class Signup
  include NameValidations
  include CommonValidations

  # We decorate PasswordValidations behavior, by requiring the confirmation too.
  # This additional validation is active only in this case.
  attribute :password, confirmation: true
end

# A valid signin requires:
#   * email (presence)
#   * password (presence)
class Signin
  include CommonValidations
end

# A valid "forgot password" requires:
#   * email (presence)
class ForgotPassword
  include EmailValidations
end
```

### Complete example

```ruby
require 'hanami/validations'

class Signup
  include Hanami::Validations

  attribute :first_name, presence: true
  attribute :last_name,  presence: true
  attribute :email,      presence: true, format: /\A(.*)@(.*)\.(.*)\z/
  attribute :password,   presence: true, confirmation: true, size: 8..64
end
```

### Errors

When you invoke `#valid?`, validation errors are available at `#errors`.
It's a set of errors grouped by attribute. Each error contains the name of the
invalid attribute, the failed validation, the expected value, and the current one.

```ruby
require 'hanami/validations'

class Signup
  include Hanami::Validations

  attribute :email, presence: true, format: /\A(.*)@(.*)\.(.*)\z/
  attribute :age, size: 18..99
end

signup = Signup.new(email: 'user@example.org')
signup.valid? # => true

signup = Signup.new(email: '', age: 17)
signup.valid? # => false

signup.errors
  # => #<Hanami::Validations::Errors:0x007fe00ced9b78
  # @errors={
  #   :email=>[
  #     #<Hanami::Validations::Error:0x007fe00cee3290 @attribute=:email, @validation=:presence, @expected=true, @actual="">,
  #     #<Hanami::Validations::Error:0x007fe00cee31f0 @attribute=:email, @validation=:format, @expected=/\A(.*)@(.*)\.(.*)\z/, @actual="">
  #   ],
  #   :age=>[
  #     #<Hanami::Validations::Error:0x007fe00cee30d8 @attribute=:age, @validation=:size, @expected=18..99, @actual=17>
  #   ]
  # }>
```

### Hanami::Entity

Integration with [`Hanami::Entity`](https://github.com/hanami/model) is straight forward.

```ruby
require 'hanami/model'
require 'hanami/validations'

class Product
  include Hanami::Entity
  include Hanami::Validations

  attribute :name,  type: String,  presence: true
  attribute :price, type: Integer, presence: true
end

product = Product.new(name: 'Book', price: '100')
product.valid? # => true

product.name  # => "Book"
product.price # => 100
```

## Contributing

1. Fork it ( https://github.com/hanami/validations/fork )
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create a new Pull Request

## Copyright

Copyright © 2014-2016 Luca Guidi – Released under MIT License

This project was formerly known as Lotus (`lotus-validations`).
