# Testing with RSpec

Testing Trailblazer architecture: operations, services, contracts, and thin models.

## Testing Philosophy

- **Operations**: Test the full workflow - success and failure paths
- **Services**: Test domain logic in isolation
- **Contracts**: Test validation edge cases
- **Models**: Test scopes and reader methods only (no validations to test)
- **Request specs**: Test API endpoints with JSON:API compliance

## Operation Specs

Operations are the primary testing target. Test complete workflows.

```ruby
# spec/concepts/members/operations/create_spec.rb
RSpec.describe Members::Operations::Create do
  let(:club) { create(:club) }
  let(:user) { create(:user) }
  let(:auth_member) { create(:member, :admin, club: club, user: user) }

  let(:valid_params) do
    {
      data: {
        first_name: "John",
        last_name: "Doe",
        email: "john@example.com",
        role: "member"
      }
    }
  end

  subject(:result) do
    described_class.call(
      params: valid_params,
      current_user: user,
      current_club: club,
      auth_member: auth_member
    )
  end

  describe "success" do
    it "creates a member" do
      expect(result).to be_success
      expect(result[:model]).to be_persisted
      expect(result[:model].first_name).to eq("John")
      expect(result[:model].last_name).to eq("Doe")
      expect(result[:model].club).to eq(club)
    end
  end

  describe "validation failure" do
    let(:valid_params) do
      { data: { first_name: "", last_name: "", role: "member" } }
    end

    it "fails with validation errors" do
      expect(result).to be_failure
      expect(result["contract.default"].errors[:first_name]).to be_present
      expect(result["contract.default"].errors[:last_name]).to be_present
    end
  end

  describe "authorization failure" do
    let(:auth_member) { create(:member, :member, club: club) }  # Not admin

    it "fails authorization" do
      expect(result).to be_failure
      expect(result["contract.default"]).to be_nil  # Never built
    end
  end
end
```

### Testing Different Operation Types

#### Index Operation
```ruby
RSpec.describe Members::Operations::Index do
  let(:club) { create(:club) }
  let(:user) { create(:user) }
  let!(:active_members) { create_list(:member, 3, club: club, status: "active") }
  let!(:archived_member) { create(:member, club: club, status: "archived") }

  describe "success" do
    it "returns active members" do
      result = described_class.call(
        params: { filter: { status: "active" } },
        current_user: user,
        current_club: club
      )

      expect(result).to be_success
      expect(result[:model]).to match_array(active_members)
      expect(result[:model]).not_to include(archived_member)
    end
  end
end
```

#### Update Operation
```ruby
RSpec.describe Members::Operations::Update do
  let(:club) { create(:club) }
  let(:member) { create(:member, club: club, first_name: "Old") }

  describe "success" do
    it "updates the member" do
      result = described_class.call(
        params: { id: member.id, data: { first_name: "New" } },
        current_user: user,
        current_club: club,
        auth_member: admin_member
      )

      expect(result).to be_success
      expect(result[:model].reload.first_name).to eq("New")
    end
  end
end
```

## Service Specs

Test services in isolation with focused unit tests.

### Query Service
```ruby
# spec/services/members/queries/filter_spec.rb
RSpec.describe Members::Queries::Filter do
  let(:club) { create(:club) }
  let!(:john) { create(:member, club: club, first_name: "John", status: "active") }
  let!(:jane) { create(:member, club: club, first_name: "Jane", status: "active") }
  let!(:archived) { create(:member, club: club, first_name: "Archived", status: "archived") }

  describe ".call" do
    it "filters by search term" do
      result = described_class.call(
        scope: club.members,
        filters: { search: "john" }
      )

      expect(result).to contain_exactly(john)
    end

    it "filters by status" do
      result = described_class.call(
        scope: club.members,
        filters: { status: "active" }
      )

      expect(result).to contain_exactly(john, jane)
    end

    it "combines filters" do
      result = described_class.call(
        scope: club.members,
        filters: { search: "ja", status: "active" }
      )

      expect(result).to contain_exactly(jane)
    end
  end
end
```

### Command Service
```ruby
# spec/services/members/commands/archive_spec.rb
RSpec.describe Members::Commands::Archive do
  let(:member) { create(:member, status: "active") }
  let(:user) { create(:user) }

  describe ".call" do
    it "archives the member" do
      result = described_class.call(member, archived_by: user)

      expect(member.reload.status).to eq("archived")
      expect(member.archived_at).to be_present
    end

    it "cancels future registrations" do
      future_event = create(:event, starts_at: 1.week.from_now)
      registration = create(:event_registration, member: member, event: future_event)

      described_class.call(member, archived_by: user)

      expect(registration.reload.status).to eq("cancelled")
    end
  end
end
```

### Validator Service
```ruby
# spec/services/members/validators/can_archive_spec.rb
RSpec.describe Members::Validators::CanArchive do
  describe ".call" do
    it "returns true for archivable member" do
      member = create(:member, role: "member")
      expect(described_class.call(member)).to be true
    end

    it "returns false for already archived member" do
      member = create(:member, status: "archived")
      expect(described_class.call(member)).to be false
    end

    it "returns false for sole admin" do
      club = create(:club)
      admin = create(:member, club: club, role: "admin")

      expect(described_class.call(admin)).to be false
    end

    it "returns true for admin when other admins exist" do
      club = create(:club)
      admin1 = create(:member, club: club, role: "admin")
      admin2 = create(:member, club: club, role: "admin")

      expect(described_class.call(admin1)).to be true
    end
  end
end
```

## Contract Specs

Test validation edge cases. Keep these focused.

```ruby
# spec/concepts/members/contracts/create_spec.rb
RSpec.describe Members::Contracts::Create do
  let(:member) { Member.new }
  let(:contract) { described_class.new(member) }

  describe "validations" do
    it "requires first_name" do
      contract.validate(first_name: "", last_name: "Doe", role: "member")
      expect(contract.errors[:first_name]).to include("First name can't be blank")
    end

    it "requires last_name" do
      contract.validate(first_name: "John", last_name: "", role: "member")
      expect(contract.errors[:last_name]).to include("Last name can't be blank")
    end

    it "validates role inclusion" do
      contract.validate(first_name: "John", last_name: "Doe", role: "invalid")
      expect(contract.errors[:role]).to be_present
    end

    it "accepts valid email format" do
      contract.validate(
        first_name: "John",
        last_name: "Doe",
        role: "member",
        email: "john@example.com"
      )
      expect(contract.errors[:email]).to be_empty
    end

    it "rejects invalid email format" do
      contract.validate(
        first_name: "John",
        last_name: "Doe",
        role: "member",
        email: "not-an-email"
      )
      expect(contract.errors[:email]).to be_present
    end
  end
end
```

## Model Specs

Only test scopes and reader methods. No validations.

```ruby
# spec/models/member_spec.rb
RSpec.describe Member, type: :model do
  describe "associations" do
    it { is_expected.to belong_to(:club) }
    it { is_expected.to have_many(:contacts).dependent(:destroy) }
    it { is_expected.to have_many(:event_registrations).dependent(:destroy) }
  end

  describe "scopes" do
    describe ".active" do
      it "returns members with active or nil status" do
        active = create(:member, status: "active")
        nil_status = create(:member, status: nil)
        archived = create(:member, status: "archived")

        expect(Member.active).to contain_exactly(active, nil_status)
      end
    end

    describe ".by_name" do
      it "orders by first_name then last_name" do
        charlie = create(:member, first_name: "Charlie", last_name: "Brown")
        alice = create(:member, first_name: "Alice", last_name: "Smith")
        alice_jones = create(:member, first_name: "Alice", last_name: "Jones")

        expect(Member.by_name).to eq([alice_jones, alice, charlie])
      end
    end
  end

  describe "#full_name" do
    it "combines first and last name" do
      member = build(:member, first_name: "John", last_name: "Doe")
      expect(member.full_name).to eq("John Doe")
    end

    it "handles missing names" do
      member = build(:member, first_name: "John", last_name: nil)
      expect(member.full_name).to eq("John")
    end
  end

  describe "#active?" do
    it "returns true for active status" do
      member = build(:member, status: "active")
      expect(member.active?).to be true
    end

    it "returns true for nil status" do
      member = build(:member, status: nil)
      expect(member.active?).to be true
    end

    it "returns false for archived status" do
      member = build(:member, status: "archived")
      expect(member.active?).to be false
    end
  end
end
```

## Request Specs

Test API endpoints with JSON:API compliance.

```ruby
# spec/requests/api/v1/members_spec.rb
RSpec.describe "Api::V1::Members", type: :request do
  let(:club) { create(:club) }
  let(:user) { create(:user) }
  let(:auth_member) { create(:member, :admin, club: club, user: user) }
  let(:headers) { auth_headers(user) }

  describe "GET /api/v1/clubs/:club_id/members" do
    let!(:members) { create_list(:member, 3, club: club) }

    it "returns members for the club" do
      get "/api/v1/clubs/#{club.id}/members", headers: headers

      expect(response).to have_http_status(:ok)
      expect(json_response[:data].size).to eq(3)
      expect(json_response[:data].first[:type]).to eq("members")
    end

    it "requires authentication" do
      get "/api/v1/clubs/#{club.id}/members"
      expect(response).to have_http_status(:unauthorized)
    end
  end

  describe "POST /api/v1/clubs/:club_id/members" do
    let(:valid_params) do
      {
        data: {
          type: "members",
          attributes: {
            first_name: "John",
            last_name: "Doe",
            role: "member"
          }
        }
      }
    end

    it "creates a member" do
      expect {
        post "/api/v1/clubs/#{club.id}/members",
             params: valid_params,
             headers: headers,
             as: :json
      }.to change(Member, :count).by(1)

      expect(response).to have_http_status(:created)
      expect(json_response[:data][:attributes][:first_name]).to eq("John")
    end

    it "returns validation errors" do
      invalid_params = { data: { type: "members", attributes: { first_name: "" } } }

      post "/api/v1/clubs/#{club.id}/members",
           params: invalid_params,
           headers: headers,
           as: :json

      expect(response).to have_http_status(:unprocessable_entity)
      expect(json_response[:errors]).to be_present
    end
  end
end
```

## Factories

```ruby
# spec/factories/members.rb
FactoryBot.define do
  factory :member do
    club
    sequence(:first_name) { |n| "Member#{n}" }
    last_name { "Test" }
    role { "member" }
    status { "active" }

    trait :admin do
      role { "admin" }
    end

    trait :coach do
      role { "coach" }
    end

    trait :archived do
      status { "archived" }
      archived_at { Time.current }
    end

    trait :with_contacts do
      after(:create) do |member|
        create(:contact, :primary, member: member)
        create(:contact, member: member)
      end
    end
  end
end
```

## Shared Examples

```ruby
# spec/support/shared_examples/operation_authorization.rb
RSpec.shared_examples "requires admin authorization" do
  context "when user is not admin" do
    let(:auth_member) { create(:member, :member, club: club) }

    it "fails authorization" do
      expect(result).to be_failure
    end
  end
end

# Usage
RSpec.describe Members::Operations::Destroy do
  it_behaves_like "requires admin authorization"
end
```

## Test Helpers

```ruby
# spec/support/request_helpers.rb
module RequestHelpers
  def json_response
    JSON.parse(response.body, symbolize_names: true)
  end

  def auth_headers(user)
    token = JwtService.encode(user_id: user.id)
    { "Authorization" => "Bearer #{token}" }
  end
end

RSpec.configure do |config|
  config.include RequestHelpers, type: :request
end
```
