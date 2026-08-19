# 2.5 Event Sourcing

**Event Sourcing** stores all changes as a sequence of events, rather than just the current state.

Event Sourcing is one possible way to implement the _write side_ of a system — it does not require CQRS, and CQRS (covered in the previous chapter) does not require it either. You can event-source an aggregate and still expose a single, simple read model straight off the event store; you can also do full CQRS with two databases and never touch Event Sourcing at all. This chapter treats it as its own, independent pattern.

It's also worth flagging a shift in vocabulary from the DDD chapter. There, a Domain Event was a _side effect_: the aggregate held its current state in fields, and an event was published _in addition to_ saving that state, mostly to notify other parts of the system. Here, there is no separate "current state" being saved — **the events themselves are the state**. Nothing is persisted except the sequence of events; "current state" becomes something you compute by replaying them. If you're carrying over the DDD chapter's mental model of events as a side channel, set it aside for this chapter — it's event-first, not state-first.

## Traditional State Storage

```
Database:
┌────────────────────────────────┐
│ projects table                 │
├────┬──────────┬─────────┬──────┤
│ id │   name   │ status  │budget│
├────┼──────────┼─────────┼──────┤
│ 1  │Website   │completed│50000 │  ← Current state only
└────┴──────────┴─────────┴──────┘

History lost! How did we get here?
- When was status changed?
- Who changed the budget?
- What was the previous name?
```

## Event Sourcing Approach

```
Event Store:
┌───────────────────────────────────────────────────────┐
│ events table                                          │
├────┬───────────────────────┬──────────────┬───────────┤
│ id │       event_type      │    data      │timestamp  │
├────┼───────────────────────┼──────────────┼───────────┤
│ 1  │ProjectCreated         │{name:Website}│2024-01-01 │
│ 2  │BudgetSet              │{budget:40000}│2024-01-05 │
│ 3  │StatusChanged          │{from:active, │2024-01-15 │
│    │                       │ to:review}   │           │
│ 4  │BudgetIncreased        │{from:40000,  │2024-01-20 │
│    │                       │ to:50000}    │           │
│ 5  │StatusChanged          │{from:review, │2024-02-01 │
│    │                       │ to:completed}│           │
└────┴───────────────────────┴──────────────┴───────────┘

Complete history! Can replay to any point in time.
```

## Event Sourcing Implementation

### 1. Define Events

Every event below is the immutable record of a Command that was accepted — a `changeBudget(...)` command call, once validated, is what produces a `ProjectBudgetChangedEvent`. The command decides _whether_ something is allowed to happen, the event records _that_ it happened.

```javascript
// Base Event
class DomainEvent {
  constructor(aggregateId, aggregateType, eventType, data, metadata = {}) {
    this.eventId = uuid();
    this.aggregateId = aggregateId;
    this.aggregateType = aggregateType;
    this.eventType = eventType;
    this.data = data;
    this.metadata = {
      ...metadata,
      timestamp: new Date(),
      version: 1,
    };
  }
}

// Specific Events
class ProjectCreatedEvent extends DomainEvent {
  constructor(projectId, tenantId, name, createdBy) {
    super(
      projectId,
      "Project",
      "ProjectCreated",
      {
        tenantId,
        name,
        createdBy,
      },
      { userId: createdBy },
    );
  }
}

class ProjectBudgetChangedEvent extends DomainEvent {
  constructor(projectId, oldBudget, newBudget, changedBy) {
    super(
      projectId,
      "Project",
      "ProjectBudgetChanged",
      {
        oldBudget,
        newBudget,
        changedBy,
      },
      { userId: changedBy },
    );
  }
}

class ProjectStatusChangedEvent extends DomainEvent {
  constructor(projectId, oldStatus, newStatus, changedBy) {
    super(
      projectId,
      "Project",
      "ProjectStatusChanged",
      {
        oldStatus,
        newStatus,
        changedBy,
      },
      { userId: changedBy },
    );
  }
}
```

**A word of caution on `metadata.version`.** The `version: 1` field set inside `DomainEvent` is easy to misread as "this is the 3rd event for this aggregate" — it isn't. It's the **schema version** of the event's shape, used later for upcasting (section 9) when `ProjectCreatedEvent`'s `data` structure changes over time. It's hardcoded to `1` here because every event starts life as schema v1; it only changes when an upcaster rewrites an old event into a newer shape. Where an event sits in a _sequence_ — 1st, 2nd, 3rd change to this particular project — is a completely different number, tracked separately by the event store (see the `version` column in section 3). Two fields, same English word, unrelated meanings. Keep them apart mentally, because section 7 shows what happens when code doesn't.

### 2. Aggregate that Produces Events

```javascript
class Project {
  constructor() {
    this.id = null;
    this.tenantId = null;
    this.name = null;
    this.status = null;
    this.budget = null;
    this.createdBy = null;
    this.version = 0;
    this.uncommittedEvents = [];
  }

  // Factory method - produces ProjectCreated event
  static create(projectId, tenantId, name, budget, userId) {
    const project = new Project();

    const event = new ProjectCreatedEvent(projectId, tenantId, name, userId);

    project.applyEvent(event);
    project.uncommittedEvents.push(event);

    return project;
  }

  // Command - produces BudgetChanged event
  changeBudget(newBudget, userId) {
    if (newBudget < 0) {
      throw new ValidationError("Budget cannot be negative");
    }

    const event = new ProjectBudgetChangedEvent(
      this.id,
      this.budget,
      newBudget,
      userId,
    );

    this.applyEvent(event);
    this.uncommittedEvents.push(event);
  }

  // Command - produces StatusChanged event
  changeStatus(newStatus, userId) {
    if (!this.canChangeStatus(newStatus)) {
      throw new ValidationError(
        `Cannot change from ${this.status} to ${newStatus}`,
      );
    }

    const event = new ProjectStatusChangedEvent(
      this.id,
      this.status,
      newStatus,
      userId,
    );

    this.applyEvent(event);
    this.uncommittedEvents.push(event);
  }

  // Apply event to change state
  applyEvent(event) {
    switch (event.eventType) {
      case "ProjectCreated":
        this.id = event.aggregateId;
        this.tenantId = event.data.tenantId;
        this.name = event.data.name;
        this.status = "active";
        this.createdBy = event.data.createdBy;
        break;

      case "ProjectBudgetChanged":
        this.budget = event.data.newBudget;
        break;

      case "ProjectStatusChanged":
        this.status = event.data.newStatus;
        break;

      default:
        throw new Error(`Unknown event type: ${event.eventType}`);
    }

    this.version++;
  }

  // Reconstruct from events
  static fromEvents(events) {
    const project = new Project();

    for (const event of events) {
      project.applyEvent(event);
    }

    return project;
  }

  getUncommittedEvents() {
    return [...this.uncommittedEvents];
  }

  clearUncommittedEvents() {
    this.uncommittedEvents = [];
  }

  canChangeStatus(newStatus) {
    const validTransitions = {
      active: ["on_hold", "completed", "cancelled"],
      on_hold: ["active", "cancelled"],
      completed: ["archived"],
      cancelled: ["archived"],
      archived: [],
    };

    return validTransitions[this.status]?.includes(newStatus) || false;
  }
}
```

### 3. Event Store

```javascript
class EventStore {
  constructor(db) {
    this.db = db;
  }

  // Save events
  async save(aggregateId, events, expectedVersion) {
    await this.db.transaction(async (trx) => {
      // Check for concurrency conflicts
      const currentVersion = await this.getCurrentVersion(trx, aggregateId);

      if (currentVersion !== expectedVersion) {
        throw new ConcurrencyError(
          `Expected version ${expectedVersion}, but current version is ${currentVersion}`,
        );
      }

      // Insert events
      for (const event of events) {
        await trx.query(
          `
          INSERT INTO events (
            event_id,
            aggregate_id,
            aggregate_type,
            event_type,
            data,
            metadata,
            version,
            created_at
          ) VALUES ($1, $2, $3, $4, $5, $6, $7, $8)
        `,
          [
            event.eventId,
            event.aggregateId,
            event.aggregateType,
            event.eventType,
            JSON.stringify(event.data),
            JSON.stringify(event.metadata),
            expectedVersion + 1,
            event.metadata.timestamp,
          ],
        );

        expectedVersion++;
      }
    });
  }

  // Load events for an aggregate
  async getEventsForAggregate(aggregateId) {
    const result = await this.db.query(
      `
      SELECT * FROM events
      WHERE aggregate_id = $1
      ORDER BY version ASC
    `,
      [aggregateId],
    );

    return result.rows.map((row) => this.deserializeEvent(row));
  }

  // Get all events (for projections), ordered across ALL aggregates
  //
  // Deliberately NOT using the `version` column here: version is scoped
  // to a single aggregate_id (see the UNIQUE(aggregate_id, version)
  // constraint below) — every aggregate independently has a version 1,
  // a version 2, and so on. Comparing it across different aggregates
  // gives no meaningful order at all. `global_position` is a separate,
  // table-wide BIGSERIAL that increases monotonically regardless of
  // which aggregate wrote the row — that's what a cross-aggregate
  // consumer like an EventProcessor needs to walk the log in order and
  // resume from where it left off. See section 7 for how it's used.
  async getAllEvents(fromPosition = 0) {
    const result = await this.db.query(
      `
      SELECT * FROM events
      WHERE global_position > $1
      ORDER BY global_position ASC
    `,
      [fromPosition],
    );

    return result.rows.map((row) => this.deserializeEvent(row));
  }

  async getCurrentVersion(trx, aggregateId) {
    const result = await trx.query(
      `
      SELECT MAX(version) as version
      FROM events
      WHERE aggregate_id = $1
    `,
      [aggregateId],
    );

    return result.rows[0].version || 0;
  }

  deserializeEvent(row) {
    const EventClass = this.getEventClass(row.event_type);

    const event = Object.create(EventClass.prototype);
    event.eventId = row.event_id;
    event.aggregateId = row.aggregate_id;
    event.aggregateType = row.aggregate_type;
    event.eventType = row.event_type;
    event.data = JSON.parse(row.data);
    event.metadata = JSON.parse(row.metadata);
    event.aggregateVersion = row.version; // per-aggregate sequence
    event.globalPosition = row.global_position; // cross-aggregate cursor

    return event;
  }

  getEventClass(eventType) {
    const eventClasses = {
      ProjectCreated: ProjectCreatedEvent,
      ProjectBudgetChanged: ProjectBudgetChangedEvent,
      ProjectStatusChanged: ProjectStatusChangedEvent,
    };

    return eventClasses[eventType] || DomainEvent;
  }
}
```

**Optimistic Concurrency Control.** The `expectedVersion` check above is a named technique, not just a defensive `if`. Two commands can race to modify the same aggregate at the same time — for example, two users both trying to change a project's status a moment apart, both having loaded version 4. Whoever saves first moves the aggregate to version 5; the second save, still expecting version 4, is rejected with a `ConcurrencyError` rather than silently overwriting the first change. This is called **optimistic concurrency control**: instead of locking the aggregate up front (pessimistic locking), you assume conflicts are rare, proceed, and only check for a conflict at save time. When a conflict is caught, the usual responses are: reload the aggregate (picking up the version that just won), retry the command against the fresh state, or, if that's not safe to do automatically, surface a conflict to the user ("someone else already changed this — please refresh").

### 4. Repository Using the Event Store

```javascript
class EventSourcedProjectRepository {
  constructor(eventStore) {
    this.eventStore = eventStore;
  }

  // Load aggregate by replaying events
  async findById(projectId) {
    const events = await this.eventStore.getEventsForAggregate(projectId);

    if (events.length === 0) {
      return null;
    }

    // Reconstruct aggregate from events
    return Project.fromEvents(events);
  }

  // Save aggregate by storing new events
  async save(project) {
    const events = project.getUncommittedEvents();

    if (events.length === 0) {
      return; // No changes
    }

    const expectedVersion = project.version - events.length;

    await this.eventStore.save(project.id, events, expectedVersion);

    project.clearUncommittedEvents();
  }
}
```

### 5. Usage Example, Including Time Travel

```javascript
class ProjectService {
  constructor(projectRepository, eventStore) {
    this.projectRepo = projectRepository;
    this.eventStore = eventStore;
  }

  async createProject(tenantId, userId, projectData) {
    // Create new project (produces ProjectCreated event)
    const project = Project.create(
      uuid(),
      tenantId,
      projectData.name,
      projectData.budget,
      userId,
    );

    // Save (stores events)
    await this.projectRepo.save(project);

    return project.id;
  }

  async changeBudget(projectId, newBudget, userId) {
    // Load project (by replaying events)
    const project = await this.projectRepo.findById(projectId);

    if (!project) {
      throw new NotFoundError("Project not found");
    }

    // Execute command (produces event)
    project.changeBudget(newBudget, userId);

    // Save (stores new events)
    await this.projectRepo.save(project);
  }

  async changeStatus(projectId, newStatus, userId) {
    const project = await this.projectRepo.findById(projectId);

    if (!project) {
      throw new NotFoundError("Project not found");
    }

    project.changeStatus(newStatus, userId);
    await this.projectRepo.save(project);
  }

  // Time travel: get project state at specific point in time
  async getProjectAtTime(projectId, timestamp) {
    const allEvents = await this.eventStore.getEventsForAggregate(projectId);

    // Filter events up to the timestamp
    const eventsUntilTime = allEvents.filter(
      (event) => event.metadata.timestamp <= timestamp,
    );

    if (eventsUntilTime.length === 0) {
      return null;
    }

    return Project.fromEvents(eventsUntilTime);
  }
}
```

---

## Advanced Event Sourcing Patterns

### 6. Projections (Read Models)

Projections build optimized read models from events for efficient querying.

```javascript
class ProjectProjection {
  constructor(db) {
    this.db = db;
  }

  // Project events into read model
  async project(event) {
    switch (event.eventType) {
      case "ProjectCreated":
        await this.handleProjectCreated(event);
        break;
      case "ProjectBudgetChanged":
        await this.handleBudgetChanged(event);
        break;
      case "ProjectStatusChanged":
        await this.handleStatusChanged(event);
        break;
    }
  }

  async handleProjectCreated(event) {
    // ON CONFLICT DO NOTHING, not a plain INSERT: this handler WILL be
    // called more than once for the same event over the system's
    // lifetime — a catch-up after a restart, a full projection rebuild,
    // an at-least-once delivery retry from a broker. Without this, the
    // second call hits the primary key on `id` and the processor
    // crashes. This is the "idempotent projections" principle from the
    // end of this chapter, applied concretely.
    await this.db.query(
      `
      INSERT INTO project_read_model (
        id, tenant_id, name, status, budget, created_by, created_at
      ) VALUES ($1, $2, $3, $4, $5, $6, $7)
      ON CONFLICT (id) DO NOTHING
    `,
      [
        event.aggregateId,
        event.data.tenantId,
        event.data.name,
        "active",
        0,
        event.data.createdBy,
        event.metadata.timestamp,
      ],
    );
  }

  async handleBudgetChanged(event) {
    await this.db.query(
      `
      UPDATE project_read_model
      SET budget = $1, updated_at = $2
      WHERE id = $3
    `,
      [event.data.newBudget, event.metadata.timestamp, event.aggregateId],
    );
  }

  async handleStatusChanged(event) {
    await this.db.query(
      `
      UPDATE project_read_model
      SET status = $1, updated_at = $2
      WHERE id = $3
    `,
      [event.data.newStatus, event.metadata.timestamp, event.aggregateId],
    );
  }

  // Query methods
  async findByTenant(tenantId) {
    const result = await this.db.query(
      `SELECT * FROM project_read_model WHERE tenant_id = $1`,
      [tenantId],
    );
    return result.rows;
  }

  async findActiveProjects() {
    const result = await this.db.query(
      `SELECT * FROM project_read_model WHERE status = 'active'`,
    );
    return result.rows;
  }
}
```

**Rebuilding a projection from scratch.** The catch-up logic in the next section handles a projection that's simply behind on the newest events. But there's a different, very common operational task: a bug is found in `handleBudgetChanged` (say, a wrong currency conversion), it gets fixed in the code, and now the _existing_ rows in `project_read_model` are wrong too — reprocessing only new events won't fix them. The fix is to **rebuild the projection**: truncate the read table (or build a brand-new one), reset the "last processed version" back to zero, and replay the entire event history from the beginning through the corrected projection logic. Because this can take a while on a large event store, it's common to build the new version of a projection under a different name/table, let it catch up fully in the background, and only then switch reads over to it — a blue/green deployment for a projection rather than a service.

### 7. Event Processor for Multiple Projections

**What an Event Processor actually is.** Strip away the code for a moment. An Event Processor is a component that consumes the event stream and derives something else from it — most often a read model, but not only that (a notification, an outbound integration call, a command sent to a different aggregate — a Saga in section 10 is really a specialized Event Processor whose "projection" issues commands instead of writing rows). Whatever it produces, it has exactly three jobs:

1. **Read events in order** — across all the aggregates it cares about, without skipping any.
2. **React to each one** — hand it to whatever logic knows what to do with that event type.
3. **Remember where it stopped** — so a restart resumes instead of reprocessing everything, or missing what came in while it was down.

This is a generic role, not a single fixed pattern — you'll see the same idea called a **Projector**, **Subscriber**, **Denormalizer**, or **Event Handler** depending on the codebase and the community. The name in this chapter is just one of several in common use.

**Pull vs. push.** The implementation below _polls_ — it asks the event store "anything new?" on a timer. That's one valid way for an Event Processor to learn an event exists, but not the only one: with a message broker, the store publishes each event right after persisting it, and the processor is _notified_ instead of asking. The three responsibilities above don't change either way — only how the processor learns an event exists does. Polling trades a small, bounded delay (here, up to the 1-second interval) for simplicity: no broker to run, no dual-write between the database and a queue to reconcile.

**Why iterate over every projection instead of picking "the right one"?** Because the relationship between an event and projections isn't one-to-one — it's fan-out. A single `ProjectCreated` event might matter to the main read model, an audit log, a billing projection, and an analytics pipeline simultaneously, or to none of them. Rather than have the processor maintain a registry of "who cares about what" — which couples it to every projection's internals and has to be updated each time a projection changes what it listens for — each projection decides for itself, via its own `switch`, what it cares about and silently ignores the rest. Calling `project()` on a projection that ends up doing nothing is a cheap in-memory dispatch, not I/O; it only becomes worth optimizing (e.g. a `handledEventTypes()` filter) with very many projections or expensive handlers.

**The cursor bug this chapter used to have.** An earlier version of this code advanced its cursor with `this.lastProcessedVersion = event.metadata.version`. Two problems, both worth naming explicitly because they're easy to reintroduce: first, `metadata.version` is the _schema_ version flagged in the callout back in section 1 — it's `1` for practically every event, so the cursor never meaningfully moved. Second, even the database's own `version` column wouldn't have fixed it, because that column is scoped _per aggregate_ — every project's event history starts its own `version` back at 1, so it carries no valid ordering across different aggregates. What a cross-aggregate consumer needs is `global_position` (introduced in section 3): a single, table-wide, monotonically increasing sequence, independent of both of the above.

**The checkpoint also needs to survive a restart.** Keeping the cursor in an instance field, as below, means every process restart forgets it and reprocesses the entire event store from the beginning. That's harmless _only_ because `handleProjectCreated` is now idempotent (`ON CONFLICT DO NOTHING`, section 6) — without that, a restart would crash the processor on the first duplicate insert. In production, the checkpoint is persisted, typically in a small table keyed by processor name:

```sql
CREATE TABLE processor_checkpoints (
  processor_name VARCHAR(255) PRIMARY KEY,
  last_position BIGINT NOT NULL DEFAULT 0,
  updated_at TIMESTAMP NOT NULL
);
```

```javascript
class EventProcessor {
  constructor(eventStore, projections, db, processorName = "default") {
    this.eventStore = eventStore;
    this.projections = projections;
    this.db = db;
    this.processorName = processorName;
    this.lastProcessedPosition = 0;
  }

  async start() {
    this.lastProcessedPosition = await this.loadCheckpoint();

    // Initial catch-up — on a fresh checkpoint (0), this replays the
    // entire event store, which is exactly what a full projection
    // rebuild needs. On a warm checkpoint, it only replays what's new
    // since the last run. Same code path, different starting position.
    await this.catchUp();

    // Poll for new events
    this.interval = setInterval(() => this.processNewEvents(), 1000);
  }

  async catchUp() {
    await this.processNewEvents();
  }

  async processNewEvents() {
    const events = await this.eventStore.getAllEvents(
      this.lastProcessedPosition,
    );

    for (const event of events) {
      await this.processEvent(event);
      this.lastProcessedPosition = event.globalPosition;
      await this.saveCheckpoint(this.lastProcessedPosition);
    }
  }

  async processEvent(event) {
    // Send event to all projections
    for (const projection of this.projections) {
      try {
        await projection.project(event);
      } catch (error) {
        console.error(`Error in projection: ${error.message}`);
        // Handle projection errors (retry, dead letter queue, etc.)
      }
    }
  }

  async loadCheckpoint() {
    const result = await this.db.query(
      `SELECT last_position FROM processor_checkpoints WHERE processor_name = $1`,
      [this.processorName],
    );
    return result.rows[0]?.last_position ?? 0;
  }

  async saveCheckpoint(position) {
    await this.db.query(
      `
      INSERT INTO processor_checkpoints (processor_name, last_position, updated_at)
      VALUES ($1, $2, $3)
      ON CONFLICT (processor_name) DO UPDATE
      SET last_position = $2, updated_at = $3
    `,
      [this.processorName, position, new Date()],
    );
  }

  stop() {
    if (this.interval) {
      clearInterval(this.interval);
    }
  }
}
```

**Scoping a processor to less than "everything."** Nothing requires a single Event Processor that fans out across every aggregate type, as above. It's just as common to run several, each narrower — one per aggregate type, one per bounded context, or one per individual projection — by filtering at the source instead of relying on projections to self-filter:

```sql
SELECT * FROM events
WHERE aggregate_type = 'Project' AND global_position > $1
ORDER BY global_position ASC
```

The one thing this requires: **each processor needs its own checkpoint**, not a shared one — `processor_checkpoints` above is already keyed by `processor_name` for exactly this reason. A `BillingProcessor` and a broad `AuditProcessor` don't advance at the same rate or over the same subset of events, so they can't share a single position. This is the same idea as Kafka consumer groups, where each group tracks its own offset independently, even when reading the same underlying stream.

**A narrower rebuild doesn't need `global_position` at all.** If what you're rebuilding only ever touches one aggregate at a time — like `project_read_model`, where each row is independent and the relative order between, say, `p1`'s events and `p2`'s events doesn't affect the final result — you don't strictly need a cross-aggregate cursor. An alternative is to iterate over distinct `aggregate_id`s of that type and replay each one's own history in full (already correctly ordered by its per-aggregate `version`), tracking "which aggregate IDs are done" as the checkpoint instead of a stream position. `global_position` earns its keep specifically when you need a resumable, ordered view _across_ aggregates — not for every rebuild.

### 8. Snapshots for Performance

**What a snapshot is.** A snapshot is a saved copy of a single aggregate's state at a specific `version`, kept so that loading that aggregate doesn't require replaying its entire event history from the beginning every time. Reconstructing a 5-event aggregate via `Project.fromEvents(events)` is trivial; reconstructing one with 50,000 events on every `findById()` call is not — most of that cost is wasted, since only the final state is actually needed.

**It's a cache, not a new source of truth.** This is the key distinction from a projection: a projection (section 6) derives a _different_ view — a denormalized table, shaped for queries the write model doesn't answer directly. A snapshot contains exactly the same fields as the aggregate itself; it's disposable and reconstructible from events at any time. If a snapshot is lost or corrupted, the worst outcome is a slower load (fall back to a full replay) — never incorrect state, as long as the replay logic itself is correct. It's also scoped to a single aggregate instance, unlike a projection or an Event Processor, which typically span many.

For aggregates with many events, snapshots improve load performance.

```javascript
class SnapshotStore {
  constructor(db) {
    this.db = db;
  }

  async saveSnapshot(aggregateId, snapshot, version) {
    await this.db.query(
      `
      INSERT INTO snapshots (aggregate_id, data, version, created_at)
      VALUES ($1, $2, $3, $4)
      ON CONFLICT (aggregate_id) DO UPDATE
      SET data = $2, version = $3, created_at = $4
    `,
      [aggregateId, JSON.stringify(snapshot), version, new Date()],
    );
  }

  async getSnapshot(aggregateId) {
    const result = await this.db.query(
      `SELECT * FROM snapshots WHERE aggregate_id = $1`,
      [aggregateId],
    );

    if (result.rows.length === 0) {
      return null;
    }

    return {
      data: JSON.parse(result.rows[0].data),
      version: result.rows[0].version,
    };
  }
}

// Enhanced Repository with Snapshots
class SnapshotProjectRepository extends EventSourcedProjectRepository {
  constructor(eventStore, snapshotStore) {
    super(eventStore);
    this.snapshotStore = snapshotStore;
    this.snapshotFrequency = 50; // Snapshot every 50 events
  }

  async findById(projectId) {
    // Try to load from snapshot
    const snapshot = await this.snapshotStore.getSnapshot(projectId);

    let project;
    let fromVersion = 0;

    if (snapshot) {
      // Reconstruct from snapshot
      project = new Project();
      Object.assign(project, snapshot.data);
      fromVersion = snapshot.version;
    } else {
      project = new Project();
    }

    // Load events since snapshot. Note: filtering on `aggregateVersion`
    // (the per-aggregate sequence from the `version` column), not
    // `metadata.version` (the schema version, hardcoded to 1 — see the
    // callout in section 1). Filtering on metadata.version here would
    // compare "1" to a snapshot version like 50 and silently find zero
    // new events every time, defeating the whole point of snapshotting.
    const events = await this.eventStore.getEventsForAggregate(projectId);
    const newEvents = events.filter((e) => e.aggregateVersion > fromVersion);

    // Apply new events
    for (const event of newEvents) {
      project.applyEvent(event);
    }

    return project.id ? project : null;
  }

  async save(project) {
    await super.save(project);

    // Create snapshot if needed
    if (project.version % this.snapshotFrequency === 0) {
      const snapshot = {
        id: project.id,
        tenantId: project.tenantId,
        name: project.name,
        status: project.status,
        budget: project.budget,
        createdBy: project.createdBy,
        version: project.version,
      };

      await this.snapshotStore.saveSnapshot(
        project.id,
        snapshot,
        project.version,
      );
    }
  }
}
```

### 9. Event Upcasting (Versioning)

Handle event schema evolution over time.

```javascript
class EventUpcaster {
  upcast(event) {
    const upcasters = {
      ProjectCreated: this.upcastProjectCreated,
      ProjectBudgetChanged: this.upcastProjectBudgetChanged,
    };

    const upcaster = upcasters[event.eventType];
    return upcaster ? upcaster(event) : event;
  }

  upcastProjectCreated(event) {
    // V1 -> V2: Add default deadline field
    if (event.metadata.version === 1) {
      return {
        ...event,
        data: {
          ...event.data,
          deadline: null, // New field in V2
        },
        metadata: {
          ...event.metadata,
          version: 2,
          upcastedFrom: 1,
        },
      };
    }

    return event;
  }

  upcastProjectBudgetChanged(event) {
    // V1 -> V2: Add reason field
    if (event.metadata.version === 1) {
      return {
        ...event,
        data: {
          ...event.data,
          reason: "Not specified", // New required field in V2
        },
        metadata: {
          ...event.metadata,
          version: 2,
          upcastedFrom: 1,
        },
      };
    }

    return event;
  }
}

// Use in EventStore
class EventStoreWithUpcasting extends EventStore {
  constructor(db, upcaster) {
    super(db);
    this.upcaster = upcaster;
  }

  deserializeEvent(row) {
    const event = super.deserializeEvent(row);
    return this.upcaster.upcast(event);
  }
}
```

### 10. Sagas / Process Managers

So far, every command has stayed inside a single aggregate. But real workflows often need to span several: "when a project is created, reserve its budget in the Billing aggregate, and once that succeeds, notify the manager." No single aggregate can do all three steps atomically — they're different aggregates, possibly in different Bounded Contexts, and (per the rule from the DDD chapter) one transaction should touch one aggregate.

A **Saga** (also called a **Process Manager**) is the pattern that coordinates this kind of multi-step workflow. It listens for events, and in response issues new commands — often targeting a different aggregate than the one that raised the event.

```javascript
// Saga: reacts to events, issues new commands
class ProjectOnboardingSaga {
  constructor(commandBus) {
    this.commandBus = commandBus;
  }

  async handle(event) {
    if (event.eventType === "ProjectCreated") {
      await this.commandBus.execute(
        new ReserveBudgetCommand(event.aggregateId, event.data.tenantId),
      );
    }

    if (event.eventType === "BudgetReserved") {
      await this.commandBus.execute(
        new NotifyManagerCommand(event.aggregateId),
      );
    }

    if (event.eventType === "BudgetReservationFailed") {
      // Compensating action: undo/flag the project instead of silently stalling
      await this.commandBus.execute(
        new FlagProjectForReviewCommand(event.aggregateId),
      );
    }
  }
}
```

The key difference from an aggregate transaction: a Saga is **not atomic**. Each step can succeed or fail independently, and time passes between them. If a later step fails, there's no automatic rollback of the earlier one — the Saga itself is responsible for issuing a **compensating command** (like `FlagProjectForReviewCommand` above) to bring the system back to a sensible state. Designing a Saga means designing not just the happy path, but what happens at each point it can fail partway through.

### 11. Testing an Event-Sourced Aggregate

Event-sourced aggregates lead to a distinctive, and often very pleasant, test style, usually described as **Given/When/Then**:

- **Given** a list of events that already happened to this aggregate
- **When** a command is executed against it
- **Then** assert on the new event(s) produced — or the specific error thrown

```javascript
test("cannot change budget to a negative amount", () => {
  // Given
  const project = Project.fromEvents([
    new ProjectCreatedEvent("p1", "tenant1", "Website Redesign", "user1"),
  ]);

  // When / Then
  expect(() => project.changeBudget(-100, "user1")).toThrow(
    "Budget cannot be negative",
  );
});

test("changing status from active to on_hold produces the expected event", () => {
  // Given
  const project = Project.fromEvents([
    new ProjectCreatedEvent("p1", "tenant1", "Website Redesign", "user1"),
  ]);

  // When
  project.changeStatus("on_hold", "user1");

  // Then
  const [newEvent] = project.getUncommittedEvents();
  expect(newEvent.eventType).toBe("ProjectStatusChanged");
  expect(newEvent.data.newStatus).toBe("on_hold");
});
```

This is frequently cited as one of the most practical day-to-day benefits of Event Sourcing: there's no database to mock, no ORM to stub out — "Given" is just an array of plain event objects, and "Then" is just an assertion on the array of events the command produced. The tests end up reading almost like the business scenario itself, which makes them useful documentation as well as regression protection.

---

## Benefits and Trade-offs

### Benefits

**Complete Audit Trail**: Every change is recorded with who, what, when, and why.

**Time Travel**: Query system state at any point in history.

**Event Replay**: Rebuild state, fix bugs retroactively, create new projections from historical data.

**Business Intelligence**: Analyze patterns, user behavior, and trends over time.

**Debugging**: Reproduce bugs by replaying exact sequence of events.

**Temporal Queries**: "Show me all projects that were active on January 15th."

**Integration**: Events can be published to other systems for real-time integration.

### Trade-offs

**Complexity**: More complex than CRUD, requires careful event design.

**Storage**: Events accumulate over time (mitigated by snapshots and archiving).

**Query Performance**: Rebuilding state from events can be slow (solved with projections).

**Eventual Consistency**: Read models may lag behind write model.

**Event Schema Evolution**: Changing event structure requires upcasting or versioning strategies.

### When to Use Event Sourcing

**Good fit**:

- Domain with complex business rules and important history
- Audit requirements (financial, healthcare, legal)
- Need for temporal queries
- Complex workflows requiring replay capability
- High-value data where every change matters

**Poor fit**:

- Simple CRUD applications
- Systems where only current state matters
- Performance-critical real-time systems with simple logic
- Small projects with limited resources

---

## Database Schema

```sql
-- Events table
CREATE TABLE events (
  event_id UUID PRIMARY KEY,
  aggregate_id UUID NOT NULL,
  aggregate_type VARCHAR(255) NOT NULL,
  event_type VARCHAR(255) NOT NULL,
  data JSONB NOT NULL,
  metadata JSONB NOT NULL,
  version INTEGER NOT NULL,        -- per-aggregate sequence (optimistic concurrency)
  global_position BIGSERIAL,       -- table-wide, monotonic (cross-aggregate cursor)
  created_at TIMESTAMP NOT NULL,

  UNIQUE(aggregate_id, version)
);

CREATE INDEX idx_events_aggregate ON events(aggregate_id);
CREATE INDEX idx_events_type ON events(event_type);
CREATE INDEX idx_events_created ON events(created_at);
CREATE INDEX idx_events_global_position ON events(global_position);

-- Event Processor checkpoints (one row per processor, so independently
-- scoped processors don't share a position — see section 7)
CREATE TABLE processor_checkpoints (
  processor_name VARCHAR(255) PRIMARY KEY,
  last_position BIGINT NOT NULL DEFAULT 0,
  updated_at TIMESTAMP NOT NULL
);

-- Snapshots table
CREATE TABLE snapshots (
  aggregate_id UUID PRIMARY KEY,
  data JSONB NOT NULL,
  version INTEGER NOT NULL,
  created_at TIMESTAMP NOT NULL
);

-- Read model (projection)
CREATE TABLE project_read_model (
  id UUID PRIMARY KEY,
  tenant_id UUID NOT NULL,
  name VARCHAR(255) NOT NULL,
  status VARCHAR(50) NOT NULL,
  budget DECIMAL(12, 2),
  created_by UUID NOT NULL,
  created_at TIMESTAMP NOT NULL,
  updated_at TIMESTAMP
);

CREATE INDEX idx_projects_tenant ON project_read_model(tenant_id);
CREATE INDEX idx_projects_status ON project_read_model(status);
```

---

## Purpose-Built Event Stores

Everything so far uses Postgres, with the event-store behaviors (per-aggregate versioning, global ordering, catch-up processing) built by hand on top of a general-purpose relational database. That's a legitimate, common choice — but it's worth knowing that dedicated event-store databases exist, because they turn several of the mechanisms built by hand in this chapter into native engine features.

**EventStoreDB (now KurrentDB)** is the best-known example. Event Store the company rebranded to Kurrent, and the product is now called KurrentDB — same underlying technology, new name. It's built specifically around an append-only event log, rather than adding one on top of a general-purpose database.

**Streams instead of hand-rolled aggregate versioning.** Every aggregate gets its own **stream** (e.g. `project-p1`), and each append to that stream produces a new stream version, checked against an expected version supplied by the caller — exactly the optimistic concurrency control this chapter implemented with `expectedVersion` and the `UNIQUE(aggregate_id, version)` constraint, except it's a first-class feature of the database rather than something to construct and maintain.

**Global ordering without the gap problem.** The database assigns each event a position that's both globally ordered across all streams and causally ordered within a stream — the same role `global_position` plays in this chapter's schema. The difference is durability of that guarantee: this chapter's version, built on a Postgres `BIGSERIAL`, has a known edge case under concurrent transactions (a lower position can become visible _after_ a higher one has already been read — see section 7). A database built around the event log from the ground up doesn't inherit that problem, because reads aren't racing writes on a column bolted onto a general-purpose table.

**Catch-up subscriptions instead of a hand-rolled `EventProcessor`.** This maps directly onto the pull-then-push pattern built in section 7: a catch-up subscription replays everything from a given position, then transparently switches to live push delivery for new events. The client still owns its checkpoint (much like this chapter's `processor_checkpoints` table), but the polling loop, the position tracking during catch-up, and the switch-over to live delivery are handled by the database rather than by application code.

**The trade-off.** These guarantees aren't free. A specialized event store gives up the general-purpose SQL this chapter's Postgres-based design keeps for free — no ad hoc joins, no rich analytical queries directly against the write side, and one more specialized piece of infrastructure to operate and monitor. In practice, many teams split the difference: use a dedicated event store as the write-side source of truth, and still project into Postgres (or another general-purpose store) for reads. The `EventProcessor` role from section 7 doesn't disappear in that setup — it just reads from a different kind of source.

## Key Principles

1. **Events are immutable**: Never modify or delete events, only append new ones.

2. **Events are facts**: Past tense, describe what happened: "OrderPlaced", not "PlaceOrder".

3. **Single source of truth**: The event store is the source of truth, projections are derived.

4. **Domain events only**: Only store domain-significant events, not every property change.

5. **Complete information**: Events should contain all information needed to understand what happened.

6. **Idempotent projections**: Replaying events should produce same result.
