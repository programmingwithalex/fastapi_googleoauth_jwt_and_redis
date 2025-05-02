<a id="readme-top"></a>

[![Contributors][contributors-shield]][contributors-url]
[![Forks][forks-shield]][forks-url]
[![Stargazers][stars-shield]][stars-url]
[![Issues][issues-shield]][issues-url]
[![BSD-3-Clause License][license-shield]][license-url]

---

<br/>
<div align="center">

© 2025 [GitHub@programmingwithalex](https://github.com/programmingwithalex)

### FastAPI with Google Oauth2, JWT (access and refresh tokens), and Redis (refresh token storage and invalidation)

Lightweight authentication microservice built with `Flask` frontend, `FastAPI` auth service, `JWT` access/refresh tokens, and `Redis`-backed refresh tokens. Features Google `OAuth2` login, secure token issuance, refresh-token rotation, and logout revocation.

[Report Bug](https://github.com/programmingwithalex/fastapi_googleoauth_jwt_and_redis/issues/new?labels=bug&template=bug-report---.md) · [Request Feature](https://github.com/programmingwithalex/fastapi_googleoauth_jwt_and_redis/issues/new?labels=enhancement&template=feature-request---.md)

</div>
<br/>

# Project Overview

This repository contains two components:

1. **web_frontend** (`Flask`):
   - Renders frontend templates (login, dashboard, settings)
   - Delegates auth logic to **auth_service**
   - Sets and clears session cookies

2. **auth_service** (`FastAPI`):
   - **Google OAuth2 login**: Exchange authorization codes for user info and issue tokens
   - **JWT access tokens**: Stateless, short-lived (default 15 minutes)
   - **Opaque refresh tokens**: Stored in Redis with single-use rotation (default 1 day TTL)
   - **Automatic token refresh**: Clients can obtain new access/refresh pairs via `/token/refresh`
   - **Logout revocation**: Invalidate refresh tokens on logout to prevent reuse
   - **Configurable via**: `.env` file or environment variables

## Prerequisites

- `Python` 3.12+
- `Docker` & [`Docker Compose`](https://www.docker.com/products/docker-desktop/)
- `Redis` (can run locally or via Docker)

## Configuration

Create a `.env` file in each service directory with the following keys:

### auth_service (`src/auth/.env`)<a href="#appendix">¹</a>

```ini
GOOGLE_OAUTH_TOKEN_URL="https://oauth2.googleapis.com/token"
GOOGLE_OAUTH_USERINFO_URL="https://www.googleapis.com/oauth2/v3/userinfo"
GOOGLE_OAUTH_CLIENT_ID=<your-google-client-id>
GOOGLE_OAUTH_CLIENT_SECRET=<your-google-client-secret>
SESSION_EXPIRE_TIME_SECONDS=3600
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_SSL=false  # Set to true if using Redis with SSL
REDIS_PORT=6379
WEB_FRONTEND_URL="http://localhost:5000"  # URL of the Auth service
JWT_SECRET_KEY="supersecret"
```

### web_service (`src/web/.env`)

```ini
AUTH_SERVICE_URL=http://auth:8000  # corresponds to docker-compose.yml
SECRET_KEY=<your-flask-secret-key>
COOKIE_SECURE=false  # Set to True in production with HTTPS
PORT_FLASK=5000
FLASK_ENV="development"  # Set to "production" in production
```

## Running Locally with Docker Compose

At the project root, you can spin up the `auth_service`, `web_frontend`, and `Redis`:

```bash
docker-compose up --build
```

- `web_service` available on `http://localhost:5000`
- `auth_service` available on `http://localhost:8000`
- `Redis` on port `6379`

## FastAPI `auth_service` Endpoints

- **GET** `/login/google`
  Returns a Google OAuth2 authorization URL.

- **GET** `/auth/google?code=...`
  Callback for Google. Issues access & refresh tokens, then redirects to your front-end with both tokens.

- **POST** `/token/refresh`
  Request body: `{ "refresh_token": "<token>" }`
  Response: `{ "access_token": "...", "refresh_token": "...", "token_type": "bearer" }`

- **POST** `/verify`
  Header: `Authorization: Bearer <access_token>`
  Response: `{ "user": { "email": "...", "name": "..." } }`

- **POST** `/logout`
  Header: `Authorization: Bearer <access_token>`
  Request body: `{ "refresh_token": "<token>" }`
  Response: `{ "message": "Logged out" }`

## Testing

Run at top-level directory.

```bash
python -m pytest -v
```

## Linting & Formatting

Use pre-commit hooks at the repo root:

```bash
pre-commit install
pre-commit run --all-files
```

Will also run on each commit to GitHub repo.

---

## Possible Issues

1. JWT access token and refresh token **TTL set differently** between frontend and backend
   1. Set in two places:
      1. `FastAPI` auth service JWT token payload `exp`
      2. `Flask` frontend web service Cookies
   2. Possible Issues:
      1. `Flask` frontend Cookie TTL longer the auth token `exp` (Cookie outlives token)
         1. Result:
            1. Browser will keep sending the cookie, but token already expired (or refresh key already vanished from Redis)
            2. Every request will 401 or every refresh attempt will fail—even though the cookie is still present
      2. `Flask` frontend Cookie TTL shorter the auth token `exp` (token outlives Cookie)
         2. Result:
            1. Browser silently drops cookie while token still technically valid on server side
            2. User’s POV → get logged out “too early,” because no cookie = no credentials, even though token not expired yet
   3. Possible solution:
      1. Set TTL values in single location (AWS `SSM Parameter Store`/`AppConfig`)

---

## To Do

- implement on AWS using:
  - `ECS`
    - single `Cluster`
    - two `Services`
      - `Flask` frontend
      - `FastAPI` auth service
  - `ElastiCache` - for `Redis` cluster
  - `CDK`/`Terraform` - IaC
  - `ECS Service Connect`/`API Gateway`
    - for communication between `ECS Services`
    - `ECS Service Connect` - communication if `Services` in same `AWS Cloud Map`
  - `API Gateway` - dedicated URL that can route to "auth service", which can be set in the "web front end" `Service`
    - set `API Gateway` URL via:
      - `SSM Parameter Store` + `ECS Task Definition` env vars
      - `AWS AppConfig`

## Appendix

### ¹Google OAuth Setup

1. Have to setup on [Google Cloud Console](https://console.cloud.google.com)
2. Create project to authenticate with OAuth2.0 on [Google Cloud Console](https://console.cloud.google.com/auth/overview)
3. After creating project:
   1. Clients > Application Type > Web Application
   2. Authorized redirect URIs:
      1. `http://localhost:8000/auth/google`
   3. Copy Google secret variables:
      1. `Client ID`
      2. `Client secret`

## References

[FastApi OAuth2 Scopes](https://fastapi.tiangolo.com/advanced/security/oauth2-scopes/)

[contributors-shield]: https://img.shields.io/github/contributors/programmingwithalex/fastapi_googleoauth_jwt_and_redis?style=for-the-badge
[contributors-url]: https://github.com/programmingwithalex/fastapi_googleoauth_jwt_and_redis/graphs/contributors
[forks-shield]: https://img.shields.io/github/forks/programmingwithalex/fastapi_googleoauth_jwt_and_redis?style=for-the-badge
[forks-url]: https://github.com/programmingwithalex/fastapi_googleoauth_jwt_and_redis/network/members
[stars-shield]: https://img.shields.io/github/stars/programmingwithalex/fastapi_googleoauth_jwt_and_redis?style=for-the-badge
[stars-url]: https://github.com/programmingwithalex/fastapi_googleoauth_jwt_and_redis/stargazers
[issues-shield]: https://img.shields.io/github/issues/programmingwithalex/fastapi_googleoauth_jwt_and_redis?style=for-the-badge
[issues-url]: https://github.com/programmingwithalex/fastapi_googleoauth_jwt_and_redis/issues
[license-shield]: https://img.shields.io/github/license/programmingwithalex/fastapi_googleoauth_jwt_and_redis.svg?style=for-the-badge
[license-url]: https://github.com/programmingwithalex/fastapi_googleoauth_jwt_and_redis/blob/main/LICENSE
