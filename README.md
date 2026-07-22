# Backend Assignment: Pharmacy Management System API
## Objective:

Build a backend system to manage medicines, customers, prescriptions, sales, and staff operations with secure access, relational data modeling, and scalable APIs.


## Core Requirements:



### 1. Authentication & Authorization

- JWT-based authentication
- Role-based access control:
- Admin: full system access
- Pharmacist: manage medicines, prescriptions, and sales
- Staff: view data and process assigned sales
- Route protection using guards




### 2. System Modules

- User & staff management
- Medicine inventory management
- Sales and billing
- Supplier management




### 3. Database & Relationships
Implement all major relational patterns:

- One-to-One (user → staff profile)
- One-to-Many (supplier → medicines, customer → prescriptions)
- Many-to-Many (prescriptions → medicines, sales → medicines)
- Ensure proper constraints and data integrity.




### 4. Inventory Management

- Add, update, and deactivate medicines
- Track stock levels and expiry awareness
- Prevent sales of out-of-stock or expired medicines
- Link medicines to suppliers




### 5. Sales & Billing

- Process medicine sales
- Automatically reduce inventory on sale
- Associate sales with staff and customers
- Support sale status tracking (completed, cancelled)




### 6. Pagination, Filtering & Sorting

- Paginated list APIs for medicines, prescriptions, sales, and suppliers
- Filter by availability, expiry range, date range, customer, or staff
- Sorting via query parameters




### 7. Validation & Error Handling

- DTO-based input validation
- Meaningful HTTP error responses
- Centralized exception handling
