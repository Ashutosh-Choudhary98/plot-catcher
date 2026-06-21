# WalkQuest Product Specification

## Vision

WalkQuest is a mobile-first platform that turns real-world walking into a territory conquest game. Users compete, explore, and progress through a gamified system that rewards genuine walking activity with XP, badges, streaks, and territorial ownership.

## Target Users

**Primary:**
- College students (18-25)
- Gen-Z users
- Young professionals
- Casual fitness enthusiasts

**Secondary:**
- Walking communities
- Running groups
- Local fitness clubs

## User Roles

### ROLE_USER

**Capabilities:**
- Register and authenticate
- Start/stop walks
- Capture territory
- Earn XP and badges
- View rankings
- View personalized map
- View profile and achievements
- View activity calendar
- Compete in leaderboards

### ROLE_ADMIN

**Capabilities:**
- View analytics and metrics
- View active/online users
- View growth and territory statistics
- Manage users (view, edit, suspend, ban)
- Manage badges and achievements
- Review suspicious activity
- Monitor anti-cheat reports

## Core Systems

### 1. Authentication System

- User registration with email verification
- Secure login with JWT tokens
- Refresh token mechanism
- BCrypt password hashing
- Session management
- Logout with token invalidation

### 2. Walking System

**Walk Tracking:**
- Start walk (with location validation)
- Stop walk (calculate final metrics)
- Real-time location updates
- Distance calculation
- Duration tracking
- Route history

**Metrics Stored:**
- Distance (km)
- Duration (minutes)
- Start/end time
- Date
- Route summary
- Average speed
- Max speed
- Calories (estimated)

### 3. Territory System

**Grid-Based Mechanics:**
- Map divided into 50m x 50m grids
- Three grid states: Unclaimed, Claimed by User, Claimed by Other
- Capture on first walk through new grid
- Natural support for non-geometric routes
- Handles real-world walking patterns (A→B, A→B→A, A→B→C→B→A)

**Territory Ownership:**
- User owns grid after first capture
- Territory contributes to XP
- Territory counts in leaderboard rankings
- Visual feedback on map (color-coded)

**Anti-Gaming Measures:**
- GPS accuracy validation
- Impossible speed detection
- Repeated route farming detection
- Diminishing rewards for same routes
- Walking vs. vehicle detection

### 4. XP & Leveling System

**XP Rewards:**
- Distance: 10 XP per km
- Territory: 20 XP per new grid
- Achievements: Up to 100 XP per challenge
- Streak bonus: +5 XP multiplier at 7+ day streak

**Leveling:**
- Level 1: 0 XP
- Level 2: 100 XP
- Level 3: 250 XP
- Level N: Exponential progression
- Maximum XP cap: None (infinite progression)

**Tracking:**
- Current XP
- Total XP (all-time)
- Current level
- XP to next level

### 5. Badge System

**Distance Badges:**
- Bronze Scout: 10 km
- Silver Explorer: 50 km
- Gold Pathfinder: 100 km
- Platinum Nomad: 500 km
- Diamond Voyager: 1000 km

**Territory Badges:**
- Territory Rookie: 1 territory
- Territory Collector: 10 territories
- Territory Master: 50 territories
- Territory Conqueror: 100 territories
- Territory King: 500 territories

**Streak Badges:**
- Streak Starter: 3-day streak
- Streak Warrior: 7-day streak
- Streak Legend: 30-day streak
- Consistency Champion: 100-day streak

**Special Badges:**
- First Steps (first walk)
- Night Owl (walk after 9 PM)
- Early Bird (walk before 6 AM)
- Marathon Walker (single walk > 20 km)
- Speed Demon (average speed > 10 km/h)

**Badge Properties:**
- Permanent once earned
- Display on profile
- Contribute to profile prestige
- May unlock special features

### 6. Streak System

**Mechanics:**
- Continues if user walks >= 1 day
- Resets if no walk in 24 hours
- Stored: Current streak, Longest streak
- Critical for retention

**Streak Bonuses:**
- 3-day: +5% XP multiplier
- 7-day: +10% XP multiplier
- 30-day: +20% XP multiplier
- 100-day: +50% XP multiplier

### 7. Activity Calendar (LeetCode Style)

**Contribution Heatmap:**
- Display entire year
- Show daily activity
- Color intensity based on distance

**Color Scale:**
- Gray: No walk
- Light Green: 1-2 km
- Medium Green: 3-5 km
- Dark Green: 5-10 km
- Darkest Green: 10+ km

**Display Information:**
- Current streak
- Longest streak
- Total walks this year
- Total distance this year

### 8. Leaderboard System

**Leaderboard Types:**
- Area (500m radius)
- City
- State
- India

**Timeframes:**
- Daily (reset 12 AM IST)
- Weekly (Sun-Sat)
- Monthly (1st-last day)
- All Time

**Ranking Metrics:**
- Distance walked
- XP earned
- Territories captured
- Composite score

**Anti-Dominance Features:**
- Handicap system for new users
- Recent activity weighted higher
- Multiple metrics to prevent single-metric dominance
- Rolling windows to keep leaderboards fresh

### 9. Champion Titles

**Auto-Awarded Titles:**
- Area Champion (Rank #1 in Area)
- City King (Rank #1 in City)
- State Maharaja (Rank #1 in State)
- Walker of India (Rank #1 nationally)

**Title Benefits:**
- Display on profile
- Special badge color
- Contribute to profile prestige
- Historical record maintained

### 10. User Profile

**Display Information:**
- Profile photo
- Username (unique)
- Current level
- Current XP / XP to next level
- Global rank
- Total distance (all-time)
- Total territories
- Current streak
- Longest streak
- All earned badges (with unlock dates)
- Activity calendar
- Achievement list
- Current champion titles
- Recent walks (last 10)
- Friends (future)

### 11. Map Screen

**Most Important Screen**

**Requirements:**
- Mobile-first responsive design
- Full-screen map
- User's current location marker
- Claimed territories (user's color)
- Claimed territories (others' color)
- Unclaimed territories (neutral color)
- Active walks in progress (red line)
- Territory stats overlay
- Quick start walk button
- Performance optimized

**Visual Feedback:**
- Smooth animations
- Color-coded territories
- XP pop-ups on new territory
- Sound effects (optional)
- Haptic feedback on capture (mobile)

### 12. Privacy & Security

**Privacy-First Approach:**
- Location never exposed publicly by default
- Location sharing OFF by default
- Future options: Friends Only, Public, Invisible
- GDPR compliant data handling
- User can delete all location history

**Location Data:**
- Server-side encryption at rest
- HTTPS in transit
- Never shared without explicit consent
- Aggregated only for leaderboards

### 13. Anti-Cheat System

**Detection Mechanisms:**

1. **GPS Validation:**
   - GPS accuracy check (>50m accuracy required)
   - Sudden teleportation detection
   - GPS drift tolerance (±50m)

2. **Speed Detection:**
   - Max walking speed: 20 km/h
   - Max running speed: 25 km/h
   - Impossible speed: >60 km/h = Flagged

3. **Route Analysis:**
   - Repeated identical routes (diminishing rewards)
   - Straight-line impossible routes
   - Historical pattern analysis

4. **Behavioral Analysis:**
   - Abnormal XP patterns
   - Territory farming (same area repetition)
   - Unnatural activity spikes

**Response Actions:**
- Flag suspicious activity
- Store in fraud database
- Send to admin dashboard
- Temporary activity hold (manual review)
- Potential account suspension/ban

### 14. Vehicle Detection

**Intent:**
- Ignore territory capture during non-walking activities
- Only reward genuine walking

**Detection Methods:**
- Speed threshold analysis
- GPS accuracy patterns
- User historical data
- Machine learning (future)

**Implementation:**
- If speed > 25 km/h, flag as non-walking
- Skip territory capture
- Still count distance for stats (marked as "ride")
- XP penalty for non-walking

### 15. Notification System

**Meaningful Notifications (Avoid Spam):**

- Walk start confirmation
- Walk stop with summary
- New level achieved
- New badge earned
- Streak milestone (7, 30, 100 days)
- Leaderboard rank change
- New champion title
- Friend achievement (future)

**Settings:**
- Notification preferences
- Quiet hours (e.g., 10 PM - 8 AM)
- Granular control per notification type

### 16. Admin Dashboard

**Analytics Section:**
- Total users
- Active users (today/this week/this month)
- Online users (right now)
- New registrations
- User retention metrics
- Churn rate

**Activity Section:**
- Total distance walked
- Distance walked today
- Average walk duration
- Total territories captured
- Top walking areas
- Peak activity times

**User Management:**
- Search users
- View user details
- View user walks history
- View user territories
- Suspend user account
- Ban user account
- Reset user statistics (admin action)

**Territory Management:**
- Total territories claimed
- Unclaimed territories
- Most claimed areas
- Territory statistics by region
- Territory heatmap

**Fraud Management:**
- Suspicious activity reports
- Flagged users list
- Impossible speed detections
- GPS anomalies
- Farming patterns
- Manual review queue

**System Health:**
- Server status
- Database metrics
- API response times
- Error rates
- Cache hit rates

## MVP Scope

### Must Have:
- [x] Authentication (JWT)
- [x] Walking system
- [x] Territory capture (grid-based)
- [x] XP & levels
- [x] Badges (core types)
- [x] Streaks
- [x] Activity calendar
- [x] Leaderboards (all types)
- [x] User profile
- [x] Map screen
- [x] Admin dashboard
- [x] Anti-cheat (basic)

### Should Have (If Time):
- [ ] Vehicle detection
- [ ] Advanced badge types
- [ ] Achievement system
- [ ] Champion titles
- [ ] Push notifications

### Nice to Have (Post-MVP):
- [ ] Friend system
- [ ] Multiplayer challenges
- [ ] Social sharing
- [ ] Community features
- [ ] Advanced ML anti-cheat

## Non-Functional Requirements

**Performance:**
- API response time: < 200ms (p95)
- Map rendering: < 500ms
- Database queries: < 100ms
- Support 10,000 concurrent users (MVP scale)
- Support 1,000,000 territories

**Scalability:**
- Horizontal scaling ready
- Database partitioning strategy
- Caching layer (Redis)
- CDN for static assets
- Load balancing

**Reliability:**
- 99.5% uptime SLA
- Automated backups
- Disaster recovery plan
- Database replication
- Health checks

**Security:**
- HTTPS/TLS 1.3
- JWT token expiration (15 min)
- Refresh tokens (7 days)
- Rate limiting
- CSRF protection
- SQL injection prevention
- XSS prevention
- CORS configuration

**Compliance:**
- GDPR compliance
- Data retention policies
- User consent tracking
- Audit logging

## Future Versions (Post-MVP)

### V2: Social & Multiplayer
- Friend system
- Friend leaderboards
- Challenges (1v1)
- Team competitions
- Live location sharing (opt-in)
- Community forums
- Social feed

### V3: Advanced Features
- Offline walk recording
- Smartwatch integration
- Route planning
- Event system
- In-app purchases (cosmetics)
- Sponsorships
- Partnerships
- Bot challenges

### V4: Enterprise & Analytics
- Corporate walking programs
- Event hosting
- Custom territory rules
- Advanced analytics export
- API for partners
- White-label option
