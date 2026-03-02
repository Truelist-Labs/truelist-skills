# Ruby / Rails Patterns

## Installation

```bash
bundle add truelist
```

## Basic Usage

```ruby
require "truelist"

client = Truelist::Client.new(api_key: ENV["TRUELIST_API_KEY"])
result = client.validate("user@example.com")

result.state       # "ok" | "email_invalid" | "risky" | "unknown" | "accept_all"
result.sub_state   # "email_ok" | "is_disposable" | "is_role" | ...
result.suggestion  # nil or "user@gmail.com"
result.valid?      # true if state == "ok"
result.disposable? # true if sub_state == "is_disposable"
result.role?       # true if sub_state == "is_role"
```

## Rails Model Validation

```ruby
# app/validators/email_deliverability_validator.rb
class EmailDeliverabilityValidator < ActiveModel::EachValidator
  def validate_each(record, attribute, value)
    return if value.blank?

    client = Truelist::Client.new(api_key: ENV["TRUELIST_API_KEY"])
    result = client.validate(value)

    if result.state == "email_invalid"
      record.errors.add(attribute, "is not a deliverable email address")
    end

    if result.disposable?
      record.errors.add(attribute, "cannot be a disposable email address")
    end

    if result.suggestion
      record.errors.add(attribute, "did you mean #{result.suggestion}?")
    end
  end
end
```

```ruby
# app/models/user.rb
class User < ApplicationRecord
  validates :email, presence: true, email_deliverability: true
end
```

## Rails Controller Example

```ruby
# app/controllers/api/email_validations_controller.rb
class Api::EmailValidationsController < ApplicationController
  def create
    email = params[:email]
    return render json: { error: "Email required" }, status: :bad_request if email.blank?

    client = Truelist::Client.new(api_key: ENV["TRUELIST_API_KEY"])
    result = client.validate(email)

    render json: {
      valid: result.valid?,
      state: result.state,
      suggestion: result.suggestion
    }
  end
end
```

## Service Object Pattern

```ruby
# app/services/validate_email.rb
class ValidateEmail
  def initialize(email)
    @email = email
  end

  def call
    client = Truelist::Client.new(api_key: ENV["TRUELIST_API_KEY"])
    result = client.validate(@email)

    {
      valid: result.valid?,
      state: result.state,
      sub_state: result.sub_state,
      suggestion: result.suggestion,
      disposable: result.disposable?,
      role: result.role?
    }
  end
end
```

## Batch Validation with Rate Limiting

```ruby
emails = ["user1@example.com", "user2@example.com", "user3@example.com"]
client = Truelist::Client.new(api_key: ENV["TRUELIST_API_KEY"])

emails.each do |email|
  result = client.validate(email)
  puts "#{email}: #{result.state}"
  sleep 0.1 # respect rate limits
end
```
