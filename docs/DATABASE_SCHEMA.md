# WalkQuest - Database Schema

## Database Overview

**Database:** PostgreSQL 15+
**Connection Pool:** HikariCP (20-30 connections)
**Migrations:** Flyway
**Geospatial Support:** PostGIS extension

## Entity Relationship Diagram

```
users
├── walks (1:M)
├── territories (1:M)
├── user_statistics (1:1)
├── badges_earned (1:M)
├── leaderboard_scores (1:M)
├── anti_cheat_reports (1:M)
└── admin_logs (1:M)

walks
├── walk_locations (1:M)
└── walk_events (1:M)

territories
├── territory_history (1:M)
└── territory_statistics (1:M)

leaderboards (static config)
└── leaderboard_scores (1:M)

badges (static config)
└── badges_earned (1:M)
```

## Core Tables

### 1. users

**Purpose:** User account information

```sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    username VARCHAR(100) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    profile_photo_url TEXT,
    bio TEXT,
    
    -- Status
    status VARCHAR(20) DEFAULT 'ACTIVE' CHECK (status IN ('ACTIVE', 'SUSPENDED', 'BANNED')),
    
    -- Privacy settings
    location_sharing_enabled BOOLEAN DEFAULT FALSE,
    location_sharing_mode VARCHAR(20) DEFAULT 'INVISIBLE' CHECK (location_sharing_mode IN ('INVISIBLE', 'FRIENDS_ONLY', 'PUBLIC')),
    
    -- Timestamps
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    last_login_at TIMESTAMP,
    
    -- Indexing
    INDEX idx_users_email (email),
    INDEX idx_users_username (username),
    INDEX idx_users_status (status)
);
```

### 2. user_statistics

**Purpose:** Denormalized user aggregate data for fast retrieval

```sql
CREATE TABLE user_statistics (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL UNIQUE REFERENCES users(id) ON DELETE CASCADE,
    
    -- XP & Levels
    current_xp BIGINT DEFAULT 0,
    total_xp BIGINT DEFAULT 0,
    current_level INT DEFAULT 1,
    
    -- Distance
    total_distance_km DECIMAL(10, 2) DEFAULT 0,
    total_walking_time_minutes INT DEFAULT 0,
    average_speed_kmh DECIMAL(5, 2) DEFAULT 0,
    
    -- Territories
    total_territories_claimed INT DEFAULT 0,
    unique_territories INT DEFAULT 0,
    
    -- Streaks
    current_streak INT DEFAULT 0,
    longest_streak INT DEFAULT 0,
    last_walk_date DATE,
    
    -- Badges
    total_badges_earned INT DEFAULT 0,
    
    -- Rankings
    area_rank INT,
    city_rank INT,
    state_rank INT,
    national_rank INT,
    
    -- Timestamps
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_user_statistics_user_id (user_id),
    INDEX idx_user_statistics_total_xp (total_xp DESC),
    INDEX idx_user_statistics_total_distance (total_distance_km DESC)
);
```

### 3. walks

**Purpose:** Individual walk records

```sql
CREATE TABLE walks (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    
    -- Status
    status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE' CHECK (status IN ('ACTIVE', 'COMPLETED', 'PAUSED', 'INVALID')),
    
    -- Metrics
    distance_km DECIMAL(8, 2) DEFAULT 0,
    duration_minutes INT DEFAULT 0,
    
    -- Speed
    average_speed_kmh DECIMAL(5, 2),
    max_speed_kmh DECIMAL(5, 2),
    
    -- Timestamps
    started_at TIMESTAMP NOT NULL,
    ended_at TIMESTAMP,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    -- Additional data
    route_summary TEXT,
    device_type VARCHAR(50),
    gps_accuracy_meters DECIMAL(6, 2),
    
    -- XP earned in this walk
    xp_earned INT DEFAULT 0,
    distance_xp INT DEFAULT 0,
    territory_xp INT DEFAULT 0,
    bonus_xp INT DEFAULT 0,
    
    -- Validation flags
    is_valid BOOLEAN DEFAULT TRUE,
    validation_notes TEXT,
    
    INDEX idx_walks_user_id (user_id),
    INDEX idx_walks_user_date (user_id, started_at DESC),
    INDEX idx_walks_status (status),
    INDEX idx_walks_created_at (created_at DESC),
    INDEX idx_walks_is_valid (is_valid)
);
```

### 4. walk_locations

**Purpose:** GPS track points for walks (high volume table)

```sql
CREATE TABLE walk_locations (
    id BIGSERIAL PRIMARY KEY,
    walk_id BIGINT NOT NULL REFERENCES walks(id) ON DELETE CASCADE,
    user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    
    -- Geospatial data
    latitude DECIMAL(10, 8) NOT NULL,
    longitude DECIMAL(11, 8) NOT NULL,
    location GEOMETRY(Point, 4326) NOT NULL,
    
    -- Accuracy
    accuracy_meters DECIMAL(6, 2),
    altitude_meters DECIMAL(8, 2),
    
    -- Speed at this point
    speed_kmh DECIMAL(5, 2),
    bearing_degrees DECIMAL(6, 2),
    
    -- Timestamp
    timestamp_at TIMESTAMP NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_walk_locations_walk_id (walk_id),
    INDEX idx_walk_locations_user_id (user_id),
    INDEX idx_walk_locations_location (location USING GIST),
    INDEX idx_walk_locations_timestamp (timestamp_at DESC)
);

-- Note: walk_locations table can grow very large. Consider:
-- - Archiving old data monthly
-- - Using time-series database (TimescaleDB) for optimization
-- - Partitioning by date
```

### 5. territories

**Purpose:** Grid-based territory ownership (most critical table for scaling)

```sql
CREATE TABLE territories (
    id BIGSERIAL PRIMARY KEY,
    
    -- Grid identification
    grid_hash VARCHAR(32) NOT NULL UNIQUE,  -- Hash of lat/lon rounded to 50m
    grid_x INT NOT NULL,                     -- Grid X coordinate
    grid_y INT NOT NULL,                     -- Grid Y coordinate
    
    -- Geospatial
    center_latitude DECIMAL(10, 8) NOT NULL,
    center_longitude DECIMAL(11, 8) NOT NULL,
    center_location GEOMETRY(Point, 4326) NOT NULL,
    bounds GEOMETRY(Polygon, 4326) NOT NULL,
    
    -- Ownership
    claimed_by_user_id BIGINT REFERENCES users(id) ON DELETE SET NULL,
    claimed_at TIMESTAMP,
    
    -- Area/City/State/Country
    area_id INT,
    city_id INT,
    state_id INT,
    country_id INT,
    
    -- Statistics
    times_claimed INT DEFAULT 1,
    last_disputed_at TIMESTAMP,
    
    -- Timestamps
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_territories_grid_hash (grid_hash),
    INDEX idx_territories_user_id (claimed_by_user_id),
    INDEX idx_territories_location (center_location USING GIST),
    INDEX idx_territories_area_id (area_id),
    INDEX idx_territories_city_id (city_id),
    INDEX idx_territories_bounds (bounds USING GIST),
    INDEX idx_territories_created_at (created_at DESC)
);

-- PARTITIONING STRATEGY for territories:
-- Partition by geographic region (lat/lon ranges)
-- Reduces query time for location-based queries
```

### 6. walk_events

**Purpose:** Significant events during a walk (new territory, level up, etc.)

```sql
CREATE TABLE walk_events (
    id BIGSERIAL PRIMARY KEY,
    walk_id BIGINT NOT NULL REFERENCES walks(id) ON DELETE CASCADE,
    user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    
    -- Event type
    event_type VARCHAR(50) NOT NULL CHECK (event_type IN (
        'TERRITORY_CAPTURED',
        'LEVEL_UP',
        'BADGE_EARNED',
        'STREAK_MILESTONE',
        'NEW_RANK'
    )),
    
    -- Event data
    event_data JSONB NOT NULL,
    
    -- Timestamps
    occurred_at TIMESTAMP NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_walk_events_walk_id (walk_id),
    INDEX idx_walk_events_user_id (user_id),
    INDEX idx_walk_events_type (event_type)
);
```

### 7. xp_history

**Purpose:** Track XP changes for auditing and replay

```sql
CREATE TABLE xp_history (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    walk_id BIGINT REFERENCES walks(id) ON DELETE SET NULL,
    
    -- XP details
    xp_amount INT NOT NULL,
    xp_type VARCHAR(50) NOT NULL CHECK (xp_type IN (
        'DISTANCE',
        'TERRITORY',
        'BADGE',
        'STREAK_BONUS',
        'ACHIEVEMENT',
        'CHALLENGE'
    )),
    reason TEXT,
    
    -- Running total
    previous_xp BIGINT NOT NULL,
    new_xp BIGINT NOT NULL,
    level_before INT,
    level_after INT,
    
    -- Timestamps
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_xp_history_user_id (user_id),
    INDEX idx_xp_history_walk_id (walk_id),
    INDEX idx_xp_history_created_at (created_at DESC)
);
```

### 8. badges

**Purpose:** Badge configuration (static data)

```sql
CREATE TABLE badges (
    id BIGSERIAL PRIMARY KEY,
    
    -- Identification
    code VARCHAR(100) UNIQUE NOT NULL,
    name VARCHAR(100) NOT NULL,
    description TEXT,
    icon_url TEXT,
    
    -- Badge type
    badge_type VARCHAR(50) NOT NULL CHECK (badge_type IN (
        'DISTANCE',
        'TERRITORY',
        'STREAK',
        'SPECIAL'
    )),
    
    -- Achievement criteria
    criteria_type VARCHAR(50) NOT NULL CHECK (criteria_type IN (
        'DISTANCE_THRESHOLD',
        'TERRITORY_COUNT',
        'STREAK_LENGTH',
        'MANUAL'
    )),
    criteria_value INT,
    
    -- Display
    rarity VARCHAR(20) CHECK (rarity IN ('COMMON', 'RARE', 'EPIC', 'LEGENDARY')),
    color_hex VARCHAR(7),
    
    -- Timestamps
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_badges_code (code),
    INDEX idx_badges_badge_type (badge_type)
);
```

### 9. badges_earned

**Purpose:** Track user badge achievements

```sql
CREATE TABLE badges_earned (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    badge_id BIGINT NOT NULL REFERENCES badges(id) ON DELETE CASCADE,
    walk_id BIGINT REFERENCES walks(id) ON DELETE SET NULL,
    
    -- Achievement
    earned_at TIMESTAMP NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    UNIQUE(user_id, badge_id),
    
    INDEX idx_badges_earned_user_id (user_id),
    INDEX idx_badges_earned_badge_id (badge_id),
    INDEX idx_badges_earned_earned_at (earned_at DESC)
);
```

### 10. leaderboards

**Purpose:** Leaderboard configuration (static data)

```sql
CREATE TABLE leaderboards (
    id BIGSERIAL PRIMARY KEY,
    
    -- Identification
    leaderboard_type VARCHAR(50) NOT NULL CHECK (leaderboard_type IN (
        'AREA',
        'CITY',
        'STATE',
        'NATIONAL'
    )),
    
    leaderboard_timeframe VARCHAR(50) NOT NULL CHECK (leaderboard_timeframe IN (
        'DAILY',
        'WEEKLY',
        'MONTHLY',
        'ALL_TIME'
    )),
    
    -- Geographic scope (if applicable)
    area_id INT,
    city_id INT,
    state_id INT,
    
    -- Ranking metric
    ranking_metric VARCHAR(50) NOT NULL CHECK (ranking_metric IN (
        'DISTANCE',
        'XP',
        'TERRITORIES',
        'COMPOSITE'
    )),
    
    -- Timing
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    UNIQUE(leaderboard_type, leaderboard_timeframe, area_id, city_id, state_id),
    
    INDEX idx_leaderboards_type (leaderboard_type),
    INDEX idx_leaderboards_timeframe (leaderboard_timeframe)
);
```

### 11. leaderboard_scores

**Purpose:** User scores on leaderboards (updated hourly)

```sql
CREATE TABLE leaderboard_scores (
    id BIGSERIAL PRIMARY KEY,
    leaderboard_id BIGINT NOT NULL REFERENCES leaderboards(id) ON DELETE CASCADE,
    user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    
    -- Score
    score BIGINT NOT NULL,
    rank INT NOT NULL,
    
    -- Historical tracking
    previous_rank INT,
    previous_score BIGINT,
    
    -- Timestamps
    last_updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    UNIQUE(leaderboard_id, user_id),
    
    INDEX idx_leaderboard_scores_leaderboard (leaderboard_id),
    INDEX idx_leaderboard_scores_user (user_id),
    INDEX idx_leaderboard_scores_rank (leaderboard_id, rank)
);
```

### 12. anti_cheat_reports

**Purpose:** Track suspicious activity for admin review

```sql
CREATE TABLE anti_cheat_reports (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    walk_id BIGINT REFERENCES walks(id) ON DELETE SET NULL,
    
    -- Report type
    report_type VARCHAR(50) NOT NULL CHECK (report_type IN (
        'IMPOSSIBLE_SPEED',
        'TELEPORTATION',
        'GPS_ANOMALY',
        'REPEATED_FARMING',
        'SUSPICIOUS_PATTERN',
        'MANUAL_REPORT'
    )),
    
    -- Details
    description TEXT NOT NULL,
    evidence JSONB,
    
    -- Status
    status VARCHAR(50) DEFAULT 'PENDING' CHECK (status IN ('PENDING', 'REVIEWED', 'FALSE_POSITIVE', 'ACTION_TAKEN')),
    severity VARCHAR(20) CHECK (severity IN ('LOW', 'MEDIUM', 'HIGH', 'CRITICAL')),
    
    -- Admin action
    reviewed_by_admin_id BIGINT REFERENCES users(id) ON DELETE SET NULL,
    admin_notes TEXT,
    action_taken VARCHAR(100),
    
    -- Timestamps
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    reviewed_at TIMESTAMP,
    
    INDEX idx_anti_cheat_user_id (user_id),
    INDEX idx_anti_cheat_walk_id (walk_id),
    INDEX idx_anti_cheat_status (status),
    INDEX idx_anti_cheat_severity (severity),
    INDEX idx_anti_cheat_created_at (created_at DESC)
);
```

### 13. admin_logs

**Purpose:** Audit trail for admin actions

```sql
CREATE TABLE admin_logs (
    id BIGSERIAL PRIMARY KEY,
    admin_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    
    -- Action
    action_type VARCHAR(100) NOT NULL,
    entity_type VARCHAR(50),
    entity_id BIGINT,
    
    -- Details
    changes JSONB,
    reason TEXT,
    
    -- Result
    status VARCHAR(50) DEFAULT 'SUCCESS' CHECK (status IN ('SUCCESS', 'FAILED')),
    error_message TEXT,
    
    -- Timestamps
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_admin_logs_admin_id (admin_id),
    INDEX idx_admin_logs_action_type (action_type),
    INDEX idx_admin_logs_created_at (created_at DESC)
);
```

## Geographic Reference Tables

### 14. areas

```sql
CREATE TABLE areas (
    id INT PRIMARY KEY AUTO_INCREMENT,
    area_name VARCHAR(255) NOT NULL,
    city_id INT REFERENCES cities(id),
    bounds GEOMETRY(Polygon, 4326),
    center_location GEOMETRY(Point, 4326),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 15. cities

```sql
CREATE TABLE cities (
    id INT PRIMARY KEY AUTO_INCREMENT,
    city_name VARCHAR(255) NOT NULL UNIQUE,
    state_id INT REFERENCES states(id),
    center_location GEOMETRY(Point, 4326),
    bounds GEOMETRY(Polygon, 4326),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 16. states

```sql
CREATE TABLE states (
    id INT PRIMARY KEY AUTO_INCREMENT,
    state_name VARCHAR(255) NOT NULL UNIQUE,
    state_code VARCHAR(10),
    country_id INT REFERENCES countries(id),
    center_location GEOMETRY(Point, 4326),
    bounds GEOMETRY(Polygon, 4326),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 17. countries

```sql
CREATE TABLE countries (
    id INT PRIMARY KEY AUTO_INCREMENT,
    country_name VARCHAR(255) NOT NULL UNIQUE,
    country_code VARCHAR(10),
    center_location GEOMETRY(Point, 4326),
    bounds GEOMETRY(Polygon, 4326),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

## Performance Optimization Indexes

```sql
-- User queries
CREATE INDEX idx_users_email_active ON users(email) WHERE status = 'ACTIVE';
CREATE INDEX idx_users_created_at ON users(created_at DESC);

-- Walk queries
CREATE INDEX idx_walks_status_user ON walks(user_id, status);
CREATE INDEX idx_walks_date_range ON walks(started_at DESC) WHERE status = 'COMPLETED';

-- Territory queries (CRITICAL)
CREATE INDEX idx_territories_city_user ON territories(city_id, claimed_by_user_id);
CREATE INDEX idx_territories_state_user ON territories(state_id, claimed_by_user_id);
CREATE INDEX idx_territories_unclaimed ON territories(claimed_by_user_id) WHERE claimed_by_user_id IS NULL;

-- Leaderboard queries
CREATE INDEX idx_leaderboard_scores_timeframe_rank ON leaderboard_scores(leaderboard_id, rank);

-- Badge queries
CREATE INDEX idx_badges_earned_recent ON badges_earned(user_id, earned_at DESC);

-- Anti-cheat queries
CREATE INDEX idx_anti_cheat_user_recent ON anti_cheat_reports(user_id, created_at DESC);
```

## Data Volume Estimates (at Scale)

```
Users:                    100,000 - 1,000,000
Walks:                    50,000,000 - 500,000,000
Walk Locations:           5,000,000,000 - 50,000,000,000 (very large - candidate for archival/time-series)
Territories:             1,000,000 - 10,000,000
Badges Earned:           500,000,000 - 5,000,000,000
Leaderboard Scores:      1,000,000 - 50,000,000 (multiple leaderboards)
Anti-Cheat Reports:      10,000 - 1,000,000
```

## Database Maintenance

**Partitioning Strategy:**

1. **walk_locations:** Monthly partitions by created_at
   - Old data (>6 months) archived to cold storage
   - Current month in hot storage

2. **territories:** By geographic region
   - Reduces lock contention
   - Faster geographic queries

3. **walks:** By created_at (quarterly)
   - Old data kept for analytics
   - Current quarter in hot storage

**Backup Strategy:**
- Daily full backups
- Hourly incremental backups
- WAL archival
- Replicated to secondary region

**Vacuum & Analyze:**
- Weekly VACUUM ANALYZE
- Monthly REINDEX on high-churn tables

**Monitoring:**
- Query performance (slow query log)
- Index fragmentation
- Table bloat
- Connection pool usage
