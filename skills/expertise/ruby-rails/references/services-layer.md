# Services Layer

Services contain domain logic: database queries, mutations, external API calls, and business rules. This keeps operations clean and models thin.

## Service Organization

```
app/services/
├── members/
│   ├── queries/
│   │   ├── find.rb
│   │   ├── filter.rb
│   │   ├── search.rb
│   │   └── with_associations.rb
│   ├── commands/
│   │   ├── archive.rb
│   │   ├── assign_defaults.rb
│   │   └── merge.rb
│   └── validators/
│       ├── can_archive.rb
│       └── can_destroy.rb
├── events/
│   ├── queries/
│   │   └── ...
│   └── commands/
│       └── ...
└── shared/
    ├── slug_generator.rb
    └── email_validator.rb
```

## Query Services

Query services encapsulate read operations. They take a scope or model and return results.

### Basic Query
```ruby
module Members
  module Queries
    class Find
      def self.call(club:, id:)
        club.members.find(id)
      end
    end
  end
end
```

### Query with Associations
```ruby
module Members
  module Queries
    class FindWithAssociations
      def self.call(club:, id:)
        club.members
          .includes(:contacts, :custom_field_values, :member_notes)
          .find(id)
      end
    end
  end
end
```

### Filter Query
```ruby
module Members
  module Queries
    class Filter
      def self.call(scope:, filters:)
        new(scope, filters).call
      end

      def initialize(scope, filters)
        @scope = scope
        @filters = filters
      end

      def call
        result = @scope
        result = apply_search(result)
        result = apply_status(result)
        result = apply_role(result)
        result = apply_sort(result)
        result
      end

      private

      def apply_search(scope)
        return scope if @filters[:search].blank?

        term = "%#{@filters[:search].downcase}%"
        scope.where(
          "LOWER(first_name) LIKE ? OR LOWER(last_name) LIKE ?",
          term, term
        )
      end

      def apply_status(scope)
        return scope if @filters[:status].blank? || @filters[:status] == "all"
        scope.where(status: @filters[:status])
      end

      def apply_role(scope)
        return scope if @filters[:role].blank?
        scope.where(role: @filters[:role])
      end

      def apply_sort(scope)
        column = @filters[:sort_by] || "first_name"
        direction = @filters[:sort_direction] || "asc"

        case column
        when "name"
          scope.order(first_name: direction, last_name: direction)
        when "created_at", "updated_at"
          scope.order(column => direction)
        else
          scope.order(first_name: :asc)
        end
      end
    end
  end
end
```

### Search Query
```ruby
module Members
  module Queries
    class Search
      def self.call(scope:, term:)
        return scope if term.blank?

        search_term = "%#{term.downcase}%"
        scope.where(
          "LOWER(first_name) LIKE :term OR " \
          "LOWER(last_name) LIKE :term OR " \
          "LOWER(email) LIKE :term",
          term: search_term
        )
      end
    end
  end
end
```

## Command Services

Command services encapsulate write operations and mutations.

### Basic Command
```ruby
module Members
  module Commands
    class Archive
      def self.call(member)
        new(member).call
      end

      def initialize(member)
        @member = member
      end

      def call
        @member.update!(
          status: "archived",
          archived_at: Time.current
        )
        @member
      end
    end
  end
end
```

### Command with Side Effects
```ruby
module Members
  module Commands
    class Archive
      def self.call(member, archived_by:)
        new(member, archived_by).call
      end

      def initialize(member, archived_by)
        @member = member
        @archived_by = archived_by
      end

      def call
        ActiveRecord::Base.transaction do
          archive_member
          cancel_registrations
          log_activity
        end
        @member
      end

      private

      def archive_member
        @member.update!(
          status: "archived",
          archived_at: Time.current,
          archived_by_id: @archived_by.id
        )
      end

      def cancel_registrations
        @member.event_registrations.pending.each do |registration|
          EventRegistrations::Commands::Cancel.call(registration)
        end
      end

      def log_activity
        Activities::Commands::Log.call(
          actor: @archived_by,
          action: "archived",
          target: @member
        )
      end
    end
  end
end
```

### Command with Result Object
For commands that can fail:

```ruby
module Members
  module Commands
    class Merge
      Result = Struct.new(:success?, :member, :errors, keyword_init: true)

      def self.call(source:, target:, merged_by:)
        new(source, target, merged_by).call
      end

      def initialize(source, target, merged_by)
        @source = source
        @target = target
        @merged_by = merged_by
        @errors = []
      end

      def call
        return failure("Cannot merge member into itself") if @source == @target
        return failure("Source member not found") if @source.nil?
        return failure("Target member not found") if @target.nil?

        ActiveRecord::Base.transaction do
          transfer_contacts
          transfer_registrations
          transfer_notes
          archive_source
        end

        success(@target)
      rescue StandardError => e
        failure(e.message)
      end

      private

      def transfer_contacts
        @source.contacts.update_all(member_id: @target.id)
      end

      def transfer_registrations
        @source.event_registrations.update_all(member_id: @target.id)
      end

      def transfer_notes
        @source.member_notes.update_all(member_id: @target.id)
      end

      def archive_source
        @source.update!(
          status: "merged",
          merged_into_id: @target.id,
          archived_at: Time.current
        )
      end

      def success(member)
        Result.new(success?: true, member: member, errors: [])
      end

      def failure(message)
        Result.new(success?: false, member: nil, errors: [message])
      end
    end
  end
end
```

## Validator Services

Validators check business rules without performing mutations.

```ruby
module Members
  module Validators
    class CanArchive
      def self.call(member)
        new(member).call
      end

      def initialize(member)
        @member = member
      end

      def call
        return false if @member.status == "archived"
        return false if @member.role == "admin" && sole_admin?
        true
      end

      private

      def sole_admin?
        @member.club.members.where(role: "admin").count == 1
      end
    end
  end
end
```

```ruby
module Members
  module Validators
    class CanDestroy
      def self.call(member)
        new(member).call
      end

      def initialize(member)
        @member = member
        @errors = []
      end

      def call
        check_not_sole_admin
        check_no_future_registrations
        check_no_unpaid_balances

        @errors.empty?
      end

      def errors
        @errors
      end

      private

      def check_not_sole_admin
        if @member.role == "admin" && sole_admin?
          @errors << "Cannot delete the only admin"
        end
      end

      def check_no_future_registrations
        if @member.event_registrations.joins(:event).where("events.starts_at > ?", Time.current).exists?
          @errors << "Member has future event registrations"
        end
      end

      def check_no_unpaid_balances
        if @member.balance.negative?
          @errors << "Member has an unpaid balance"
        end
      end

      def sole_admin?
        @member.club.members.where(role: "admin").count == 1
      end
    end
  end
end
```

## Shared Services

For cross-domain utilities:

```ruby
module Shared
  class SlugGenerator
    def self.call(text, scope: nil, model_class: nil, existing_id: nil)
      new(text, scope, model_class, existing_id).call
    end

    def initialize(text, scope, model_class, existing_id)
      @text = text
      @scope = scope
      @model_class = model_class
      @existing_id = existing_id
    end

    def call
      base_slug = @text.to_s.parameterize
      return base_slug unless @model_class

      ensure_unique(base_slug)
    end

    private

    def ensure_unique(base_slug)
      slug = base_slug
      counter = 1

      while slug_exists?(slug)
        slug = "#{base_slug}-#{counter}"
        counter += 1
      end

      slug
    end

    def slug_exists?(slug)
      query = @model_class.where(slug: slug)
      query = query.where(@scope) if @scope
      query = query.where.not(id: @existing_id) if @existing_id
      query.exists?
    end
  end
end
```

## Using Services in Operations

```ruby
module Members
  module Operations
    class Update < Trailblazer::Operation
      step :model
      step :validate_can_update
      step ClubContext::Policy.step(Members::DefaultPolicy, :update?)
      step Contract::Build(constant: Members::Contracts::Update)
      step Contract::Validate(key: :data)
      step :generate_slug
      step Contract::Persist()
      step :log_activity

      private

      def model(ctx, params:, current_club:, **)
        ctx[:model] = Members::Queries::FindWithAssociations.call(
          club: current_club,
          id: params[:id]
        )
      end

      def validate_can_update(ctx, params:, **)
        # Only validate role changes
        return true unless params.dig(:data, :role)

        validator = Members::Validators::RoleChange.new(ctx[:model], params[:data][:role])
        return true if validator.call

        ctx[:error] = validator.errors.join(", ")
        false
      end

      def generate_slug(ctx, **)
        contract = ctx["contract.default"]
        return true unless contract.name_changed?

        contract.slug = Shared::SlugGenerator.call(
          contract.name,
          scope: { club_id: ctx[:model].club_id },
          model_class: Member,
          existing_id: ctx[:model].id
        )
      end

      def log_activity(ctx, current_user:, **)
        Activities::Commands::Log.call(
          actor: current_user,
          action: "updated",
          target: ctx[:model]
        )
      end
    end
  end
end
```
