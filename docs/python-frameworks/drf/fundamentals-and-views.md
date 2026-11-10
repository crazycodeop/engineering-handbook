# DRF Fundamentals & Views

> This guide explores Django REST Framework (DRF), comparing APIViews, Generic Views, and ViewSets, and explaining how DRF handles request parsing, content negotiation, and routing under the hood.

---

# 1. Why Django REST Framework Exists

## Interview Question

**Why do we need Django REST Framework? Django already has Views and `JsonResponse`. What problem does DRF actually solve?**

---

### Answer

This is the exact question many developers ask when they first transition from building server-rendered Django apps to REST APIs.

Suppose DRF did not exist.

Imagine building a standard API endpoint: `POST /api/orders/`

Using only standard Django views, you would have to write:

```python
import json
from django.http import JsonResponse, HttpResponseBadRequest, HttpResponseNotAllowed
from django.views.decorators.csrf import csrf_exempt

@csrf_exempt
def create_order(request):
    if request.method != 'POST':
        return HttpResponseNotAllowed(['POST'])

    # 1. Parse JSON body manually
    try:
        data = json.loads(request.body)
    except json.JSONDecodeError:
        return HttpResponseBadRequest("Malformed JSON")

    # 2. Manual Authentication check
    if not request.user.is_authenticated:
        return JsonResponse({"error": "Unauthorized"}, status=401)

    # 3. Manual Data Validation
    item_id = data.get('item_id')
    quantity = data.get('quantity')
    if not item_id or not quantity or quantity <= 0:
        return JsonResponse({"error": "Invalid item_id or quantity"}, status=400)

    # 4. Business logic
    order = Order.objects.create(user=request.user, item_id=item_id, quantity=quantity)

    # 5. Manual serialization of model to dict
    return JsonResponse({
        "id": order.id,
        "item_id": order.item_id,
        "quantity": order.quantity,
        "status": order.status
    }, status=201)
```

Now imagine building 150 different API endpoints.

Every single endpoint requires the exact same repetitive plumbing:
- Reading and parsing request bodies
- Validating datatypes and constraints
- Checking authentication and authorization
- Handling HTTP status codes and exceptions
- Serializing database models into JSON dictionaries
- Supporting pagination, filtering, and rate limiting

Every engineering team ends up writing their own brittle set of custom helpers.

**DRF standardizes this entire infrastructure.**

Instead of reinventing API plumbing, DRF provides battle-tested abstractions:
- `Request` and `Response` objects
- Declarative serializers for validation and data conversion
- `APIView`, `GenericAPIView`, and `ModelViewSet`
- Pluggable authentication and permission layers
- Browsable API for instant interactive debugging

---

# 2. View Hierarchy: APIView vs GenericAPIView vs ViewSet

## Interview Question

**Explain the difference between `APIView`, `GenericAPIView`, and `ModelViewSet`. How do you decide which one to use?**

---

### The Abstraction Spectrum

DRF provides views at different levels of abstraction:

```
Low Abstraction                                           High Abstraction
(More manual code,                                       (Less code,
 maximum control)                                         standardized CRUD)
       │                                                         │
    APIView ──────► GenericAPIView + Mixins ──────► ModelViewSet
```

---

### Level 1: `APIView` (Maximum Control)

`APIView` is the base class for all DRF views. It directly subclasses Django's standard `View`.

```python
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status

class PaymentWebhookAPIView(APIView):
    def post(self, request):
        event = request.data.get('event')
        # Custom payment processing logic ...
        return Response({"received": True}, status=status.HTTP_200_OK)
```

**What `APIView` does automatically:**
1. Wraps the raw Django `HttpRequest` inside DRF's enhanced `Request` object (`request.data`).
2. Runs authentication checks (`authentication_classes`).
3. Runs permission checks (`permission_classes`).
4. Executes throttling checks (`throttle_classes`).
5. Handles exceptions and converts them into standardized JSON error responses.

**When to use `APIView`:**
Use `APIView` when your endpoint does **not** represent a standard CRUD database resource (e.g. Stripe webhooks, user login/logout, calculation tools, external API proxying).

---

### Level 2: `GenericAPIView` & Mixins (Standardized Query Logic)

After building a few APIs, you notice that almost every CRUD endpoint follows the exact same pattern:
1. Query a database table.
2. Filter the QuerySet.
3. Pass results to a Serializer.
4. Return a response.

`GenericAPIView` standardizes this by adding two core attributes:
- `queryset`: What data are we querying?
- `serializer_class`: How do we validate and format that data?

DRF provides mixin classes that implement specific HTTP actions:
- `ListModelMixin` (`.list()`)
- `CreateModelMixin` (`.create()`)
- `RetrieveModelMixin` (`.retrieve()`)
- `UpdateModelMixin` (`.update()`)
- `DestroyModelMixin` (`.destroy()`)

Django wraps these mixins into ready-to-use concrete generic views:

```python
from rest_framework.generics import ListCreateAPIView, RetrieveUpdateDestroyAPIView
from .models import Product
from .serializers import ProductSerializer

# Handles GET (list all) and POST (create new)
class ProductListCreateAPIView(ListCreateAPIView):
    queryset = Product.objects.filter(is_active=True)
    serializer_class = ProductSerializer

# Handles GET (detail), PUT/PATCH (update), and DELETE (remove)
class ProductDetailAPIView(RetrieveUpdateDestroyAPIView):
    queryset = Product.objects.filter(is_active=True)
    serializer_class = ProductSerializer
```

---

### Level 3: `ModelViewSet` (Complete Resource Grouping)

In the generic views above, `ProductListCreateAPIView` and `ProductDetailAPIView` are still two separate classes with separate URL patterns.

A `ViewSet` groups all operations for the same resource into a single class:

```python
from rest_framework.viewsets import ModelViewSet
from .models import Product
from .serializers import ProductSerializer

class ProductViewSet(ModelViewSet):
    queryset = Product.objects.filter(is_active=True)
    serializer_class = ProductSerializer
```

With just 4 lines of code, `ModelViewSet` provides:
- `GET /products/` $\rightarrow$ `list()`
- `POST /products/` $\rightarrow$ `create()`
- `GET /products/42/` $\rightarrow$ `retrieve()`
- `PUT /products/42/` $\rightarrow$ `update()`
- `PATCH /products/42/` $\rightarrow$ `partial_update()`
- `DELETE /products/42/` $\rightarrow$ `destroy()`

---

### Summary: Which one do I choose in interviews?

```
Is the endpoint a standard CRUD database resource?
  ├── YES: Use ModelViewSet (or GenericAPIView if you only need List or Detail)
  └── NO (Custom workflow, webhook, OAuth, RPC): Use APIView
```

---

# 3. ViewSets and Routers

## Interview Question

**How do DRF Routers work, and what is the difference between `SimpleRouter` and `DefaultRouter`?**

---

### Answer

Normally, you manually connect views to URLs in `urls.py`.

Because `ViewSet` classes combine multiple HTTP actions (`list`, `create`, `retrieve`, `update`, `destroy`), connecting them manually would require cumbersome dictionary mappings:

```python
# MANUAL MAPPING (Tedious)
path('products/', ProductViewSet.as_view({'get': 'list', 'post': 'create'})),
path('products/<int:pk>/', ProductViewSet.as_view({'get': 'retrieve', 'put': 'update'})),
```

DRF **Routers** automate this wiring:

```python
# urls.py
from rest_framework.routers import DefaultRouter
from .views import ProductViewSet

router = DefaultRouter()
router.register(r'products', ProductViewSet, basename='product')

urlpatterns = [
    path('', include(router.urls)),
]
```

---

### `SimpleRouter` vs `DefaultRouter`

| Feature | `SimpleRouter` | `DefaultRouter` |
|---|---|---|
| **Root API View** | None (Accessing `/` returns 404) | Automatically generates an API root listing all registered endpoints |
| **Format Suffixes** | No format suffixes by default | Automatically includes `.json` routes (e.g. `/products.json`) |
| **Use Case** | Lean production APIs where root discovery isn't needed | APIs that leverage the Browsable API for developer exploration |

---

# 4. DRF `Request` and `Response` Objects

## Interview Question

**How does DRF's `Request` object differ from Django's native `HttpRequest`, and why should you use `request.data` instead of `request.POST`?**

---

### Answer

DRF wraps Django's standard `HttpRequest` inside an extended `rest_framework.request.Request` object.

The most critical difference is how data is accessed:

```python
# Django native HttpRequest
request.POST  # ONLY parses application/x-www-form-urlencoded or multipart/form-data
request.body  # Raw byte string of incoming payload

# DRF Request
request.data  # Automatically parses JSON, form data, or multipart payloads into a Python dictionary!
```

### Why `request.data` is superior:
1. **Handles JSON natively**: If a client sends `Content-Type: application/json`, Django's `request.POST` is completely empty! In DRF, `request.data` automatically parses the JSON body into clean Python primitives.
2. **Handles `PUT` and `PATCH` uploads**: Django's native `request.POST` only works for `POST` requests. If a client submits form data or file uploads via `PUT` or `PATCH`, native Django ignores it. DRF's `request.data` handles all HTTP methods identically.

### Accessing Query Parameters:
Always use `request.query_params` in DRF instead of `request.GET`:

```python
# DRF Best Practice
search_term = request.query_params.get('search')
```

---

# 5. Parsers, Renderers & Content Negotiation

## Interview Question

**What happens during Content Negotiation in DRF, and how do Parsers and Renderers work?**

---

### Answer

REST APIs should be capable of accepting and returning multiple data formats (e.g. JSON, XML, YAML).

DRF accomplishes this via **Content Negotiation**:

```
Client sends request
    │
    ├── Content-Type: application/json ──► Selected Parser: JSONParser
    │                                      (Converts raw bytes to request.data)
    │
    └── Accept: application/json ────────► Selected Renderer: JSONRenderer
                                           (Converts Response(data) to JSON bytes)
```

1. **Parsers** inspect the incoming `Content-Type` header (e.g. `application/json` or `multipart/form-data`) and determine how to deserialize the raw request body into `request.data`.
2. **Renderers** inspect the client's `Accept` header. If the client is a browser requesting `text/html`, DRF uses the `BrowsableAPIRenderer` to display an interactive web interface. If the client is a mobile app requesting `application/json`, DRF uses the `JSONRenderer` to return raw JSON bytes.
