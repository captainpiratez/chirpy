# Chirpy

Chirpy is a backend HTTP server project built in Go, as part of the "Learn HTTP Servers in Go" course on Boot.dev. This project serves as a practical application of building a production-style web server from scratch, without using any major Go frameworks.

## Project Overview

The primary goal of this project is to build a RESTful API for a simple Twitter-like service called "Chirpy". It includes features like creating users, authentication with JWTs, creating and retrieving "chirps", and interacting with a database. The project also includes webhooks for the Polka service and administrative endpoints for metrics and database resets.

### Technologies Used

-   **Go:** For the core application logic and HTTP server (using the standard library `net/http`).
-   **PostgreSQL:** As the database for storing user and chirp data.
-   **goose:** For database migrations.
-   **sqlc:** For generating type-safe SQL query code.
-   **JSON Web Tokens (JWTs):** For authentication, using the `github.com/golang-jwt/jwt/v5` library.
-   **argon2id:** For secure password hashing.

## Configuration

The application is configured using environment variables. You can place these in a `.env` file in the root of the project, and they will be loaded automatically.

| Variable      | Description                                                                                             | Example                                                               |
| :------------ | :------------------------------------------------------------------------------------------------------ | :-------------------------------------------------------------------- |
| `DB_URL`      | The connection string for your PostgreSQL database.                                                     | `postgres://user:password@localhost:5432/chirpy?sslmode=disable`      |
| `JWT_SECRET`  | A secret key used to sign and verify JWT access tokens.                                                 | `a-very-secret-key`                                                   |
| `POLKA_KEY`   | An API key for validating webhooks from the Polka service.                                              | `your-polka-api-key`                                                  |
| `PLATFORM`    | The deployment environment. Set to `dev` to enable the `/admin/reset` endpoint.                         | `dev`                                                                 |

## Getting Started

To get a local copy up and running, follow these simple steps.

### Prerequisites

-   Go (latest version recommended)
-   PostgreSQL running locally or in a Docker container.
-   `goose` installed for database migrations.
-   `sqlc` installed for type-safe SQL query generation.

### Installation & Running

1.  **Clone the repository:**
    ```sh
    git clone https://github.com/captainpiratez/chirpy.git
    cd chirpy
    ```

2.  **Install tools:**
    Install `goose` and `sqlc`:
    ```sh
    go install github.com/pressly/goose/v3/cmd/goose@latest
    go install github.com/sqlc-dev/sqlc/cmd/sqlc@latest
    ```

3.  **Install Go dependencies:**
    ```sh
    go mod tidy
    ```

4.  **Set up the database:**
    Ensure your PostgreSQL server is running. Create a database for the project (e.g., `chirpy`).

5.  **Configure environment variables:**
    Create a `.env` file in the project root and add the necessary variables (see [Configuration](#configuration)).

6.  **Run database migrations:**
    Use `goose` to apply the SQL migrations from the `sql/schema` directory.
    ```sh
    # The DB_URL from your .env file will be used
    goose -dir sql/schema postgres "$DB_URL" up
    ```

7.  **Generate SQL code:**
    ```sh
    sqlc generate
    ```

8.  **Run the server:**
    ```sh
    go run main.go
    ```
    The server will start on `http://localhost:8080`.

## API Endpoints

The server exposes the following RESTful API endpoints.

### App

| Method | Path   | Description                                                                 |
| :----- | :----- | :-------------------------------------------------------------------------- |
| `GET`  | `/app` | Serves the frontend application files from the project's root directory.    |

### Health & Metrics

| Method | Path                  | Description                                                                 |
| :----- | :-------------------- | :-------------------------------------------------------------------------- |
| `GET`  | `/api/healthz`        | Responds with `200 OK` if the server is running.                            |
| `GET`  | `/admin/metrics`      | Responds with an HTML page showing the number of hits to the `/app` endpoint. |
| `POST` | `/admin/reset`        | Resets server metrics and the database. Only available if `PLATFORM=dev`.   |

### Users

| Method | Path          | Description                   | Request Body                            | Response Body (Success)                 |
| :----- | :------------ | :---------------------------- | :-------------------------------------- | :-------------------------------------- |
| `POST` | `/api/users`  | Creates a new user.           | `{"email": "...", "password": "..."}`   | `{"id": "...", "email": "...", "is_chirpy_red": false}` |
| `PUT`  | `/api/users`  | Updates an authenticated user's email and password. | `{"email": "...", "password": "..."}`   | `{"id": "...", "email": "...", "is_chirpy_red": false}` |

### Authentication

| Method | Path            | Description                                       | Request Body                          | Response Body (Success)                               |
| :----- | :-------------- | :------------------------------------------------ | :------------------------------------ | :---------------------------------------------------- |
| `POST` | `/api/login`    | Logs in a user and returns JWTs.                  | `{"email": "...", "password": "..."}` | `{"id": "...", "email": "...", "is_chirpy_red": false, "token": "...", "refresh_token": "..."}` |
| `POST` | `/api/refresh`  | Issues a new access token using a valid refresh token. | _(Refresh token in `Authorization: Bearer <token>`)_ | `{"token": "..."}`                                    |
| `POST` | `/api/revoke`   | Revokes a refresh token.                          | _(Refresh token in `Authorization: Bearer <token>`)_ | `204 No Content`                                      |

### Chirps

Chirps are limited to 140 characters. The words `kerfuffle`, `sharbert`, and `fornax` are censored and replaced with `****`.

| Method   | Path                  | Description                               | Authorization      | Query Params | Response Body (Success) |
| :------- | :-------------------- | :---------------------------------------- | :----------------- | :----------- | :---------------------- |
| `POST`   | `/api/chirps`         | Creates a new chirp.                      | `Bearer <token>`   | _none_       | `{"id": "...", "body": "...", "user_id": "..."}` |
| `GET`    | `/api/chirps`         | Retrieves all chirps.                     | _none_             | `author_id` (uuid), `sort` (`asc`/`desc`) | `[{"id": "...", "body": "...", "user_id": "..."}, ...]` |
| `GET`    | `/api/chirps/{id}`    | Retrieves a single chirp by its ID.       | _none_             | _none_       | `{"id": "...", "body": "...", "user_id": "..."}` |
| `DELETE` | `/api/chirps/{id}`    | Deletes a chirp if the user is the author. | `Bearer <token>`   | _none_       | `204 No Content`        |

### Webhooks

| Method | Path                 | Description                               | Authorization         |
| :----- | :------------------- | :---------------------------------------- | :-------------------- |
| `POST` | `/api/polka/webhooks`| Handles user upgrade webhooks from the Polka service. | `ApiKey <polka_key>`  |

## Project Structure

```
.
├── internal/
│   ├── auth/       # Authentication logic (JWTs, password hashing)
│   └── database/   # Database connection, models, and generated sqlc code
├── sql/
│   ├── queries/    # SQL queries for sqlc
│   └── schema/     # Goose database schema migrations
├── main.go         # Main application entry point and router setup
├── *.go            # HTTP handlers for different API resources
└── README.md       # This file
```
