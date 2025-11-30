# Thin Models

Models are data containers. They define structure (associations, scopes) and provide simple readers. All logic lives elsewhere.

## What Models Contain

### 1. Associations
```ruby
class Member < ApplicationRecord
  belongs_to :club
  belongs_to :user, optional: true

  has_many :contacts, dependent: :destroy
  has_many :event_registrations, dependent: :destroy
  has_many :registered_events, through: :event_registrations, source: :event
  has_many :member_notes, dependent: :destroy
  has_many :authored_notes, class_name: "MemberNote", foreign_key: :author_id, dependent: :nullify
end
```

### 2. Scopes
Scopes are query shortcuts. They don't contain business logic, just query building:

```ruby
class Member < ApplicationRecord
  # Status scopes
  scope :active, -> { where(status: ["active", nil]) }
  scope :archived, -> { where(status: "archived") }
  scope :suspended, -> { where(status: "suspended") }

  # Role scopes
  scope :admins, -> { where(role: "admin") }
  scope :coaches, -> { where(role: "coach") }
  scope :members, -> { where(role: "member") }

  # Ordering scopes
  scope :by_name, -> { order(:first_name, :last_name) }
  scope :recent, -> { order(created_at: :desc) }

  # Filtering scopes
  scope :for_club, ->(club) { where(club: club) }
  scope :by_role, ->(role) { where(role: role) }
  scope :by_status, ->(status) { where(status: status) }

  # Eager loading scopes
  scope :with_contacts, -> { includes(:contacts) }
  scope :with_notes, -> { includes(:member_notes) }
  scope :with_associations, -> { includes(:contacts, :event_registrations, :member_notes) }
end
```

### 3. Simple Reader Methods
Readers compute derived values. They:
- Return data without side effects
- Don't modify state
- Are named as questions (`enabled?`) or nouns (`full_name`)

```ruby
class Member < ApplicationRecord
  # Computed attribute
  def full_name
    "#{first_name} #{last_name}".strip
  end

  def name
    full_name
  end

  # Boolean readers
  def active?
    status.nil? || status == "active"
  end

  def archived?
    status == "archived"
  end

  def admin?
    role == "admin"
  end

  def minor?
    birth_date.present? && birth_date > 18.years.ago.to_date
  end

  # Simple lookups (no business logic)
  def primary_contact
    contacts.find_by(primary: true)
  end

  def primary_email
    primary_contact&.email
  end
end
```

## What Models Do NOT Contain

### 1. Validations
Validations live in contracts:

```ruby
# Bad - validations in model
class Member < ApplicationRecord
  validates :first_name, presence: true
  validates :email, format: { with: URI::MailTo::EMAIL_REGEXP }
end

# Good - model has no validations
class Member < ApplicationRecord
  # No validations here - they're in Members::Contracts::Create
end
```

### 2. Mutator Methods
Methods that change state live in services:

```ruby
# Bad - mutator methods in model
class Member < ApplicationRecord
  def archive!
    update!(status: "archived", archived_at: Time.current)
  end

  def activate!
    update!(status: "active", archived_at: nil)
  end

  def assign_role!(new_role)
    update!(role: new_role)
  end
end

# Good - use services instead
# Members::Commands::Archive.call(member)
# Members::Commands::Activate.call(member)
# Members::Commands::AssignRole.call(member, role: "admin")
```

### 3. Business Logic
Complex decisions live in services:

```ruby
# Bad - business logic in model
class Member < ApplicationRecord
  def can_be_archived?
    return false if status == "archived"
    return false if admin? && club.members.admins.count == 1
    return false if event_registrations.future.any?
    true
  end

  def archive!
    raise "Cannot archive" unless can_be_archived?
    update!(status: "archived", archived_at: Time.current)
  end
end

# Good - use validator and command services
# Members::Validators::CanArchive.call(member)
# Members::Commands::Archive.call(member)
```

### 4. Callbacks with Side Effects
Callbacks that do more than simple data normalization live in operations:

```ruby
# Bad - side effect callbacks
class Member < ApplicationRecord
  after_create :send_welcome_email
  after_update :notify_admin_of_changes
  before_destroy :cancel_all_registrations

  private

  def send_welcome_email
    MemberMailer.welcome(self).deliver_later
  end
end

# Good - handle in operations
module Members
  module Operations
    class Create < Trailblazer::Operation
      step :model
      step Contract::Build(...)
      step Contract::Validate(...)
      step Contract::Persist()
      step :send_welcome_email  # Explicit, not hidden

      private

      def send_welcome_email(ctx, **)
        MemberMailer.welcome(ctx[:model]).deliver_later
      end
    end
  end
end
```

### 5. Complex Queries
Complex queries live in query services:

```ruby
# Bad - complex query in model
class Member < ApplicationRecord
  def self.search(term, filters = {})
    scope = all
    if term.present?
      search_term = "%#{term.downcase}%"
      scope = scope.where(
        "LOWER(first_name) LIKE ? OR LOWER(last_name) LIKE ?",
        search_term, search_term
      )
    end
    scope = scope.where(status: filters[:status]) if filters[:status].present?
    scope = scope.where(role: filters[:role]) if filters[:role].present?
    scope
  end
end

# Good - simple scope in model, complex query in service
class Member < ApplicationRecord
  scope :search_by_name, ->(term) {
    where("LOWER(first_name) LIKE :t OR LOWER(last_name) LIKE :t", t: "%#{term.downcase}%")
  }
end

# Complex filtering in service:
# Members::Queries::Filter.call(scope: members, filters: params)
```

## Model Template

```ruby
# frozen_string_literal: true

class Member < ApplicationRecord
  # === Associations ===
  belongs_to :club
  belongs_to :user, optional: true

  has_many :contacts, dependent: :destroy
  has_many :event_registrations, dependent: :destroy
  has_many :member_notes, dependent: :destroy

  # === Scopes ===
  scope :active, -> { where(status: ["active", nil]) }
  scope :archived, -> { where(status: "archived") }
  scope :by_name, -> { order(:first_name, :last_name) }
  scope :for_club, ->(club) { where(club: club) }
  scope :with_contacts, -> { includes(:contacts) }

  # === Readers ===
  def full_name
    "#{first_name} #{last_name}".strip
  end

  def active?
    status.nil? || status == "active"
  end

  def admin?
    role == "admin"
  end
end
```

## Guidelines Summary

| Allowed in Models | Not Allowed in Models |
|-------------------|----------------------|
| `belongs_to`, `has_many`, etc. | `validates` |
| `scope :name, -> { ... }` | `before_save`, `after_create` (with side effects) |
| `def full_name` (reader) | `def archive!` (mutator) |
| `def active?` (boolean reader) | `def can_be_archived?` (business logic) |
| `def primary_contact` (simple lookup) | `def self.complex_search(...)` |
