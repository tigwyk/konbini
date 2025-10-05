# WARP.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

## Overview

Konbini is a partially indexed Bluesky AppView that provides a "Friends of Friends" experience. It consists of a Go backend that connects to the Bluesky firehose and a React TypeScript frontend. The application indexes posts, follows, likes, and other social interactions in a PostgreSQL database.

## Architecture

### Backend (Go)
- **Main server**: `main.go` - Entry point with CLI setup, database initialization, and firehose connection
- **API handlers**: `handlers.go` - REST API endpoints for the frontend (port 4444)
- **Event processing**: `events.go` - Handles real-time firehose events from Bluesky
- **Database layer**: `pgbackend.go` - PostgreSQL operations and caching layer
- **Models**: `models.go` - Database schema definitions (mostly aliases from whyrusleeping/market)
- **Missing data handling**: `missing.go` - Fetches missing profiles and posts asynchronously

### Frontend (React TypeScript)
- **Components**: Located in `frontend/src/components/`
  - `FollowingFeed.tsx` - Main feed view
  - `PostCard.tsx` - Individual post display
  - `ProfilePage.tsx` - User profile view
  - `ThreadView.tsx` - Thread/conversation view
  - `PostComposer.tsx` - Creating new posts
  - `NotificationsPage.tsx` - User notifications
- **API layer**: `frontend/src/api.ts` - HTTP client for backend communication
- **Types**: `frontend/src/types.ts` - TypeScript type definitions

### Database
- Uses PostgreSQL with GORM for ORM
- Caches with LRU caches for repos, posts, and reverse lookups
- Models include: Repo, Post, Follow, Block, Like, Repost, Profile, Notification, etc.

## Development Commands

### Full Stack Development (Docker Compose - Recommended)
```bash
# Start all services (PostgreSQL, backend, frontend)
docker-compose up -d

# View logs
docker-compose logs -f

# Stop all services
docker-compose down

# Stop and remove database volume
docker-compose down -v

# View logs for specific service
docker-compose logs -f backend
docker-compose logs -f frontend
docker-compose logs -f postgres

# Rebuild and restart services
docker-compose up -d --build

# Access backend container shell
docker-compose exec backend sh

# Access database directly
docker-compose exec postgres psql -U konbini -d konbini
```

### Backend Development
```bash
# Build the Go application
go build

# Run with environment variables (requires DATABASE_URL, BSKY_HANDLE, BSKY_PASSWORD)
./konbini

# Run with custom database URL
./konbini --db-url="postgresql://user:pass@localhost:5432/konbini"

# Run tests (currently no Go tests exist)
go test ./...

# Get dependencies
go mod download

# Format Go code
go fmt ./...

# Vet Go code for common issues
go vet ./...

# Run specific Go file for quick testing
go run main.go handlers.go events.go models.go pgbackend.go missing.go seqno.go
```

### Frontend Development
```bash
# Install dependencies
cd frontend && npm install

# Start development server (connects to backend on port 4444)
npm start

# Build for production
npm run build

# Run tests (currently only App.test.tsx exists)
npm test

# Run tests in watch mode
npm test --watchAll

# Run specific test file
npm test -- --testPathPattern=App.test.tsx

# Run tests with coverage
npm test -- --coverage --watchAll=false

# Eject from Create React App (irreversible)
npm run eject
```

### Database Setup
```bash
# Start PostgreSQL with Docker
docker run --name konbini-postgres \
  -e POSTGRES_DB=konbini \
  -e POSTGRES_USER=konbini \
  -e POSTGRES_PASSWORD=your_password \
  -p 5432:5432 \
  -d postgres:15
```

## Environment Setup

Create a `.env` file from `.env.example`:
```bash
cp .env.example .env
```

Required environment variables:
- `DATABASE_URL`: PostgreSQL connection string (optional, defaults to Docker Compose setup)
- `BSKY_HANDLE`: Your Bluesky handle (e.g., user.bsky.social) - **Required**
- `BSKY_PASSWORD`: Your Bluesky app password (get from https://bsky.app/settings/app-passwords) - **Required**

Note: The application creates a `sequence.txt` file to track firehose position for resuming after restarts.

## Key Ports
- Backend API: 4444
- Frontend dev server: 3000
- pprof debug: 4445
- PostgreSQL: 5432

## Development Notes

### Backend Architecture Patterns
- The application uses a firehose consumer pattern to listen to real-time Bluesky events
- Implements a "relevant DIDs" system to only index posts from followed users and their networks
- Uses multiple caches (LRU) for performance: repo cache, post info cache, reverse cache
- Asynchronous missing data fetching via channels to handle incomplete data from the firehose

### API Structure
All API endpoints are prefixed with `/api` and include:
- `/api/me` - Current user info
- `/api/followingfeed` - Main social feed
- `/api/profile/:account` - User profile data
- `/api/profile/:account/posts` - User's posts with pagination
- `/api/thread/:postid` - Thread view
- `/api/notifications` - User notifications

### Database Migrations
Migrations happen automatically on startup via GORM's AutoMigrate feature in `main.go`.

### Testing
- Frontend uses React Testing Library with Create React App's built-in Jest setup
- Current frontend test: `frontend/src/App.test.tsx` - basic render test
- No Go tests currently exist - test files would follow `*_test.go` naming convention
- Go version requirement: 1.25.1+ (as specified in go.mod)

### Performance Considerations
- The application maintains several in-memory caches to reduce database load
- Uses connection pooling for PostgreSQL
- Implements cursor-based pagination for feeds
- Prometheus metrics available on port 4445