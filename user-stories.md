# User Stories (initial backlog)

Stories only; none is implemented or scheduled by TASK-001. IDs are referenced by acceptance criteria.

## Customer
- US-01 As a visitor, I can register an account so I can order.
- US-02 As a customer, I can log in and out securely.
- US-03 As a visitor, I can browse products by category and view product details.
- US-04 As a visitor, I can search, filter, and sort products.
- US-05 As a customer, I can add/update/remove items in my cart.
- US-06 As a customer, I can manage saved addresses.
- US-07 As a customer, I can check out and place an order.
- US-08 As a customer, I can pay for my order.
- US-09 As a customer, I can view my order history and order status.
- US-10 As a customer, I can review products I purchased.

## CMS Operators (via the custom CMS; permission shown in brackets)
- US-11 As an operator, I can create/edit/archive products and manage images, price, discount price, stock, SKU, status, visibility, categories, featured flag, and ordering. [`products.*`, `inventory.update`]
- US-12 As an operator, I can create/edit/archive categories with image, description, slug, ordering, visibility. [`categories.*`]
- US-13 As an operator, I can search/filter orders, view details, update status, and add internal notes. [`orders.read/update/notes`]
- US-14 As an operator, I can view payment status and role-appropriate payment information. [`payments.read`, `payments.read_sensitive`]
- US-15 As an operator, I can view customers, their profile, order history, and account status. [`customers.read`]
- US-16 As an operator, I can see product availability, stock, and demand indicators (aggregate). [`products.read`, `dashboard.read`]
- US-17 As a content manager, I can manage homepage sections, banners, announcement bar, featured items, static pages, and SEO metadata without code changes. [`content.*`]
- US-18 As a manager, I can set discount prices, coupons, and campaigns. [`promotions.*`]
- US-19 As an administrator, I can configure integrations and tracking (pixels/analytics) without exposing secrets. [`integrations.manage`]
- US-20 As an operator, I can view a dashboard of orders, revenue, low stock, top products, and trends. [`dashboard.read`]
- US-21 As staff, I can moderate reviews. [`reviews.moderate`]
- US-22 As a super admin, I can manage staff accounts, roles, and permissions, and review the audit log. [`staff.manage`, `audit.read`]
- US-23 As a marketer/owner, I want storefront events (view content, add to cart, initiate checkout, purchase) tracked in browser and server side, deduplicated. [`integrations.manage` to configure]
