# Contracts & Validation

Contracts (Reform forms) handle input validation and serve as form helpers. They do NOT contain business logic.

## Contract Structure

```ruby
module Members
  module Contracts
    class Create < Reform::Form
      property :first_name
      property :last_name
      property :email
      property :role
      property :status

      validates :first_name, presence: true, length: { maximum: 255 }
      validates :last_name, presence: true, length: { maximum: 255 }
      validates :email, format: { with: URI::MailTo::EMAIL_REGEXP }, allow_blank: true
      validates :role, presence: true, inclusion: { in: %w[admin member coach] }
      validates :status, inclusion: { in: %w[active inactive suspended], allow_blank: true }
    end
  end
end
```

## Key Rules

### 1. Validation Only
Contracts contain ONLY:
- Property declarations
- Standard Rails validations
- Custom validation methods (for complex validation logic)

Contracts do NOT contain:
- Business logic (slug generation, defaults, etc.)
- Database queries beyond uniqueness checks
- Side effects

```ruby
# Bad - business logic in contract
module Members
  module Contracts
    class Create < Reform::Form
      property :name
      property :slug

      def validate(params = {})
        super.tap do |result|
          # DON'T: Business logic here
          self.slug = name.parameterize if result && name.present?
        end
      end
    end
  end
end

# Good - validation only
module Members
  module Contracts
    class Create < Reform::Form
      property :name
      property :slug

      validates :name, presence: true
      validates :slug, presence: true, format: { with: /\A[a-z0-9-]+\z/ }
    end
  end
end
```

### 2. Custom Validation Methods
Use Rails validation hooks for complex validation:

```ruby
module Events
  module Contracts
    class Create < Reform::Form
      property :starts_at
      property :ends_at
      property :registration_deadline

      validates :starts_at, presence: true
      validates :ends_at, presence: true

      validate :ends_after_starts
      validate :registration_before_start

      private

      def ends_after_starts
        return unless starts_at && ends_at
        errors.add(:ends_at, "must be after start time") if ends_at <= starts_at
      end

      def registration_before_start
        return unless registration_deadline && starts_at
        if registration_deadline > starts_at
          errors.add(:registration_deadline, "must be before event starts")
        end
      end
    end
  end
end
```

### 3. Nested Properties
For associated records:

```ruby
module Members
  module Contracts
    class Create < Reform::Form
      property :first_name
      property :last_name

      # Nested form for primary contact
      property :primary_contact, form: Contacts::Contracts::Create

      # Collection of contacts
      collection :contacts, form: Contacts::Contracts::Create

      validates :first_name, presence: true
      validates :last_name, presence: true
    end
  end
end
```

## Contract Templates

### Create Contract
```ruby
module Members
  module Contracts
    class Create < Reform::Form
      property :first_name
      property :last_name
      property :email
      property :role
      property :status
      property :birth_date

      validates :first_name, presence: true, length: { maximum: 255 }
      validates :last_name, presence: true, length: { maximum: 255 }
      validates :email, format: { with: URI::MailTo::EMAIL_REGEXP }, allow_blank: true
      validates :role, presence: true, inclusion: { in: %w[admin member coach] }
      validates :status, inclusion: { in: %w[active inactive], allow_blank: true }
    end
  end
end
```

### Update Contract
Often inherits from Create with modifications:

```ruby
module Members
  module Contracts
    class Update < Create
      # Override validations that differ for update
      # Or add update-specific properties

      property :archived_at

      # Can relax some validations for updates
      validates :role, inclusion: { in: %w[admin member coach] }, allow_blank: true
    end
  end
end
```

### Index/Filter Contract
For filtering collections:

```ruby
module Members
  module Contracts
    class Index < Reform::Form
      property :search
      property :status
      property :role
      property :sort_by
      property :sort_direction

      validates :status, inclusion: { in: %w[active inactive all] }, allow_blank: true
      validates :role, inclusion: { in: %w[admin member coach] }, allow_blank: true
      validates :sort_by, inclusion: { in: %w[name created_at updated_at] }, allow_blank: true
      validates :sort_direction, inclusion: { in: %w[asc desc] }, allow_blank: true

      def to_h
        {
          search: search,
          status: status,
          role: role,
          sort_by: sort_by || "name",
          sort_direction: sort_direction || "asc"
        }.compact
      end
    end
  end
end
```

## Using Contracts as Form Helpers

Contracts can act as form objects in views via the `@form` pattern:

### In Operations
```ruby
module Members
  module Operations
    class New < Trailblazer::Operation
      step :model
      step Contract::Build(constant: Members::Contracts::Create)

      private

      def model(ctx, current_club:, **)
        ctx[:model] = Member.new(club_id: current_club.id)
      end
    end
  end
end
```

### In Controllers
```ruby
class MembersController < ApplicationController
  def new
    result = Members::Operations::New.call(current_club: current_club)
    @form = result["contract.default"]
  end

  def create
    result = Members::Operations::Create.call(
      params: member_params,
      current_club: current_club
    )

    if result.success?
      redirect_to member_path(result[:model])
    else
      @form = result["contract.default"]
      render :new
    end
  end

  def edit
    result = Members::Operations::Update.call(
      params: { id: params[:id] },
      current_club: current_club
    )
    @form = result["contract.default"]
  end
end
```

### In Views
```erb
<%= form_with model: @form, url: members_path do |f| %>
  <% if @form.errors.any? %>
    <div class="errors">
      <% @form.errors.full_messages.each do |msg| %>
        <p><%= msg %></p>
      <% end %>
    </div>
  <% end %>

  <%= f.label :first_name %>
  <%= f.text_field :first_name %>

  <%= f.label :last_name %>
  <%= f.text_field :last_name %>

  <%= f.label :email %>
  <%= f.email_field :email %>

  <%= f.submit %>
<% end %>
```

## Validation Patterns

### Conditional Validations
```ruby
validates :guardian_name, presence: true, if: :minor?
validates :guardian_email, presence: true, if: :minor?

private

def minor?
  birth_date.present? && birth_date > 18.years.ago
end
```

### Custom Error Messages
```ruby
validates :first_name, presence: { message: "First name can't be blank" }
validates :email, format: {
  with: URI::MailTo::EMAIL_REGEXP,
  message: "doesn't look like a valid email"
}
```

### Cross-Field Validation
```ruby
validate :password_confirmation_matches

private

def password_confirmation_matches
  return unless password.present? && password_confirmation.present?
  errors.add(:password_confirmation, "doesn't match") if password != password_confirmation
end
```

### Uniqueness (Use Sparingly)
For uniqueness, prefer checking in services, but if needed:

```ruby
validates :email, uniqueness: { scope: :club_id, message: "already registered" }
```

Note: This hits the database. For complex uniqueness checks, use a service in the operation.
