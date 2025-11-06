# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Training assistant for analyzing Strava workout data and generating personalized training plans. The system uses a SQLite database synced from a remote server containing comprehensive workout history (371+ activities from March 2024 onwards across Run, Ride, GravelRide, Hike, Walk, WeightTraining, and Pickleball).

## Database Commands

### Sync Database from Server

```bash
mise run sync
```

This syncs `strava.db` from the remote Zeus server to `.data/strava.db`.

### Query Database

Direct SQLite access is pre-approved:

```bash
sqlite3 .data/strava.db "SELECT ..."
```

The `query-workouts` skill provides common query patterns and database documentation. Use it when analyzing workout data.

## Database Schema

The core data model consists of:

- **Activity** - Primary workout records with distance, time, heart rate, power, pace, elevation, gear, weather, and location
- **ActivityBestEffort** - Personal records at standardized distances achieved during activities
- **ActivityLap** - Lap-by-lap breakdown of workouts with per-lap metrics
- **ActivitySplit** - Standardized splits (mile/km) with pace zones
- **ActivityStream** - Time-series data (heart rate, power, cadence, altitude, position) with computed best averages and normalized power
- **SegmentEffort** - Performance on specific geographic segments with comparative data
- **Segment** - Reusable course segments with climb category and topology
- **Gear** - Equipment tracking (bikes, shoes) with total distance and retirement status
- **ChatMessage** - Conversation history for context persistence

### Key Schema Details

- **Distances** are in meters
- **Times** are in seconds
- **Dates** use ISO 8601 format: `YYYY-MM-DD HH:MM:SS`
- **ActivityStream.data** and **Activity.data** contain JSON with detailed metrics
- **ActivityStream.bestAverages** provides rolling averages (useful for FTP/threshold calculations)
- **ActivityStream.normalizedPower** is pre-computed for cycling workouts

## Training Plan Template

The `templates/training-plan.md` file defines the structure for generated training plans:

### Required Sections

1. **Goal** - Specific, measurable objective (format: progress from X to Y for Z)
2. **Current State** - Data-driven assessment from database queries:
   - Weekly volume (last 4-8 weeks from Activity table)
   - Weekly schedule and typical distances
   - Current fitness (pace/power/HR from recent activities)
   - Historical best (from ActivityBestEffort)
   - Recent training gaps
   - Age and additional training context
3. **General Philosophy** - Core training principles (e.g., 80/20 rule)
4. **Phases** - Periodized training blocks with:
   - Primary goals (2-3 per phase)
   - Weekly structure (day-by-day workouts with specifics)
   - Key notes (progressions, warnings, success indicators)

### Plan Generation Process

When creating training plans:

1. Query Activity table for recent volume and pacing baseline
2. Check ActivityBestEffort for historical performance context
3. Use ActivityStream for establishing heart rate/power zones
4. Apply 10% weekly volume increase rule for safe progression
5. Customize workout types based on sportType and user's activity patterns

## Data Analysis Patterns

### Recent Training Load

```sql
SELECT sportType, COUNT(*) as workouts,
       SUM(distance) as totalDistance,
       SUM(movingTimeInSeconds) / 3600.0 as hours
FROM Activity
WHERE startDateTime >= date('now', '-30 days')
GROUP BY sportType;
```

### Performance Trends

```sql
SELECT strftime('%Y-%W', startDateTime) as week,
       COUNT(*) as workouts,
       ROUND(AVG(averageSpeed), 2) as avgSpeed,
       ROUND(AVG(averageHeartRate), 0) as avgHR
FROM Activity
WHERE sportType = 'Run'
  AND startDateTime >= date('now', '-12 weeks')
GROUP BY week
ORDER BY week DESC;
```

### Personal Records

```sql
SELECT distanceInMeter,
       MIN(timeInSeconds) as bestTime,
       ROUND(MIN(timeInSeconds) / 60.0, 2) as minutesTime
FROM ActivityBestEffort
WHERE sportType = 'Run'
GROUP BY distanceInMeter
ORDER BY distanceInMeter;
```

## Conversation Context

ChatMessage table stores conversation history for maintaining context across sessions. Store important user preferences, training constraints, and goal evolution here.

## Architecture Notes

- Database is read-only from Claude's perspective (synced externally)
- All fitness data comes from Strava export/sync
- Template-driven plan generation ensures consistency
- Plans should reference specific data points from queries (e.g., "Based on your last 4 weeks averaging 15 miles...")

## Gym Equipment

- Adjustable Dumbells
- Weight Bench
- Cable Machine
- Treadmill  (10% incline / 10mph)
- Exercise Bike
- Weight Vest
- Calf Raise Block
- Exercise Bands
