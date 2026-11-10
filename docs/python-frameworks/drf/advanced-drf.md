# Advanced DRF & Production Best Practices

> This guide explores advanced DRF engineering patterns: custom ViewSet actions, dynamic serializers, query optimization, bulk database operations, automated API testing, and production tuning.

---

# 1. Custom ViewSet Actions with `@action`

## Interview Question

**How do you add non-standard endpoints to a `ModelViewSet` using the `@action` decorator? What is the difference between `detail=True` and `detail=False`?**

---

### Answer

Standard `ModelViewSet` actions cover basic CRUD: `list`, `create`, `retrieve`, `update`, `partial_update`, and `destroy`.

In real systems, resources often have domain actions that don't fit into plain CRUD (e.g. canceling an order, resetting a password, or archiving an account).

The `@action` decorator lets you add custom routes directly to a `ViewSet`:

```python
from rest_framework.decorators import action
from rest_framework.response import Response
from rest_framework import status
from rest_framework.viewsets import ModelViewSet

class OrderViewSet(ModelViewSet):
    queryset = Order.objects.all()
    serializer_class = OrderSerializer

    # 1. DETAIL=TRUE: Operates on a SINGLE instance (e.g. POST /orders/42/cancel/)
    @action(detail=True, methods=['post'], url_path='cancel')
    def cancel_order(self, request, pk=None):
        order = self.get_object()  # Automatically checks permissions & 404!
        
        if order.status != 'pending':
            return Response(
                {"error": "Only pending orders can be canceled."},
                status=status.HTTP_400_BAD_REQUEST
            )
            
        order.status = 'canceled'
        order.save()
        return Response({"status": "order canceled"})

    # 2. DETAIL=FALSE: Operates on the ENTIRE collection (e.g. GET /orders/summary/)
    @action(detail=False, methods=['get'], url_path='summary')
    def order_summary(self, request):
        total_revenue = Order.objects.filter(user=request.user).aggregate(Sum('total'))
        return Response({"total_revenue": total_revenue})
```

---

### `detail=True` vs `detail=False`

| Feature | `detail=True` | `detail=False` |
|---|---|---|
| **URL Pattern** | `/orders/{pk}/action-name/` | `/orders/action-name/` |
| **Object Access** | Calls `self.get_object()` to fetch the specific model instance | Operates on `self.get_queryset()` or collection-level data |
| **Permissions** | Runs both `has_permission` and `has_object_permission` | Runs only view-level `has_permission` |

---

# 2. Dynamic Serializers & Action-Based Querysets

## Interview Question

**How do you use different serializers for the `list` action vs the `retrieve` action in DRF, and why is this critical for performance?**

---

### Answer

In a list view (`GET /products/`), returning 50 products with full descriptions, customer reviews, seller profiles, and nested image galleries wastes bandwidth and causes massive database queries.

A list view should return a lightweight summary, while the detail view (`GET /products/42/`) returns the complete object graph.

You can switch serializers dynamically by overriding `get_serializer_class()`:

```python
class ProductViewSet(ModelViewSet):
    queryset = Product.objects.all()

    def get_serializer_class(self):
        if self.action == 'list':
            return ProductListLightweightSerializer
        elif self.action in ['create', 'update', 'partial_update']:
            return ProductWriteSerializer
        return ProductDetailComprehensiveSerializer
```

---

### Action-Based Queryset Optimization

Similarly, you should only load related tables when that specific action actually needs them:

```python
class ProductViewSet(ModelViewSet):
    def get_queryset(self):
        queryset = Product.objects.filter(is_active=True)

        if self.action == 'list':
            # Lightweight list only needs category name
            return queryset.select_related('category')
        
        if self.action == 'retrieve':
            # Detail view needs category, seller, and all reviews
            return queryset.select_related('category', 'seller').prefetch_related('reviews')

        return queryset
```

This ensures the list endpoint stays blazing fast (10ms) without loading unnecessary joins or prefetch queries.

---

# 3. Bulk Operations in DRF

## Interview Question

**How do you handle bulk creation and bulk updates in DRF using `many=True`?**

---

### Bulk Creation

To create multiple objects in a single API call, pass `many=True` to the serializer:

```python
# View
class BulkProductCreateView(APIView):
    def post(self, request):
        serializer = ProductSerializer(data=request.data, many=True)
        serializer.is_valid(raise_exception=True)
        
        # Serializer's create method will be called with a list of dictionaries
        serializer.save()
        return Response(serializer.data, status=status.HTTP_201_CREATED)
```

In your serializer, use Django's `bulk_create` for database efficiency:

```python
class ProductSerializer(serializers.ModelSerializer):
    class Meta:
        model = Product
        fields = ['name', 'price', 'sku']

    def create(self, validated_data):
        if isinstance(validated_data, list):
            instances = [Product(**item) for item in validated_data]
            return Product.objects.bulk_create(instances)
        return super().create(validated_data)
```

Now, creating 500 items takes **1 single SQL `INSERT`** instead of 500 individual database roundtrips.

---

# 4. Testing DRF APIs with `APITestCase`

## Interview Question

**How do you write fast, reliable unit and integration tests for DRF endpoints? What is `client.force_authenticate()`?**

---

### Answer

DRF provides `APITestCase` and `APIClient` (built on top of Django's test framework):

```python
# tests/test_orders.py
from django.contrib.auth import get_user_model
from rest_framework.test import APITestCase
from rest_framework import status
from orders.models import Order

User = get_user_model()

class OrderAPITests(APITestCase):
    def setUp(self):
        # Create test users
        self.user = User.objects.create_user(username='alice', password='password123')
        self.other_user = User.objects.create_user(username='bob', password='password123')
        
        self.order = Order.objects.create(user=self.user, total=100)

    def test_unauthenticated_user_cannot_access_orders(self):
        # GET without authentication should return 401
        response = self.client.get('/api/v1/orders/')
        self.assertEqual(response.status_code, status.HTTP_401_UNAUTHORIZED)

    def test_authenticated_user_can_list_own_orders(self):
        # FAST AUTHENTICATION: bypasses password hashing & token generation
        self.client.force_authenticate(user=self.user)

        response = self.client.get('/api/v1/orders/')
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(len(response.data['results']), 1)
        self.assertEqual(response.data['results'][0]['id'], self.order.id)

    def test_user_cannot_update_other_users_order(self):
        # Bob tries to modify Alice's order
        self.client.force_authenticate(user=self.other_user)

        response = self.client.patch(f'/api/v1/orders/{self.order.id}/', {'total': 500})
        # Should be forbidden (or 404 if filtered by get_queryset)
        self.assertIn(response.status_code, [status.HTTP_403_FORBIDDEN, status.HTTP_404_NOT_FOUND])
```

---

### Why `client.force_authenticate()` is critical in tests

In an interview, mention why you use `force_authenticate` instead of logging in via `client.post('/api/token/', ...)`:

Password hashing (PBKDF2 / Argon2) is intentionally designed to be CPU-intensive (taking 200–300ms per attempt).

If your test suite has 300 tests, and every test calls the login endpoint to hash a password, your test suite wastes **60–90 seconds just hashing passwords**!

`self.client.force_authenticate(user=self.user)` bypasses the authentication handshake and directly injects `request.user` into the test client, making your test suite run in seconds.

---

# 5. Production Tuning: Disabling the Browsable API

## Interview Question

**What is the Browsable API in DRF, and why should you disable it in production?**

---

### Answer

The Browsable API is one of DRF's best developer features. When you open an endpoint in Chrome, DRF renders a rich HTML page with forms to test `POST` and `PUT` requests.

However, in production:

1. **CPU & Memory Overhead**: Generating the Browsable API HTML requires rendering Jinja/Django templates, parsing forms, and serializing HTML inputs. For an API that only serves mobile apps and frontend SPAs, this consumes significant CPU and RAM.
2. **Security & Information Disclosure**: While not inherently insecure, exposing an interactive testing interface on public production endpoints invites automated reconnaissance from attackers.

### How to configure production renderers:

```python
# settings.py
if not DEBUG:
    REST_FRAMEWORK['DEFAULT_RENDERER_CLASSES'] = [
        'rest_framework.renderers.JSONRenderer',  # ONLY return JSON in production!
    ]
```

This guarantees that every response in production is pure, high-speed JSON with zero HTML template rendering overhead.
