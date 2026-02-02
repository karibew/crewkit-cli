---
doc_tier: 2
doc_type: reference
doc_status: active
created: 2026-01-03
last_reviewed: 2026-01-06
owner: platform-team
related_docs:
  - docs/prds/API_STANDARDIZATION_PLAN.md
  - CLAUDE.md
tags: [api, reference, response-format]
---

# API Response Format Quick Reference

**For Developers**: Use this guide when creating new API endpoints or updating existing ones.

---

## Standard Response Formats

### Single Resource

```json
{
  "data": {
    "id": "uuid",
    "name": "example",
    "status": "active"
  },
  "_links": {
    "self": "/api/v1/resources/uuid",
    "update": { "href": "/api/v1/resources/uuid", "method": "PATCH" }
  }
}
```

### Collection

```json
{
  "data": [
    { "id": "uuid1", "name": "first" },
    { "id": "uuid2", "name": "second" }
  ],
  "_links": {
    "self": "/api/v1/resources",
    "first": "/api/v1/resources?page=1",
    "next": "/api/v1/resources?page=2",
    "create": { "href": "/api/v1/resources", "method": "POST" }
  },
  "meta": {
    "total_count": 50,
    "current_page": 1,
    "total_pages": 2,
    "per_page": 25
  }
}
```

### Error

```json
{
  "error": {
    "code": "validation_error",
    "message": "Validation failed",
    "details": { "name": ["can't be blank"] },
    "request_id": "uuid"
  }
}
```

---

## Creating a New Endpoint

### 1. Create Serializer

```ruby
# app/serializers/example_serializer.rb
class ExampleSerializer < BaseSerializer
  attributes :id, :name, :status

  link :self do |record, _ctx|
    "/api/v1/examples/#{record.external_id}"
  end

  link :update, method: :patch, if: :can_update? do |record, _ctx|
    "/api/v1/examples/#{record.external_id}"
  end

  def id
    record.external_id
  end
end
```

### 2. Update Controller

```ruby
# app/controllers/api/v1/examples_controller.rb
class Api::V1::ExamplesController < BaseController
  include HateoasResponses

  def index
    examples = Example.all.page(params[:page])
    render_collection examples, serializer: ExampleSerializer
  end

  def show
    example = Example.find_by!(external_id: params[:id])
    authorize example
    render_resource example, serializer: ExampleSerializer
  end

  def create
    example = Example.new(example_params)
    authorize example

    if example.save
      render_resource example, serializer: ExampleSerializer, status: :created
    else
      render_validation_error(example)
    end
  end
end
```

### 3. Write Tests

```ruby
# test/controllers/api/v1/examples_controller_test.rb
test "index returns standardized format" do
  get api_v1_examples_url, headers: auth_headers
  json = JSON.parse(response.body)

  assert json.key?("data")
  assert json.key?("_links")
  assert json.key?("meta")
end
```

---

## Link Types (IANA Standard)

| Relation | Usage | Example |
|----------|-------|---------|
| `self` | Current resource | `"/api/v1/resources/uuid"` |
| `collection` | Parent collection | `"/api/v1/resources"` |
| `item` | Collection item | `"/api/v1/resources/uuid"` |
| `next` / `prev` | Pagination | `"/api/v1/resources?page=2"` |
| `first` / `last` | Pagination bounds | `"/api/v1/resources?page=1"` |
| `related` | Related resources | `"/api/v1/organizations/uuid"` |
| `edit` | Update URL (PUT/PATCH) | `{ "href": "...", "method": "PATCH" }` |
| `create` | Create URL (POST) | `{ "href": "...", "method": "POST" }` |

---

## Common Patterns

### Conditional Links (Authorization)

```ruby
link :delete, method: :delete, if: :can_destroy? do |record, _ctx|
  "/api/v1/examples/#{record.external_id}"
end

private

def can_destroy?
  can?(:destroy?)  # Uses Pundit policy from context
end
```

### State-Based Links

```ruby
link :publish, method: :post, if: :is_draft? do |record, _ctx|
  "/api/v1/examples/#{record.external_id}/publish"
end

private

def is_draft?
  record.status == "draft"
end
```

### Nested Data

```ruby
def serialize_data
  data = super
  data[:user] = {
    id: record.user.external_id,
    name: record.user.name
  }
  data
end
```

---

## Testing Checklist

- [ ] Response has `data` key
- [ ] Response has `_links` key (not `links`)
- [ ] Collections have `meta` with pagination
- [ ] Links include `method` for non-GET actions
- [ ] Conditional links respect authorization
- [ ] All tests pass: `rails test`
- [ ] OpenAPI docs updated: `rake rswag:specs:swaggerize`

---

## Manual Testing (curl)

```bash
# List resources
curl http://localhost:3050/api/v1/examples \
  -H "Authorization: Bearer YOUR_TOKEN"

# Show single resource
curl http://localhost:3050/api/v1/examples/uuid \
  -H "Authorization: Bearer YOUR_TOKEN"

# Create resource
curl -X POST http://localhost:3050/api/v1/examples \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"example": {"name": "test"}}'

# Expect error (validation)
curl -X POST http://localhost:3050/api/v1/examples \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"example": {}}'
```

---

## Don't Do This ❌

```ruby
# ❌ Manual JSON builders
def show
  render json: { example: example_json(@example) }
end

# ❌ Inconsistent envelope
def index
  render json: { examples: @examples }
end

# ❌ Using 'links' instead of '_links'
render json: { data: @data, links: @links }
```

## Do This Instead ✅

```ruby
# ✅ Use serializers
def show
  render_resource @example, serializer: ExampleSerializer
end

# ✅ Consistent envelope
def index
  render_collection @examples, serializer: ExampleSerializer
end

# ✅ Use '_links' key
# (BaseSerializer handles this automatically)
```

---

## Resources

- **Full Plan**: `API_STANDARDIZATION_PLAN.md`
- **BaseSerializer**: `app/serializers/base_serializer.rb`
- **Example Serializers**: `app/serializers/resource_serializer.rb`, `app/serializers/project_serializer.rb`
- **HATEOAS Concern**: `app/controllers/concerns/hateoas_responses.rb`

---

**Questions?** Ask rest-api-architect or check the full plan.