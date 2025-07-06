# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

WuzAPI is a RESTful API service built in Go that acts as a bridge to the whatsmeow library, enabling programmatic interaction with WhatsApp. It supports multi-device connections and concurrent sessions, communicating directly with WhatsApp's WebSocket servers.

## Essential Commands

### Dependencies
```bash
# Update WhatsApp library to latest version
go get -u go.mau.fi/whatsmeow@latest
go mod tidy
```

### Build and Run
```bash
# Build the application
go build .

# Run with default settings (port 8080)
./wuzapi

# Run with custom configuration
./wuzapi -port 8080 -logtype console -color true -admintoken YOUR_TOKEN

# Common flags:
# -admintoken: Admin authentication token (or use WUZAPI_ADMIN_TOKEN in .env)
# -port: Server port (default 8080)
# -logtype: console or json
# -color: Enable colored console output
# -wadebug: Enable whatsmeow debug (INFO or DEBUG)
```

### Environment Configuration

The application uses a `.env` file. Example for PostgreSQL:
```
WUZAPI_ADMIN_TOKEN=your_admin_token_here
DB_USER=wuzapi
DB_PASSWORD=wuzapi
DB_NAME=wuzapi
DB_HOST=localhost
DB_PORT=5432
```

For SQLite, use:
```
WUZAPI_ADMIN_TOKEN=your_admin_token_here
WUZAPI_DATABASE=sqlite
WUZAPI_DATABASE_URL=wuzapi.db
```

## Architecture

### Core Components

- **main.go**: Entry point, initializes database and HTTP server
- **routes.go**: API endpoint definitions and authentication middleware
- **handlers.go**: Request handling logic for all API operations
- **clients.go**: WhatsApp client lifecycle management (connect/disconnect/events)
- **db.go**: Database abstraction layer (PostgreSQL/SQLite)
- **migrations.go**: Database schema migrations
- **rabbitmq.go**: Optional RabbitMQ integration for event distribution
- **s3manager.go**: Optional S3 integration for media storage

### API Structure

All endpoints require `Authorization` header with user token, except admin endpoints which use admin token.

Key endpoint categories:
- **/admin/**: User management (create/list/delete users)
- **/api**: Swagger UI documentation
- **Session**: /connect, /disconnect, /status, /qr
- **Messages**: /send/text, /send/image, /send/audio, etc.
- **Users**: /users/info, /users/avatar, /users/contacts
- **Groups**: /groups/create, /groups/info, /groups/participants
- **Webhooks**: /webhooks/set, /webhooks/get

### Database Schema

The application uses either PostgreSQL (production) or SQLite (development). Key tables:
- Users: Stores user accounts with tokens and webhook URLs
- Sessions: WhatsApp session data
- Media: Optional media file references when using S3

### External Dependencies

- **whatsmeow**: Core WhatsApp protocol implementation
- **gorilla/mux**: HTTP routing
- **zerolog**: Structured logging
- **PostgreSQL/SQLite**: Database backends
- **RabbitMQ** (optional): Event distribution
- **AWS S3** (optional): Media storage

## Development Guidelines

### Code Conventions

- Use standard Go error handling patterns
- All API responses should be JSON formatted
- Authentication via Authorization header
- Middleware in routes.go handles authentication
- Database operations abstracted in db.go

### API Testing

Use the included Postman collection (`wuzapi_postman.json`) for testing endpoints. Basic workflow:
1. Create user via admin endpoint
2. Connect using user token
3. Scan QR code to authenticate
4. Send messages and interact with WhatsApp

### Adding New Features

1. Add route in `routes.go`
2. Implement handler in `handlers.go`
3. Add database operations in `db.go` if needed
4. Update API documentation in `API.md`
5. Test with Postman or curl

### Important Notes

- No automated tests currently exist in the codebase
- The application connects directly to WhatsApp WebSocket servers
- Changes to WhatsApp protocol may require library updates
- Use caution to avoid WhatsApp Terms of Service violations