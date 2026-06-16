---
name: rails
description: |
  Conventions and patterns for building Rails applications: business logic in concerns
  and (possibly tableless) models rather than service objects, Hotwire/Turbo for page
  updates and broadcasting, Stimulus for client-side interactivity, Minitest with
  fixtures (no RSpec, no factories), and local CI via Rails 8's
  `ActiveSupport::ContinuousIntegration` plus `gh-signoff` instead of cloud CI.
  Use whenever modifying or scaffolding Rails code in this codebase — these are
  house rules, not suggestions.
---

# Rails Skill

Conventions and patterns for Rails development. This is a skill — a set of guidelines that define how we build Rails applications.

## Architecture

### No Service Objects

Do not use service objects. Place business logic in **concerns** or **models** instead.

Models do not need a corresponding database table. Use plain Ruby classes under `app/models/` for encapsulating domain logic that doesn't need persistence.

Organize models into subdirectories for clarity:

```
app/models/
├── user.rb
├── post.rb
├── authentication/
│   ├── session.rb          # No DB table needed
│   └── token_generator.rb  # No DB table needed
├── billing/
│   ├── charge.rb
│   └── receipt_builder.rb  # No DB table needed
└── notifications/
    ├── delivery.rb
    └── preferences.rb
```

#### Models Without Database Tables

For models that don't back a database table, use `ActiveModel` APIs:

```ruby
# app/models/authentication/token_generator.rb
class Authentication::TokenGenerator
  include ActiveModel::Model
  include ActiveModel::Attributes

  attribute :user
  attribute :scope, :string, default: "default"

  validates :user, presence: true

  def generate
    # Business logic here
  end
end
```

#### Concerns for Shared Behavior

Extract shared behavior into concerns under `app/models/concerns/`:

```ruby
# app/models/concerns/trackable.rb
module Trackable
  extend ActiveSupport::Concern

  included do
    has_many :events, as: :trackable, dependent: :destroy
  end

  def track(action, metadata = {})
    events.create!(action: action, metadata: metadata)
  end
end
```

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  include Trackable
end
```

---

## Frontend

### Hotwire / Turbo

Use **Turbo** for page navigation and dynamic updates. Turbo Drive, Frames, and Streams cover most interactive UI needs without writing custom JavaScript.

#### Turbo Frames

Decompose pages into independently-loading frames:

```erb
<%# app/views/posts/show.html.erb %>
<%= turbo_frame_tag @post do %>
  <h1><%= @post.title %></h1>
  <p><%= @post.body %></p>
  <%= link_to "Edit", edit_post_path(@post) %>
<% end %>
```

#### Turbo Streams

Broadcast real-time updates from models:

```ruby
# app/models/message.rb
class Message < ApplicationRecord
  after_create_commit -> { broadcast_append_to "messages" }
  after_update_commit -> { broadcast_replace_to "messages" }
  after_destroy_commit -> { broadcast_remove_to "messages" }
end
```

Respond with Turbo Stream actions from controllers:

```ruby
# app/controllers/messages_controller.rb
def create
  @message = Message.new(message_params)

  if @message.save
    respond_to do |format|
      format.turbo_stream
      format.html { redirect_to messages_path }
    end
  else
    render :new, status: :unprocessable_entity
  end
end
```

```erb
<%# app/views/messages/create.turbo_stream.erb %>
<%= turbo_stream.append "messages", @message %>
```

### Stimulus

Use **Stimulus** for client-side interactivity that Turbo cannot handle. Keep controllers small and focused.

```javascript
// app/javascript/controllers/toggle_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["content"]

  toggle() {
    this.contentTarget.classList.toggle("hidden")
  }
}
```

```erb
<div data-controller="toggle">
  <button data-action="click->toggle#toggle">Show/Hide</button>
  <div data-toggle-target="content" class="hidden">
    Content here
  </div>
</div>
```

---

## Testing

### Minitest with Fixtures

Use **Minitest** for all tests. Do not use RSpec. Use **fixtures** for test data — do not use factories (no FactoryBot).

#### Fixtures

```yaml
# test/fixtures/users.yml
alice:
  name: Alice
  email: alice@example.com

bob:
  name: Bob
  email: bob@example.com
```

```yaml
# test/fixtures/posts.yml
hello_world:
  title: Hello World
  body: This is a post.
  user: alice
```

#### Model Tests

```ruby
# test/models/post_test.rb
require "test_helper"

class PostTest < ActiveSupport::TestCase
  test "valid post" do
    post = posts(:hello_world)
    assert post.valid?
  end

  test "invalid without title" do
    post = posts(:hello_world)
    post.title = nil
    assert_not post.valid?
  end
end
```

#### Controller / Integration Tests

```ruby
# test/controllers/posts_controller_test.rb
require "test_helper"

class PostsControllerTest < ActionDispatch::IntegrationTest
  test "should get index" do
    get posts_url
    assert_response :success
  end

  test "should create post" do
    assert_difference("Post.count") do
      post posts_url, params: { post: { title: "New", body: "Content" } }
    end
    assert_redirected_to post_url(Post.last)
  end
end
```

#### System Tests

```ruby
# test/system/posts_test.rb
require "application_system_test_case"

class PostsTest < ApplicationSystemTestCase
  test "creating a post" do
    visit posts_url
    click_on "New Post"

    fill_in "Title", with: "System Test Post"
    fill_in "Body", with: "Created via system test."
    click_on "Create Post"

    assert_text "System Test Post"
  end
end
```

---

## CI: Local with Rails `CI` runner and gh-signoff

Use Rails 8's built-in `ActiveSupport::ContinuousIntegration` runner together with
[basecamp/gh-signoff](https://github.com/basecamp/gh-signoff). Run the full CI
pipeline on your own machine and sign off when everything passes. No cloud CI
required.

### Setup

Install the GitHub CLI extension:

```bash
gh extension install basecamp/gh-signoff
```

Require signoff for PR merges:

```bash
gh signoff install
```

### config/ci.rb

Define the CI pipeline in `config/ci.rb`. Keep `bin/ci` as the Rails 8 default,
which loads `ActiveSupport::ContinuousIntegration` and requires this file.

```ruby
# Run using bin/ci

CI.run do
  step "Setup", "bin/setup --skip-server"

  step "Tests: Unit & integration", "bin/rails test"
  step "Tests: System", "bin/rails test:system"

  step "Style: Ruby", "bin/rubocop"

  step "Security: Gem audit", "bin/bundler-audit"
  step "Security: Importmap vulnerability audit", "bin/importmap audit"
  step "Security: Brakeman code analysis", "bin/brakeman --quiet --no-pager --exit-on-warn --exit-on-error"

  if success?
    step "Signoff: All systems go. Ready for merge and deploy.", "gh signoff"
  else
    failure "Signoff: CI failed. Do not merge or deploy.", "Fix the issues and try again."
  end
end
```

### bin/ci

Leave `bin/ci` as the Rails 8 default:

```ruby
#!/usr/bin/env ruby
require_relative "../config/boot"
require "active_support/continuous_integration"

CI = ActiveSupport::ContinuousIntegration
require_relative "../config/ci.rb"
```

Make sure it is executable:

```bash
chmod +x bin/ci
```

### Usage

Run your local CI before merging:

```bash
bin/ci
```

This runs setup, the full Minitest suite, style checks, security audits, and if
everything passes, signs off on the current commit via `gh signoff`. The green
status will appear on your PR, satisfying the branch protection rule.

### GitHub repository settings

After installing `gh signoff`, configure the repository so PRs cannot be merged
without a passing signoff:

1. Go to **Settings → Branches**.
2. Edit the rule for your default branch (`main`).
3. Enable **"Restrict deletions"** and **"Require pull request reviews before merging"**.
4. Under **"Require status checks to pass before merging"**, enable it and search
   for the `signoff` status check. Add it as required.
5. Enable **"Require conversation resolution before merging"** if desired.
6. Save the rule.

Now `gh signoff` acts as the required CI gate for every PR.

---

## Summary

| Area | Convention |
|---|---|
| Business logic | Concerns and models (no service objects) |
| Models | Can be tableless; nest in subdirectories for organization |
| Frontend interactivity | Stimulus controllers |
| Page navigation & updates | Hotwire / Turbo (Drive, Frames, Streams) |
| Testing framework | Minitest |
| Test data | Fixtures (no factories) |
| CI | Local via `ActiveSupport::ContinuousIntegration` + `gh signoff` |
