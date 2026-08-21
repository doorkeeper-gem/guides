# Testing

You can use Doorkeeper models in your application test suite. Note that starting from
Doorkeeper 4.3.0 it uses [ActiveSupport lazy loading hooks](http://api.rubyonrails.org/classes/ActiveSupport/LazyLoadHooks.html)
to load models. There are [known issue](https://github.com/doorkeeper-gem/doorkeeper/issues/1043)
with the `factory_bot_rails` gem (it executes factories building before `ActiveRecord::Base` is
initialized using hooks in gem railtie, so you can catch a `uninitialized constant` error).
It is recommended to use pure `factory_bot` gem to solve this problem.

## Controller specs (RSpec)

In controller specs, the token is typically stubbed rather than persisted. Use
`instance_double` to mock `doorkeeper_token` and `doorkeeper_authorize!` in a
`before_action`.

```ruby
RSpec.describe Api::V1::UsersController, type: :controller do
  let(:user) { create(:user) }

  before do
    token = instance_double('Doorkeeper::AccessToken',
                            acceptable?: true,
                            resource_owner_id: user.id,
                            scopes: [:write])
    allow(controller).to receive(:doorkeeper_token).and_return(token)
    # doorkeeper_authorize! is called as a before_action in the controller;
    # stubbing doorkeeper_token with acceptable?: true makes it pass.
  end

  it 'returns 200 with a valid token' do
    get :index
    expect(response).to have_http_status(:ok)
  end

  context 'with an invalid or expired token' do
    before do
      token = instance_double('Doorkeeper::AccessToken',
                              acceptable?: false,
                              resource_owner_id: nil)
      allow(controller).to receive(:doorkeeper_token).and_return(token)
    end

    it 'returns 401' do
      get :index
      expect(response).to have_http_status(:unauthorized)
    end
  end
end
```

The key is `acceptable?: true` — `doorkeeper_authorize!` checks this method,
so a false value triggers a 401 (or 403 for scope mismatches).

## Request specs (recommended)

Request specs exercise the full middleware stack and are the preferred approach.
Create a real `Doorkeeper::AccessToken` record (via FactoryBot or directly) and
pass the `Authorization: Bearer <token>` header.

**Factory definition:**

```ruby
# spec/factories/doorkeeper.rb
FactoryBot.define do
  factory :access_token, class: 'Doorkeeper::AccessToken' do
    association :application, factory: :oauth_application
    expires_in { 2.hours }
    scopes { '' }
  end
end

FactoryBot.define do
  factory :oauth_application, class: 'Doorkeeper::Application' do
    name { 'Test App' }
    redirect_uri { 'urn:ietf:wg:oauth:2.0:oob' }
    scopes { '' }
    confidential { true }
  end
end
```

**Spec example:**

```ruby
RSpec.describe 'Api::V1::Users', type: :request do
  let(:user)          { create(:user) }
  let(:application)   { create(:oauth_application) }
  let(:access_token)  { create(:access_token, resource_owner_id: user.id, application: application) }

  it 'returns 200 with a valid token' do
    get '/api/v1/users', headers: { 'Authorization' => "Bearer #{access_token.token}" }
    expect(response).to have_http_status(:ok)
  end

  it 'returns 401 without a token' do
    get '/api/v1/users'
    expect(response).to have_http_status(:unauthorized)
  end
end
```

## Minitest

With Minitest, mock `doorkeeper_token` using `Minitest::Mock` and stub it on
the controller with `@controller.stub`.

```ruby
require 'test_helper'

class Api::V1::UsersControllerTest < ActionController::TestCase
  def setup
    @user = users(:one)
  end

  test 'returns 200 with a valid token' do
    token = Minitest::Mock.new
    token.expect :acceptable?, true
    token.expect :resource_owner_id, @user.id

    @controller.stub :doorkeeper_token, token do
      get :index
      assert_response :ok
    end

    token.verify
  end

  test 'returns 401 when token is not acceptable' do
    token = Minitest::Mock.new
    token.expect :acceptable?, false

    @controller.stub :doorkeeper_token, token do
      get :index
      assert_response :unauthorized
    end

    token.verify
  end
end
```

## Helper reference

These are the key controller methods provided by `Doorkeeper::Rails::Helpers`
(verified against Doorkeeper 5.9.5 `lib/doorkeeper/rails/helpers.rb`).

| Helper | Purpose |
| :--- | :--- |
| `doorkeeper_authorize!(*scopes)` | `before_action` that requires a valid token. Accepts optional scope names, combined with a logical **OR**: `doorkeeper_authorize! :read, :write` accepts a token that has either scope. Call the helper once per scope to require several at the same time. Renders 401 (invalid/missing token) or 403 (none of the required scopes). |
| `doorkeeper_token` | Returns the current `Doorkeeper::AccessToken` instance (or nil). Memoized per-request via `OAuth::Token.authenticate`. |
| `current_resource_owner` | Available as a view helper since 5.9.1. Defined in `Doorkeeper::Helpers::Controller` and exposed via `helper_method`. Returns `@current_resource_owner` or evaluates `Doorkeeper.config.authenticate_resource_owner`. |
