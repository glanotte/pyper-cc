# Trailblazer Operations

Operations are the orchestration layer. They define the "what" of business workflows, delegating the "how" to services.

## Standard Operation Structure

```ruby
module Members
  module Operations
    class Create < Trailblazer::Operation
      step :model                                              # Always first
      step ClubContext::Policy.step(Members::DefaultPolicy, :create?)
      step Contract::Build(constant: Members::Contracts::Create)
      step Contract::Validate(key: :data)
      step :assign_defaults                                    # Business logic via services
      step Contract::Persist()

      private

      def model(ctx, current_club:, **)
        ctx[:model] = Member.new(club_id: current_club.id)
      end

      def assign_defaults(ctx, **)
        # Call services for business logic - don't do it inline
        Members::Commands::AssignDefaults.call(ctx[:model])
      end
    end
  end
end
```

## Key Rules

### 1. `:model` is ALWAYS First Step
Every operation starts with `:model` to set `ctx[:model]`:

```ruby
# Create - new record
def model(ctx, current_club:, **)
  ctx[:model] = Member.new(club_id: current_club.id)
end

# Show/Update/Destroy - find existing
def model(ctx, params:, current_club:, **)
  ctx[:model] = current_club.members.find(params[:id])
end

# Index - collection
def model(ctx, current_club:, **)
  ctx[:model] = current_club.members.active
end
```

### 2. Operations Orchestrate, Don't Execute
Operations should NOT contain:
- SQL queries or ActiveRecord calls (except simple finds in `:model`)
- Complex business rules
- External API calls

Instead, delegate to services:

```ruby
# Bad - SQL in operation
def filter_members(ctx, params:, **)
  ctx[:model] = ctx[:model]
    .where("LOWER(first_name) LIKE ?", "%#{params[:search]}%")
    .where(status: params[:status])
    .order(created_at: :desc)
end

# Good - delegate to service
def filter_members(ctx, params:, **)
  ctx[:model] = Members::Queries::Search.call(
    scope: ctx[:model],
    term: params[:search],
    status: params[:status]
  )
end
```

### 3. Keep Steps Single-Purpose
Each step does one thing:

```ruby
step :model
step :authorize                    # Policy check
step :validate_business_rules      # Call validation service
step Contract::Build(...)
step Contract::Validate(...)
step :generate_slug                # Call service
step :notify_admins                # Call service
step Contract::Persist()
```

## Operation Templates

### Create Operation
```ruby
module Members
  module Operations
    class Create < Trailblazer::Operation
      step :model
      step ClubContext::Policy.step(Members::DefaultPolicy, :create?)
      step Contract::Build(constant: Members::Contracts::Create)
      step Contract::Validate(key: :data)
      step Contract::Persist()

      private

      def model(ctx, current_club:, **)
        ctx[:model] = Member.new(club_id: current_club.id)
      end
    end
  end
end
```

### Update Operation
```ruby
module Members
  module Operations
    class Update < Trailblazer::Operation
      step :model
      step ClubContext::Policy.step(Members::DefaultPolicy, :update?)
      step Contract::Build(constant: Members::Contracts::Update)
      step Contract::Validate(key: :data)
      step Contract::Persist()

      private

      def model(ctx, params:, current_club:, **)
        ctx[:model] = current_club.members.find(params[:id])
      end
    end
  end
end
```

### Show Operation
```ruby
module Members
  module Operations
    class Show < Trailblazer::Operation
      step :model
      step ClubContext::Policy.step(Members::DefaultPolicy, :show?)

      private

      def model(ctx, params:, current_club:, **)
        ctx[:model] = Members::Queries::FindWithAssociations.call(
          club: current_club,
          id: params[:id]
        )
      end
    end
  end
end
```

### Index Operation
```ruby
module Members
  module Operations
    class Index < Trailblazer::Operation
      step :model
      step ClubContext::Policy.step(Members::DefaultPolicy, :index?)
      step Contract::Build(constant: Members::Contracts::Index)
      step Contract::Validate(key: :filter)
      step :apply_filters

      private

      def model(ctx, current_club:, **)
        ctx[:model] = current_club.members
      end

      def apply_filters(ctx, **)
        contract = ctx["contract.default"]
        ctx[:model] = Members::Queries::Filter.call(
          scope: ctx[:model],
          filters: contract.to_h
        )
      end
    end
  end
end
```

### Destroy Operation
```ruby
module Members
  module Operations
    class Destroy < Trailblazer::Operation
      step :model
      step ClubContext::Policy.step(Members::DefaultPolicy, :destroy?)
      step :validate_can_destroy
      step :destroy

      private

      def model(ctx, params:, current_club:, **)
        ctx[:model] = current_club.members.find(params[:id])
      end

      def validate_can_destroy(ctx, **)
        # Delegate to service for complex validation
        Members::Validators::CanDestroy.call(ctx[:model])
      end

      def destroy(ctx, **)
        ctx[:model].destroy
      end
    end
  end
end
```

## Custom Action Operations

For non-CRUD actions, follow the same pattern:

```ruby
module Members
  module Operations
    class Archive < Trailblazer::Operation
      step :model
      step ClubContext::Policy.step(Members::DefaultPolicy, :archive?)
      step :validate_can_archive
      step :archive

      private

      def model(ctx, params:, current_club:, **)
        ctx[:model] = current_club.members.find(params[:id])
      end

      def validate_can_archive(ctx, **)
        Members::Validators::CanArchive.call(ctx[:model])
      end

      def archive(ctx, **)
        Members::Commands::Archive.call(ctx[:model])
      end
    end
  end
end
```

## Calling Operations

### From Controllers
```ruby
class MembersController < ApplicationController
  def create
    result = Members::Operations::Create.call(
      params: permitted_params,
      current_user: current_user,
      current_club: current_club
    )

    if result.success?
      render json: MemberSerializer.new(result[:model]), status: :created
    else
      render_errors(result)
    end
  end
end
```

### From Jobs
```ruby
class ProcessMemberImportJob < ApplicationJob
  def perform(import_id)
    import = Import.find(import_id)

    import.rows.each do |row|
      Members::Operations::Create.call(
        params: { data: row },
        current_club: import.club,
        current_user: import.user
      )
    end
  end
end
```

### From Other Operations
```ruby
module Registrations
  module Operations
    class Complete < Trailblazer::Operation
      step :model
      step :create_member
      step :send_welcome

      private

      def create_member(ctx, **)
        result = Members::Operations::Create.call(
          params: ctx[:member_params],
          current_club: ctx[:current_club],
          current_user: ctx[:current_user]
        )

        return false unless result.success?
        ctx[:member] = result[:model]
      end
    end
  end
end
```

## Error Handling

### Let Infrastructure Errors Bubble
Don't catch `ActiveRecord::RecordNotFound` in operations. Let controllers handle it:

```ruby
# Bad - catching infrastructure errors
def model(ctx, params:, **)
  ctx[:model] = Member.find(params[:id])
rescue ActiveRecord::RecordNotFound
  ctx[:error] = "Member not found"
  false
end

# Good - let it bubble
def model(ctx, params:, **)
  ctx[:model] = Member.find(params[:id])
end
```

### Business Errors Return False
For business rule violations, return false with error context:

```ruby
def validate_can_archive(ctx, **)
  unless Members::Validators::CanArchive.call(ctx[:model])
    ctx[:error] = "Cannot archive member with active registrations"
    return false
  end
  true
end
```
