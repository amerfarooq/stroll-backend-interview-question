
# Question Rotation System Design

## Architecture Overview

- **AWS RDS (PostgreSQL)**: Primary database
- **AWS ElastiCache (Redis)**: Caching layer for quick question lookups
- **AWS Lambda**: Question rotation logic and API endpoints
- **Amazon EventBridge**: Scheduling question rotation
- **AWS API Gateway**: RESTful API interface
- **AWS Parameter Store**: Store configuration such as cycle length


## Database Schema

```sql
-- Core tables for the rotation system
CREATE TABLE regions (
    region_id PRIMARY KEY,
    region_name VARCHAR(50)
);

CREATE TABLE questions (
    question_id PRIMARY KEY,
    content TEXT
);

CREATE TABLE region_questions (
    region_id INTEGER REFERENCES regions(region_id),
    question_id INTEGER REFERENCES questions(question_id),
    PRIMARY KEY (region_id, question_id)
);

CREATE TABLE cycles (
    cycle_id PRIMARY KEY,
    start_time TIMESTAMP,
    end_time TIMESTAMP,
    is_active BOOLEAN
);

CREATE TABLE cycle_assignments (
    cycle_id INTEGER REFERENCES cycles(cycle_id),
    region_id INTEGER REFERENCES regions(region_id),
    question_id INTEGER REFERENCES questions(question_id),
    PRIMARY KEY (cycle_id, region_id)
);
```

## API Endpoint

### Get Current Question for Region
-  **Endpoint**: `GET /api/v1/questions/current` 
-  **Purpose**: Invokes the `get_current_question` lambda to retrieve the currently active question for a specific region
-   **Parameters**:   `region` (required)
-  **Response**: Returns the current question details for that region 


## Implementation Details

### 1. Question Rotation Lambda Function

```python
def rotate_questions():
    # 1. Get configuration
    cycle_duration = get_ssm_parameter("cycle-duration")
    
    # 2. Get current cycle
    current_cycle = get_active_cycle()
    
    # 3. Create new cycle using configured duration
    new_cycle = {
        start_time: current_cycle.end_time,
        end_time: current_cycle.end_time + cycle_duration_days,
        is_active: true
    }
    
    # 4. Deactivate current cycle
    deactivate_cycle(current_cycle.id)
    
    # 5. Save new cycle
    save_cycle(new_cycle)
    
    # 6. Assign questions for each region
    for each_region in get_all_regions():
        next_question = find_next_unused_question(region)
        create_assignment(new_cycle.id, region.id, next_question.id)
    
    # 7. Update cache with configured TTL
    update_cache_with_ttl(cycle_duration_days)
```

### 2. Question Lookup Lambda Function

```python
def get_current_question(region):
    # 1. Try cache first
    cached_result = cache.get(region)
    if cached_result:
        return cached_result
    
    # 2. Cache miss - query database
    question = database.query("""
        SELECT question details
        FROM current cycle assignments
        WHERE region = given_region
        AND cycle is active
    """)
    
    # 3. Get configuration
    cycle_duration = get_ssm_parameter("cycle-duration")
    
    # 4. Update cache and return
    cache.set(region, question, ttl=cycle_duration )
    return question
```

## System Flow


### **1. Question Rotation**

-  Amazon EventBridge triggers the Question Rotation Lambda function according to the configured schedule (e.g., weekly).
-   The function creates a new cycle in RDS, deactivates the previous cycle, and assigns new questions to each region in the `cycle_assignments` table.
-   ElastiCache is updated with the latest region-question assignments, with a TTL matching the cycle duration for efficient lookups.

### 2. **Question Retrieval**

-   A user requests the current question through API Gateway, which invokes the Question Lookup Lambda function.
-   The Lambda function first checks ElastiCache for the question assigned to the user’s region.
    -   **Cache Hit**: If the question is in cache, it’s returned immediately.
    -   **Cache Miss**: If the cache doesn’t have the question, the function queries RDS for the latest assignment and updates the cache.

## Scaling Considerations

- Setting up read replicas for RDS to ensure that question retrieval remains fast and reliable even under heavy load.
-  ElastiCache can be clustered to scale and distribute the cache load across multiple nodes. Furthermore, these nodes can be spread across multiple regions to reduce latency for users.
-   Cache Expiration Strategy:
    -   **TTL Matching Cycle Duration**: The caches TTL can be set to match the cycle duration to reduce the load on RDS while maintaining fresh data for each cycle.
    -   **Cache Updates**: After each question rotation, the Question Rotation Lambda can refresh ElastiCache entries for each region, reducing initial cache misses.

## Pros and Cons

### Pros:

1.  Using ElastiCache speeds up access to current questions, which boosts performance and lowers the load on the database by serving common requests from memory.
2.  AWS services offer benefits like automatic scaling based on demand, a flexible pay-as-you-go pricing model, and a modular setup.
5.  Matching the caching TTL with cycle duration maximizes cache use while keeping data fresh and cutting down on unnecessary database queries.

### Cons:

1.  Managing cache updates and expirations can be complex and may result in stale data if not done properly.
