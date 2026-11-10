# Pydantic v2 & Data Validation in FastAPI

> This guide explores how Pydantic powers data validation, parsing, and serialization in FastAPI, with a deep dive into Pydantic v2 architecture, validators, and settings management.

---

# 1. What is Pydantic & How Does It Work?

## Interview Question

**What is Pydantic? Is it a data validation library or a data parsing library?**

---

### Answer

Pydantic is both, but it is primarily a **data parsing and type enforcement library**.

Many developers assume Pydantic only checks types and throws errors if something doesn't match.

In reality, Pydantic's core philosophy is **coercion**: it tries to parse incoming data into the declared Python type whenever possible:

```python
from pydantic import BaseModel

class User(BaseModel):
    id: int
    name: str
    is_active: bool

# Notice the types being passed in:
user = User(id="42", name="Alice", is_active="true")

print(type(user.id), user.id)               # <class 'int'> 42
print(type(user.is_active), user.is_active) # <class 'bool'> True
```

Pydantic saw the string `"42"` and safely coerced it to the integer `42`. It saw the string `"true"` and coerced it to the boolean `True`.

Only when data *cannot* be coerced (e.g. passing `"not-a-number"` for an `int`) does Pydantic raise a `ValidationError`.

---

# 2. Modern Pydantic v2 vs v1

## Interview Question

**What changed in Pydantic v2? What are the key architectural differences and breaking API changes from v1?**

---

### Answer

Pydantic v2 (released in mid-2023) was a complete architectural rewrite.

### The Rust Core Engine (`pydantic-core`)

In Pydantic v1, all validation logic was written in pure Python. While flexible, it had noticeable CPU overhead when parsing large JSON arrays containing thousands of items.

In Pydantic v2, the validation and serialization core was completely rewritten in **Rust** as `pydantic-core`.

**The Result**: Pydantic v2 is **5x to 50x faster** than v1, giving FastAPI throughput that rivals Node.js and Go for typical API workloads.

---

### Key API Migration Changes (v1 vs v2)

Interviewers frequently ask candidates if they understand the v2 migration:

| Operation | Pydantic v1 (Deprecated) | Pydantic v2 (Modern Standard) |
|---|---|---|
| **Export to dict** | `model.dict()` | `model.model_dump()` |
| **Export to JSON** | `model.json()` | `model.model_dump_json()` |
| **Parse raw dict** | `Model.parse_obj(data)` | `Model.model_validate(data)` |
| **Parse raw JSON string** | `Model.parse_raw(json_str)` | `Model.model_validate_json(json_str)` |
| **JSON Schema** | `Model.schema()` | `Model.model_json_schema()` |
| **Model Configuration** | Inner `class Config:` | `model_config = ConfigDict(...)` |
| **Field Validator** | `@validator('field')` | `@field_validator('field')` |
| **Root/Model Validator**| `@root_validator` | `@model_validator(mode='after')` |

---

# 3. The `Field()` Function & Constraints

## Interview Question

**How do you enforce numerical boundaries, string formats, and JSON aliases using `Field()`?**

---

### Answer

`Field()` allows you to attach validation constraints and OpenAPI metadata directly to model attributes:

```python
from pydantic import BaseModel, Field

class ProductCreate(BaseModel):
    name: str = Field(
        min_length=3,
        max_length=100,
        description="The commercial name of the product",
        examples=["Mechanical Keyboard"]
    )
    price: float = Field(
        gt=0,  # Greater than zero (Strictly positive)
        le=10000,  # Less than or equal to 10,000
        description="Price in USD"
    )
    sku: str = Field(
        pattern=r'^[A-Z]{3}-\d{4}$',  # Enforces regex: e.g. "PRD-1234"
        description="Stock Keeping Unit"
    )
```

If a client sends `price: -5`, Pydantic rejects the request before it reaches your view logic, returning an HTTP 422 with:

```json
{
    "loc": ["body", "price"],
    "msg": "Input should be greater than 0",
    "type": "greater_than"
}
```

---

### Handling Frontend CamelCase with Aliases

Frontends (JavaScript/TypeScript) typically use `camelCase`, while Python PEP8 conventions require `snake_case`.

You can accept camelCase JSON from the client while working with clean snake_case variables in Python:

```python
from pydantic import BaseModel, Field, ConfigDict

class UserProfile(BaseModel):
    model_config = ConfigDict(populate_by_name=True)

    first_name: str = Field(alias="firstName")
    last_name: str = Field(alias="lastName")
    phone_number: str | None = Field(default=None, alias="phoneNumber")
```

Client sends:

```json
{"firstName": "Alice", "lastName": "Smith"}
```

In your FastAPI route:

```python
@app.post("/profile")
async def save_profile(profile: UserProfile):
    # Python accesses clean snake_case attributes!
    print(profile.first_name, profile.last_name)
```

---

# 4. Custom Field & Model Validators

## Interview Question

**How do you write custom field-level and model-level validators in Pydantic v2?**

---

### 1. Field-Level Validator (`@field_validator`)

Use `@field_validator` to validate or normalize a single field:

```python
from pydantic import BaseModel, field_validator

class RegisterRequest(BaseModel):
    username: str
    email: str

    @field_validator('username')
    @classmethod
    def validate_username(cls, value: str) -> str:
        if not value.isalnum():
            raise ValueError('Username must contain only alphanumeric characters')
        return value.lower()  # Normalizes to lowercase
```

---

### 2. Model-Level Validator (`@model_validator`)

Use `@model_validator` when a business rule involves comparing **two or more fields**:

```python
from pydantic import BaseModel, model_validator

class PasswordResetRequest(BaseModel):
    new_password: str
    confirm_password: str

    @model_validator(mode='after')
    def verify_passwords_match(self) -> 'PasswordResetRequest':
        if self.new_password != self.confirm_password:
            raise ValueError('Passwords do not match')
        return self
```

In `mode='after'`, the validator runs *after* all individual fields have passed basic type checks. `self` is a fully instantiated instance of the model.

---

# 5. Environment & Settings Management (`pydantic-settings`)

## Interview Question

**How do you manage application settings and environment variables in FastAPI using Pydantic?**

---

### Answer

In Pydantic v2, settings management was moved into a dedicated package: `pydantic-settings`.

It allows you to define application configuration as a strongly typed class that automatically reads from the environment, `.env` files, or Docker secrets:

```python
# core/config.py
from pydantic_settings import BaseSettings, SettingsConfigDict
from pydantic import PostgresDsn

class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        case_sensitive=False
    )

    APP_NAME: str = "My FastAPI Backend"
    ENVIRONMENT: str = "development"
    DEBUG: bool = False
    
    # Strongly typed database URL (Validates protocol, host, port)
    DATABASE_URL: PostgresDsn
    SECRET_KEY: str
    ACCESS_TOKEN_EXPIRE_MINUTES: int = 30

# Create a singleton settings object
settings = Settings()
```

If `DATABASE_URL` or `SECRET_KEY` is missing from the environment when the application boots up, Pydantic crashes immediately on startup with a descriptive error rather than failing silently at runtime when a user hits a database endpoint!
