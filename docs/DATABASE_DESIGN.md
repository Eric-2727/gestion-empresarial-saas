# Diseño de Base de Datos — MVP

Este documento registra las decisiones de diseño de la base de datos para el MVP del sistema de gestión empresarial. El objetivo es que cualquier persona (incluido tú mismo en el futuro) entienda **por qué** cada tabla y cada decisión existen, no solo su estructura.

## Alcance del MVP

Enfocado en negocios tipo **tienda / minimercado** (sin módulo de mesas). El núcleo (productos, inventario, compras, ventas) está diseñado para que restaurantes/cafeterías puedan agregarse después como un módulo adicional, sin reescribir esta base.

## Estrategia multi-tenant

**Shared schema con discriminador `company_id`.**

- Una sola base de datos, unas solas tablas.
- Cada tabla que depende de una empresa tiene una columna `company_id`.
- Toda consulta al backend DEBE filtrar por `company_id` sin excepción — este es el riesgo de seguridad #1 del proyecto (fuga de datos entre empresas).
- Se descartaron "database per tenant" y "schema per tenant" por la complejidad operativa que agregan sin necesidad en esta etapa (backups, migraciones y mantenimiento multiplicados).

## Roles (MVP)

Se usa un campo simple `role` en `users` (`admin`, `cashier`), en vez de tablas separadas `roles`/`permissions`. Un sistema de permisos granular se considera sobre-ingeniería para el MVP — se puede migrar más adelante si aparece la necesidad real.

## Entidades

### companies
Tabla raíz del sistema multi-tenant. Todas las demás tablas (excepto ella misma) tienen `company_id` apuntando aquí.

- id (PK)
- name
- tax_id
- business_type (`store`, `restaurant`, `hardware_store`, etc.)
- currency
- created_at
- updated_at

### users
Personas que inician sesión.

- id (PK)
- company_id (FK → companies.id)
- name
- email (único por empresa, no global)
- password_hash
- role (`admin`, `cashier`)
- is_active
- created_at
- updated_at

### categories
Organización de productos para filtros y reportes.

- id (PK)
- company_id (FK → companies.id)
- name
- created_at

### products
- id (PK)
- company_id (FK → companies.id)
- category_id (FK → categories.id, nullable)
- name
- sku
- barcode (nullable)
- purchase_price
- sale_price
- stock — valor cacheado, actualizado en cada movimiento (no calculado al vuelo, por rendimiento del POS)
- min_stock
- is_active — eliminación LÓGICA, no física (preserva historial de ventas/compras pasadas)
- created_at
- updated_at

### inventory_movements
Historial de entradas/salidas. Fuente de auditoría, no fuente de verdad del stock actual (esa es `products.stock`).

- id (PK)
- company_id (FK → companies.id)
- product_id (FK → products.id)
- type (`purchase`, `sale`, `adjustment`)
- quantity (positivo o negativo según el tipo)
- user_id (FK → users.id)
- created_at

### suppliers
- id (PK)
- company_id (FK → companies.id)
- name
- phone
- email
- created_at

### purchases / purchase_items
Relación uno a muchos: una compra tiene varios ítems de producto.

**purchases**
- id (PK)
- company_id (FK → companies.id)
- supplier_id (FK → suppliers.id)
- user_id (FK → users.id)
- total
- status (`pending`, `confirmed`) — el inventario solo se actualiza al confirmar
- created_at

**purchase_items**
- id (PK)
- purchase_id (FK → purchases.id)
- product_id (FK → products.id)
- quantity
- unit_price

### customers
- id (PK)
- company_id (FK → companies.id)
- name
- identification (nullable)
- phone
- email
- created_at

### sales / sale_items
Relación uno a muchos, igual que compras. `unit_price` se congela en `sale_items` al momento de la venta (no referencia el precio actual del producto), para que cambios de precio futuros no alteren facturas pasadas. Los campos de `sales` (subtotal, tax, discount, total) ya cubren lo necesario para generar la factura en PDF sin cambios estructurales futuros.

**sales**
- id (PK)
- company_id (FK → companies.id)
- customer_id (FK → customers.id, nullable)
- user_id (FK → users.id)
- subtotal
- tax
- discount
- total
- payment_method (`cash`, `card`, `transfer`)
- status (`completed`, `cancelled`)
- created_at

**sale_items**
- id (PK)
- sale_id (FK → sales.id)
- product_id (FK → products.id)
- quantity
- unit_price

### expenses
- id (PK)
- company_id (FK → companies.id)
- category (texto simple: `rent`, `payroll`, `services`, etc.)
- description
- amount
- user_id (FK → users.id)
- created_at

## Relaciones (resumen)

```
companies (1) ──── (N) users
companies (1) ──── (N) categories
companies (1) ──── (N) products ──── (N) inventory_movements
companies (1) ──── (N) suppliers ──── (N) purchases ──── (N) purchase_items
companies (1) ──── (N) customers ──── (N) sales ──── (N) sale_items
companies (1) ──── (N) expenses
```

## Explícitamente fuera del MVP (para fases posteriores)

- Mesas / pedidos por mesa (restaurantes, cafeterías).
- Tablas separadas de roles/permisos granulares.
- Facturación electrónica oficial (DIAN).
- Asistente de IA (no requiere cambios en este modelo, solo consume los endpoints del core una vez existan).
