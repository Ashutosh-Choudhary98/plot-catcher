# WalkQuest - System Architecture

## 1. Architectural Overview

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Client Layer (Web/Mobile)                │
│  HTML5/CSS3 | JavaScript | Leaflet.js | OpenStreetMap      │
└────────────────────┬────────────────────────────────────────┘
                     │ HTTPS/REST
┌────────────────────▼────────────────────────────────────────┐
│                  API Gateway Layer                          │
│              Spring Boot REST API (Port 8080)               │
│  ├─ Request Validation                                      │
│  ├─ Rate Limiting                                           │
│  ├─ CORS Management                                         │
│  └─ Request Logging                                         │
└────────────────────┬────────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────────┐
│              Application Layer (Spring Boot)                │
│  ├─ Authentication Service (JWT)                            │
│  ├─ User Service                                            │
│  ├─ Walk Service                                            │
│  ├─ Territory Service (Grid-based)                          │
│  ├─ XP & Leveling Service                                   │
│  ├─ Badge Service                                           │
│  ├─ Leaderboard Service                                     │
│  ├─ Anti-Cheat Service                                      │
│  ├─ Admin Service                                           │
│  └─ Notification Service                                    │
└────────────────────┬────────────────────────────────────────┘
                     │
        ┌────────────┼────────────┬────────────┐
        │            │            │            │
┌───────▼──┐  ┌─────▼────┐  ┌───▼────┐  ┌──▼──────┐
│PostgreSQL│  │  Redis   │  │  S3    │  │ Logging │
│ Database │  │  Cache   │  │Storage │  │ Service │
└──────────┘  └──────────┘  └────────┘  └─────────┘
```

## 2. Layered Architecture

### Presentation Layer
- Mobile-responsive HTML5 frontend
- RESTful API endpoints
- WebSocket support (future multiplayer)
- Request/response validation

### Service Layer
Business logic organized by domain:

**Authentication Service**
- User registration
- Login/logout
- JWT token generation
- Token refresh
- Password hashing (BCrypt)

**User Service**
- User profile management
- User statistics
- User preferences
- Privacy settings

**Walk Service**
- Walk creation
- Real-time location updates
- Walk completion
- Route history
- Distance/duration calculation

**Territory Service**
- Grid-based territory system
- Territory capture logic
- Territory ownership
- Territory statistics

**XP & Leveling Service**
- XP calculation
- Level progression
- Streak tracking
- Bonus multipliers

**Badge Service**
- Badge checking
- Badge awarding
- Badge display
- Achievement unlocking

**Leaderboard Service**
- Leaderboard ranking
- Score calculation
- Rank updates
- Historical tracking

**Anti-Cheat Service**
- GPS validation
- Speed detection
- Route analysis
- Fraud reporting
- User flagging

**Admin Service**
- Analytics computation
- User management
- System monitoring
- Report generation

### Data Access Layer (Spring Data JPA)
- Repository pattern
- Query optimization
- Transaction management
- Batch operations

### Infrastructure Layer
- PostgreSQL database
- Redis cache
- S3 storage (future)
- Logging service

## 3. Database Layer

See [Database Schema](./DATABASE_SCHEMA.md) for complete details.

**Key Tables:**
- users
- walks
- territories (grid-based)
- user_statistics
- xp_history
- badges
- leaderboard_scores
- anti_cheat_reports
- admin_logs

## 4. Caching Strategy

### Redis Cache Layers

**Session Cache:**
- User authentication tokens
- Session data
- TTL: 15 minutes (tokens), 7 days (refresh tokens)

**User Cache:**
- User profile data
- User statistics
- TTL: 5 minutes

**Leaderboard Cache:**
- Computed leaderboard rankings
- TTL: 1 hour (updates every hour)

**Territory Cache:**
- Grid territory ownership
- Territory statistics
- TTL: 10 minutes

**Admin Cache:**
- Analytics metrics
- User counts
- TTL: 30 minutes

### Cache Invalidation Strategy

```
Event                  → Invalidated Caches
─────────────────────────────────────────
Walk Completed        → User, Stats, Leaderboard, Territory
Badge Earned          → User, Profile
Level Up              → User, Leaderboard
Territory Captured    → Territory, Leaderboard
Streak Updated        → User, Stats
```

## 5. Security Architecture

See [Security Design](./SECURITY.md) for complete details.

**Key Components:**
- JWT-based authentication
- BCrypt password hashing
- HTTPS/TLS 1.3
- CORS configuration
- Rate limiting
- Input validation
- SQL injection prevention
- XSS prevention
- CSRF protection

## 6. API Architecture

See [API Design](./API_DESIGN.md) for complete endpoints.

**REST Principles:**
- Stateless design
- Resource-based URLs
- Standard HTTP methods
- Consistent response format
- Error handling

**Example Endpoints:**
```
GET    /api/v1/auth/me
POST   /api/v1/auth/register
POST   /api/v1/auth/login
GET    /api/v1/users/{id}
POST   /api/v1/walks
GET    /api/v1/walks/{id}
POST   /api/v1/walks/{id}/stop
GET    /api/v1/territories
GET    /api/v1/leaderboards/{type}/{timeframe}
GET    /api/v1/profile
GET    /api/v1/admin/analytics
```

## 7. Scalability Design

### Horizontal Scaling

**Stateless Services:**
- Each API instance independent
- No session affinity required
- Load balancer distributes traffic
- Auto-scaling based on CPU/memory

**Database Scaling:**
- Read replicas for analytics queries
- Connection pooling (HikariCP)
- Query optimization
- Indexing strategy

### Caching Layer
- Redis cluster for distributed caching
- Cache warming strategies
- Cache consistency protocols

### Database Partitioning

**Territories Table (Most Critical):**
```
Partition by: Geographic region (lat/long ranges)
Reason: Most queries are location-based
Benefits: Faster queries, easier pruning of old data
```

**Walks Table:**
```
Partition by: Time (monthly partitions)
Reason: Most queries recent walks
Benefits: Faster queries, archive old data
```

## 8. Performance Optimization

### Database Optimization

**Indexes:**
```sql
-- User authentication
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_username ON users(username);

-- Territory lookups
CREATE INDEX idx_territories_user_id ON territories(user_id);
CREATE INDEX idx_territories_grid_hash ON territories(grid_hash);
CREATE INDEX idx_territories_location ON territories USING GIST(location);

-- Walk queries
CREATE INDEX idx_walks_user_id_date ON walks(user_id, created_at DESC);
CREATE INDEX idx_walks_created_at ON walks(created_at DESC);

-- Leaderboard queries
CREATE INDEX idx_leaderboard_scores_timeframe ON leaderboard_scores(timeframe, rank);

-- Anti-cheat
CREATE INDEX idx_anti_cheat_user_date ON anti_cheat_reports(user_id, created_at);
```

### Query Optimization

- Eager loading for related entities
- Pagination for large result sets
- Query result caching
- N+1 query prevention (using fetch joins)
- Batch processing for bulk operations

### API Response Optimization

- Gzip compression
- JSON serialization optimization
- Response pagination
- Field filtering (client can request specific fields)

## 9. Error Handling Architecture

```
Exception Handling Flow:
┌──────────────────────────────────────┐
│ Application Exception Occurs          │
└──────────────┬───────────────────────┘
               │
┌──────────────▼───────────────────────┐
│ Global Exception Handler              │
│ (@RestControllerAdvice)              │
└──────────────┬───────────────────────┘
               │
        ┌──────┴──────────┐
        │                 │
   ┌────▼─────┐      ┌───▼──────┐
   │ Validation│      │ Business │
   │ Error     │      │ Logic    │
   │ 400       │      │ Error    │
   │           │      │ 422      │
   └───────────┘      └──────────┘
               │
┌──────────────▼───────────────────────┐
│ Return Error Response                │
│ - HTTP Status Code                   │
│ - Error Message                      │
│ - Error Code                         │
│ - Timestamp                          │
│ - Trace ID (for logging)             │
└──────────────────────────────────────┘
```

## 10. Monitoring & Observability

**Metrics to Monitor:**
- API response times (p50, p95, p99)
- Database query performance
- Cache hit rates
- Error rates by endpoint
- User activity metrics
- System resource usage

**Logging Strategy:**
- Structured logging (JSON format)
- Log levels: DEBUG, INFO, WARN, ERROR
- Request/response logging with trace IDs
- Anti-cheat fraud logging
- Admin action logging

**Alerting:**
- Error rate spike (>1% failures)
- API latency spike (>500ms p95)
- Database connection pool exhaustion
- Cache miss rate spike
- Anti-cheat suspicious activity surge

## 11. Deployment Architecture

See [Deployment Guide](./DEPLOYMENT.md) for complete details.

**Components:**
- Docker containerization
- Kubernetes orchestration (future)
- CI/CD pipeline
- Staging environment
- Production environment
- Database migrations

## 12. Folder Structure

```
plot-catcher/
├── src/
│   ├── main/
│   │   ├── java/com/walkquest/
│   │   │   ├── config/              # Configuration classes
│   │   │   ├── controller/          # REST controllers
│   │   │   ├── dto/                 # Data transfer objects
│   │   │   ├── entity/              # JPA entities
│   │   │   ├── repository/          # Spring Data repositories
│   │   │   ├── service/             # Business logic
│   │   │   ├── security/            # JWT & security
│   │   │   ├── exception/           # Custom exceptions
│   │   │   ├── util/                # Utility classes
│   │   │   ├── event/               # Domain events
│   │   │   └── WalkQuestApplication.java
│   │   ├── resources/
│   │   │   ├── application.yml      # Configuration
│   │   │   ├── application-dev.yml
│   │   │   ├── application-prod.yml
│   │   │   ├── db/migration/        # Flyway migrations
│   │   │   └── static/              # Frontend assets
│   ├── test/
│   │   ├── java/com/walkquest/
│   │   │   ├── controller/          # Controller tests
│   │   │   ├── service/             # Service tests
│   │   │   ├── repository/          # Repository tests
│   │   │   └── integration/         # Integration tests
│   │   └── resources/
│       └── application-test.yml
├── frontend/
│   ├── index.html
│   ├── css/
│   │   ├── styles.css
│   │   ├── map.css
│   │   ├── profile.css
│   │   └── admin.css
│   ├── js/
│   │   ├── app.js
│   │   ├── api.js
│   │   ├── auth.js
│   │   ├── map.js
│   │   ├── walk.js
│   │   ├── profile.js
│   │   └── admin.js
│   └── pages/
│       ├── login.html
│       ├── register.html
│       ├── map.html
│       ├── profile.html
│       ├── leaderboard.html
│       └── admin.html
├── docs/
│   ├── ARCHITECTURE.md
│   ├── DATABASE_SCHEMA.md
│   ├── API_DESIGN.md
│   ├── SECURITY.md
│   ├── DEPLOYMENT.md
│   └── SETUP.md
├── pom.xml
├── Dockerfile
├── docker-compose.yml
├── .gitignore
└── README.md
```

## 13. Technology Stack Details

**Backend:**
- Java 21 (LTS)
- Spring Boot 3.3.x
- Spring Security 6.x
- Spring Data JPA 3.x
- Apache Maven
- Hibernate ORM
- HikariCP (connection pooling)

**Database:**
- PostgreSQL 15+
- Flyway (schema migrations)
- PostGIS extension (geospatial queries)

**Caching:**
- Redis 7.x
- Lettuce client

**Frontend:**
- HTML5
- CSS3 (Flexbox, Grid)
- JavaScript ES6+
- Leaflet.js (mapping)
- OpenStreetMap (base maps)
- Bootstrap 5 or Tailwind CSS

**DevOps:**
- Docker
- GitHub Actions (CI/CD)
- PostgreSQL Backups

**Testing:**
- JUnit 5
- Mockito
- Spring Boot Test

**Monitoring:**
- SLF4J + Logback
- Micrometer (metrics)

## 14. Data Flow Examples

### Walk Creation Flow

```
1. User clicks "Start Walk"
2. Frontend validates location permission
3. GET /api/v1/walks (create session)
4. Backend creates Walk entity (status=ACTIVE)
5. Frontend tracks GPS in background
6. Every 5 seconds: POST /api/v1/walks/{id}/location
7. Backend updates walk location, checks grid
8. If new grid: Create Territory, +20 XP
9. Cache invalidation for user stats
10. When walk stops: POST /api/v1/walks/{id}/stop
11. Backend calculates final metrics
12. Updates user XP, levels, streaks
13. Leaderboard recalculation (async)
14. Response with walk summary
```

### Leaderboard Update Flow

```
1. Walk completes
2. Event: WalkCompletedEvent emitted
3. LeaderboardService listens
4. Calculates user's new scores
5. Updates leaderboard_scores for all timeframes
6. Invalidates leaderboard cache
7. Checks for rank changes
8. Sends notifications if needed
9. Updates champion titles if applicable
```

### Anti-Cheat Detection Flow

```
1. Location update received
2. AntiCheatService validates
3. Checks GPS accuracy > 50m? NO → Flag as unreliable
4. Checks speed > 25 km/h? YES → Flag as non-walking
5. Checks impossible teleportation? YES → Flag as fraud
6. Checks repeated route? YES → Reduce rewards
7. If fraud flags exceed threshold: Create report
8. Store in anti_cheat_reports table
9. Notify admin dashboard
10. Pending manual review
```
