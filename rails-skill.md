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

## CI: Local with gh-signoff

Use [basecamp/gh-signoff](https://github.com/basecamp/gh-signoff) for local CI. Run the test suite on your own machine and sign off when tests pass. No cloud CI required.

### Setup

Install the GitHub CLI extension:

```bash
gh extension install basecamp/gh-signoff
```

Require signoff for PR merges:

```bash
gh signoff install
```

### bin/ci

Create a `bin/ci` script that runs the full test suite and signs off on success:

```bash
#!/bin/bash
set -e

echo "== Running tests =="
bin/rails test
bin/rails test:system

echo ""
echo "== All tests passed. Signing off. =="
gh signoff
```

Make it executable:

```bash
chmod +x bin/ci
```

### Usage

Run your local CI before merging:

```bash
bin/ci
```

This runs all tests (unit, integration, system) and if everything passes, signs off on the current commit via `gh signoff`. The green status will appear on your PR, satisfying the branch protection rule.

### Partial Signoff (Optional)

For larger projects, use partial signoff to track different CI steps:

```bash
#!/bin/bash
set -e

echo "== Running unit & integration tests =="
bin/rails test
gh signoff tests

echo "== Running system tests =="
bin/rails test:system
gh signoff system

echo ""
echo "== All checks passed =="
```

Require partial signoff:

```bash
gh signoff install tests system
```

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
| CI | Local via `bin/ci` + `gh signoff` |
