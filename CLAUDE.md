# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Django REST Framework application for managing a gas (LPG) distribution agency. The system tracks consumers, delivery routes, connections, and various administrative aspects of gas distribution.

## Development Commands

### Running the Application
```bash
python manage.py runserver
```

### Database Operations
```bash
# Apply migrations
python manage.py migrate

# Create migrations after model changes
python manage.py makemigrations

# Access Django shell for database queries
python manage.py shell
```

### Testing
```bash
# Run all tests
python manage.py test

# Run tests for a specific app
python manage.py test <app_name>
```

### Data Population Commands
The application includes several custom management commands in `commands/management/commands/`:

```bash
# Populate lookup tables
python manage.py populate_lookups

# Populate consumers from data
python manage.py populate_consumers

# Populate delivery routes
python manage.py populate_routes

# Populate delivery persons
python manage.py populate_delivery_persons

# Assign consumers to routes
python manage.py populate_consumers_route

# Data validation and review
python manage.py review_consumers
python manage.py review_lookups
python manage.py find_duplicate_ration_cards
python manage.py find_duplicate_sv_numbers
python manage.py shared_bluebook
```

## Architecture

### Core Design Patterns

**1. Assignment System with History Tracking**
The application uses a sophisticated assignment pattern for both consumer-route and delivery-route relationships:

- `ConsumerRouteAssignment` - One-to-one relationship between Consumer and Route
- `ConsumerRouteAssignmentHistory` - Audit trail for all assignment changes
- `DeliveryRouteAssignment` - One-to-one relationship between Route and DeliveryPerson
- `DeliveryRouteAssignmentHistory` - Audit trail for delivery assignments

Both history models use Django signals (`consumers/signals.py`, `delivery/signals.py`) to automatically log CREATE, UPDATE, and DELETE actions. History records denormalize critical fields (names, codes) to preserve data even if parent records are deleted.

**2. Generic Relations for Addresses and Contacts**
The `address` app provides reusable Address and Contact models using Django's ContentType framework. Any model can have multiple addresses and contacts through GenericRelation:

```python
# In Consumer model
addresses = GenericRelation('address.Address', related_query_name='consumer')
contacts = GenericRelation('address.Contact', related_query_name='consumer')
```

This pattern is used by Consumer and DeliveryPerson models.

**3. Lookup Tables Pattern**
The `lookups` app centralizes reference data (ConsumerCategory, ConsumerType, BPLType, DCTType, ConnectionType, MarketType). All lookup models follow the same structure with name, description, and unique constraints.

### App Responsibilities

- **consumers** - Core consumer management, route assignments, and consumer-specific history tracking
- **delivery** - Delivery person management and route-to-delivery-person assignments
- **routes** - Route definitions and route areas
- **connections** - Connection details linking consumers to products (SV numbers, service dates)
- **address** - Generic address and contact management for any model
- **products** - Product catalog (Product, ProductVariant, Unit)
- **schemes** - Government schemes and subsidy details
- **lookups** - Reference data and classification types
- **lgd** - Local Government Directory information (district, sub-district, village codes)
- **commands** - Houses all custom management commands for data operations
- **abstracts** - Base models (TimeStampedModel) for inheritance

### Key Model Relationships

```
Consumer (1) ←→ (1) ConsumerRouteAssignment →→ (N) Route
Consumer (1) →→ (N) ConnectionDetails →→ (1) ProductVariant
Route (1) ←→ (1) DeliveryRouteAssignment →→ (N) DeliveryPerson
Route (1) →→ (N) RouteArea

[Any Model] →→ (N) Address (via GenericForeignKey)
[Any Model] →→ (N) Contact (via GenericForeignKey)
```

Consumer has foreign keys to: ConsumerCategory, ConsumerType, BPLType, DCTType, Scheme
ConnectionDetails has foreign keys to: Consumer, ConnectionType, ProductVariant

### Database Configuration

Currently configured for SQLite (development). MySQL configuration is commented out in `core/settings.py:98-107`. To switch:

1. Uncomment MySQL DATABASES config
2. Comment out SQLite config
3. Ensure MySQL credentials match your setup
4. Run migrations

### REST API Structure

API endpoints are structured under `/api/` prefix:

- `/api/consumers/` - Consumer CRUD and custom actions (see `consumers/urls.py:13-42` for full endpoint list)
- `/api/routes/` - Route management
- `/api/delivery/` - Delivery person management

The application uses:
- Django REST Framework with ViewSets
- Default router for automatic endpoint generation
- PageNumberPagination (20 items per page)
- django-filter for queryset filtering
- CORS enabled (allow all origins in development)

### Important Implementation Notes

**Consumer Model Flexibility**
- `ration_card_num` and `blue_book` do NOT have unique constraints (multiple consumers can share these)
- Route assignment is through separate `ConsumerRouteAssignment` model, not a direct FK
- Generic relations enable multiple addresses and contacts per consumer

**Signal-Based Audit Logging**
Both consumer and delivery assignment changes are automatically logged via post_save and post_delete signals. The pre_save signal populates denormalized fields in history records. This ensures complete audit trails without explicit save calls.

**Custom Management Commands Location**
All data population and validation commands are in `commands/management/commands/`. This is a dedicated app separate from the core business logic apps.

## Development Workflow

1. When modifying models, always create migrations: `python manage.py makemigrations`
2. For models with generic relations (Address, Contact), ensure GenericRelation is defined on the parent model
3. When adding assignment-style relationships, consider implementing history tracking with signals
4. Use lookup tables (lookups app) for categorical data rather than hardcoding choices
5. Management commands for data operations belong in `commands/management/commands/`
