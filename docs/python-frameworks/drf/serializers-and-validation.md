# DRF Serializers & Validation

> This guide explains how Django REST Framework serializers convert complex data types to JSON and back, how the validation pipeline executes step by step, and how to build writable nested serializers safely.

---

# 1. Serialization vs Deserialization

## Interview Question

**What is the difference between serialization and deserialization, and what is the difference between `Serializer` and `ModelSerializer`?**

---

### Answer

Serializers act as bidirectional translators between Django's Python models and the outside world of HTTP/JSON:

```
               SERIALIZATION (Outbound)
Django Model ────────► Python Primitives ────────► JSON String
  Instance               (dict / list)                (Browser / Client)

               DESERIALIZATION (Inbound)
JSON String  ────────► Python Primitives ────────► Validated Data ──► Django Model
  (Client)               (request.data)              (attrs)            Instance
```

1. **Serialization**: Takes Python objects (like a `User` instance or QuerySet), extracts their attributes, and translates them into Python primitives (`dict`, `list`, `str`, `int`) that can be rendered into JSON.
2. **Deserialization**: Takes incoming JSON primitives (`request.data`), validates that types, lengths, and business constraints are satisfied, and transforms them into clean Python objects ready to be saved into the database.

---

### `Serializer` vs `ModelSerializer`

```python
# 1. Base Serializer: Manual declaration of every field
class ContactFormSerializer(serializers.Serializer):
    name = serializers.CharField(max_length=100)
    email = serializers.EmailField()
    message = serializers.CharField()

# 2. ModelSerializer: Automatically inspects the Django Model
class ProductSerializer(serializers.ModelSerializer):
    class Meta:
        model = Product
        fields = ['id', 'name', 'price', 'category']
```

**Why use `ModelSerializer`?**
- Automatically generates fields based on model definitions.
- Automatically generates validators (e.g. `unique=True` on a model field generates a `UniqueValidator`).
- Automatically provides default implementations of `.create()` and `.update()`.

**When to use base `Serializer`?**
- When the incoming or outgoing payload does **not** map to a database model (e.g. a password reset request containing `old_password` and `new_password`, or an analytics summary endpoint).

---

# 2. The Validation Pipeline & Execution Order

## Interview Question

**Walk me through the exact execution order of validation in DRF. When should you use field-level validation vs object-level validation?**

---

### Answer

Validation in DRF executes in a strict, predictable sequence when you call `serializer.is_valid()`:

```
                      serializer.is_valid()
                                │
                                ▼
         1. Basic Data Type Checks & Required Constraints
             (e.g. Ensures email has an '@', integer is numeric)
                                │
                                ▼
         2. Field-Level Validators (Built-in or Custom)
             (e.g. RegexValidator, MinValueValidator)
                                │
                                ▼
         3. Field-Level validate_<field_name>() Methods
             (e.g. validate_price, validate_username)
                                │
                                ▼
         4. Object-Level validate() Method
             (Cross-field comparisons: start_date vs end_date)
                                │
                                ▼
         Populates serializer.validated_data (OR raises ValidationError)
```

---

### Field-Level Validation: `validate_<field_name>`

Use field-level validation when you want to validate a **single field in isolation**:

```python
class UserRegistrationSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ['username', 'email', 'password']

    def validate_email(self, value):
        # Value is already parsed and guaranteed to be an email string
        if not value.endswith('@company.com'):
            raise serializers.ValidationError("Only company email addresses are allowed.")
        return value.lower()  # You can normalize data here!
```

**Rule**: Always return the value (or modified value) at the end of `validate_<field>()`!

---

### Object-Level Validation: `validate`

Use object-level validation when a validation rule depends on **multiple fields working together**:

```python
class ReservationSerializer(serializers.ModelSerializer):
    class Meta:
        model = Reservation
        fields = ['room_number', 'start_date', 'end_date']

    def validate(self, attrs):
        start_date = attrs.get('start_date')
        end_date = attrs.get('end_date')

        if start_date and end_date and start_date >= end_date:
            raise serializers.ValidationError({
                "end_date": "End date must be strictly after the start date."
            })

        return attrs
```

---

### Why you should always call `is_valid(raise_exception=True)`

In your views, prefer:

```python
# BEST PRACTICE
serializer = ProductSerializer(data=request.data)
serializer.is_valid(raise_exception=True)
serializer.save()
```

Over:

```python
# TEDIOUS & BOILERPLATE
if serializer.is_valid():
    serializer.save()
else:
    return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
```

When `raise_exception=True` is passed, DRF automatically raises a `serializers.ValidationError`. DRF's built-in exception handler catches it and returns a clean, standardized HTTP 400 Bad Request JSON response automatically.

---

# 3. Serializer Context (`context={'request': request}`)

## Interview Question

**What is serializer context, and why is it essential for building absolute URLs or accessing the currently authenticated user?**

---

### Answer

By default, serializers only know about the data explicitly passed into them. They do not have access to the incoming HTTP request.

If your serializer needs to:
- Access `request.user` (e.g. to check permissions or associate records)
- Generate absolute URLs for image/file fields (e.g. `https://api.mycompany.com/media/avatar.png`)

You pass extra data into the serializer using the `context` dictionary:

```python
# In your view:
serializer = OrderSerializer(data=request.data, context={'request': request})
```

Inside the serializer, you access context via `self.context`:

```python
class OrderSerializer(serializers.ModelSerializer):
    class Meta:
        model = Order
        fields = ['id', 'total_amount']

    def create(self, validated_data):
        # Read the authenticated user directly from serializer context
        user = self.context['request'].user
        return Order.objects.create(user=user, **validated_data)
```

GenericAPIView and ModelViewSet pass `{'request': request, 'view': self, 'format': self.format_kwarg}` into serializer context automatically.

---

# 4. Writable Nested Serializers

## Interview Question

**How do you create a writable nested serializer in DRF, and why is `transaction.atomic()` critical when saving nested children?**

---

### Answer

Suppose you have an `Order` model that has multiple `OrderItem` records:

```python
class Order(models.Model):
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    shipping_address = models.CharField(max_length=255)

class OrderItem(models.Model):
    order = models.ForeignKey(Order, related_name='items', on_delete=models.CASCADE)
    product_name = models.CharField(max_length=100)
    quantity = models.PositiveIntegerField()
```

A client wants to create the order and all its items in a single `POST` payload:

```json
{
    "shipping_address": "123 Main St",
    "items": [
        {"product_name": "Keyboard", "quantity": 1},
        {"product_name": "Mouse", "quantity": 2}
    ]
}
```

DRF's default `ModelSerializer` cannot automatically write to nested relationships because it doesn't know how you want child models created or updated.

You must explicitly implement the `create()` method:

```python
from django.db import transaction

class OrderItemSerializer(serializers.ModelSerializer):
    class Meta:
        model = OrderItem
        fields = ['product_name', 'quantity']

class OrderSerializer(serializers.ModelSerializer):
    items = OrderItemSerializer(many=True)

    class Meta:
        model = Order
        fields = ['id', 'shipping_address', 'items']

    def create(self, validated_data):
        # 1. Pop nested items from validated data
        items_data = validated_data.pop('items')

        # 2. Always wrap multi-table writes in an atomic transaction!
        with transaction.atomic():
            order = Order.objects.create(**validated_data)
            
            # 3. Create nested child records
            order_items = [
                OrderItem(order=order, **item_data)
                for item_data in items_data
            ]
            OrderItem.objects.bulk_create(order_items)

        return order
```

### Why `transaction.atomic()` is mandatory here:

If item 2 fails a database constraint, but item 1 succeeded, without `transaction.atomic()`, an orphaned order row would exist in the database with missing items. `transaction.atomic()` ensures all records are created together or rolled back completely.

---

# 5. `SerializerMethodField` & The N+1 Performance Trap

## Interview Question

**What is a `SerializerMethodField`, and how can overusing it silently cause massive N+1 query performance disasters?**

---

### Answer

`SerializerMethodField` is a read-only field that computes its value dynamically by calling a method on the serializer class:

```python
class AuthorSerializer(serializers.ModelSerializer):
    book_count = serializers.SerializerMethodField()

    class Meta:
        model = Author
        fields = ['id', 'name', 'book_count']

    def get_book_count(self, obj):
        # obj is the Author model instance
        return obj.books.count()  # DANGER: Hits the DB for EVERY author!
```

---

### The Hidden N+1 Bug

Look closely at `obj.books.count()`.

If you serialize a list of 100 authors:

```python
authors = Author.objects.all()  # 1 query
serializer = AuthorSerializer(authors, many=True)
return Response(serializer.data)
```

As DRF loops through the 100 authors to generate the JSON response, `get_book_count()` executes **100 individual `SELECT COUNT(*)` queries**!

One query to fetch authors, plus 100 queries inside the serializer: classic N+1 disaster.

### The Fix: Annotate in the View Queryset

Never perform database queries inside a `SerializerMethodField` if you can calculate the data in SQL:

```python
# views.py
from django.db.models import Count

class AuthorViewSet(ModelViewSet):
    # Calculate book_count in SQL using annotate()!
    queryset = Author.objects.annotate(book_count=Count('books'))
    serializer_class = AuthorSerializer
```

```python
# serializers.py
class AuthorSerializer(serializers.ModelSerializer):
    # Simply read the annotated field directly without SerializerMethodField!
    book_count = serializers.IntegerField(read_only=True)

    class Meta:
        model = Author
        fields = ['id', 'name', 'book_count']
```

Now, the entire response takes **exactly 1 database query** regardless of how many authors are serialized.
