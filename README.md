# User Management API

A simple Node.js API to manage users with **CRUD functionality**, validation, and middleware logging.

## Features

- **CRUD Endpoints**:
  - `GET /users` – Get all users
  - `GET /users/:id` – Get a user by ID
  - `POST /users` – Add a new user
  - `PUT /users/:id` – Update an existing user
  - `DELETE /users/:id` – Delete a user
- **Validation**: Ensures `name` and `email` are provided and email format is valid.
- **Middleware**:
  - Logging middleware for all requests.
- Uses **JSON file** (`data/users.json`) as a simple database.
