# Super Alarm Clock: Implementation Design

## Overview

A mobile app that helps university students manage their sleep. Students set wake alarms and bedtime reminders, log when they sleep and wake, and receive yellow and red alerts when sleep debt builds. Three red alerts in a row notify a trusted contact. Collected data feeds a campus sleep research project.

Target stack for the course: Java, Apache Tapestry 5, Apache Cayenne ORM, Maven.

## System Parts

The brief asks for between 2 and 10 parts. This design uses 8.

| # | Part | Owner | Responsibility |
|---|------|-------|----------------|
| 1 | Student App (UI) | Our team | Displays state and captures input. Holds no business logic. |
| 2 | Alarm Scheduler | Our team | Stores alarms and fires them on the device. Runs locally so alarms ring without a network. |
| 3 | Sleep Event Logger | Our team | Records sleep and wake events. Decides whether a wake was caused by an alarm. |
| 4 | Sync Service | Our team | Sends events to the back end and receives alerts. Queues while offline and retries. |
| 5 | Alert Engine | Back-end team | Computes sleep debt and issues yellow and red alerts. |
| 6 | Guardian Notifier | Back-end team | Tracks consecutive red alerts. Messages the trusted contact at three. |
| 7 | Report Generator | Our team | Builds the Patterns view: averages, ignored alarms, streaks. |
| 8 | Research Data Store | Back-end team | Holds anonymized data for opted-in students. |

### Data flow

```
Student App ──► Alarm Scheduler ──► (alarm fires) ──► Student App
     │
     ├──► Sleep Event Logger ──► Sync Service ──► Back End
     │                                               │
     │                                   ┌───────────┼─────────────┐
     │                                   ▼           ▼             ▼
     │                             Alert Engine  Guardian     Research
     │                                   │       Notifier     Data Store
     │                                   ▼
     └◄──────────── alerts ◄──── Sync Service
     │
     └──► Report Generator (reads SleepEvents, Alarms, Alerts)
```

The app talks directly to parts 2, 3, 4, and 7. Everything else goes through the Sync Service.

## Cayenne Data Model

### Conventions

These match the Assignment 2 and 3 setup.

- DBEntity names are plural (`Students`). ObjEntity names are singular (`Student`).
- Every DBEntity has a `PK` column: INTEGER, Primary Key and Mandatory checked.
- Each persistent class gets the standard `getPK()` method.
- Enums are stored as VARCHAR(20), with the ObjEntity attribute's Java type set to the enum class. This is the same approach as the `State` enum in Assignment 3.

### Entities

#### Students → `Student`

| Column | DB Type | Java Type | Notes |
|--------|---------|-----------|-------|
| PK | INTEGER | Integer | Primary key |
| name | VARCHAR(50) | String | |
| email | VARCHAR(100) | String | Used for login |
| sleepGoal | DOUBLE | Double | Target hours per night. Default 8.0 |
| shareData | BOOLEAN | Boolean | Research opt-in |
| researchId | VARCHAR(36) | String | Random UUID. The only identifier sent to researchers |

#### TrustedContacts → `TrustedContact`

| Column | DB Type | Java Type | Notes |
|--------|---------|-----------|-------|
| PK | INTEGER | Integer | Primary key |
| name | VARCHAR(50) | String | |
| phoneOrEmail | VARCHAR(100) | String | |
| studentPK | INTEGER | (FK) | → Students.PK |

#### Alarms → `Alarm`

| Column | DB Type | Java Type | Notes |
|--------|---------|-----------|-------|
| PK | INTEGER | Integer | Primary key |
| type | VARCHAR(20) | `AlarmType` | `WAKE` or `BEDTIME` |
| time | TIME | `java.time.LocalTime` | Time of day |
| repeatDays | INTEGER | Integer | Bitmask. Mon = 1, Tue = 2, Wed = 4, ... Sun = 64. Zero means one-time |
| enabled | BOOLEAN | Boolean | |
| studentPK | INTEGER | (FK) | → Students.PK |

#### SleepEvents → `SleepEvent`

| Column | DB Type | Java Type | Notes |
|--------|---------|-----------|-------|
| PK | INTEGER | Integer | Primary key |
| type | VARCHAR(20) | `SleepEventType` | `SLEEP` or `WAKE` |
| timestamp | TIMESTAMP | `java.time.LocalDateTime` | |
| wokeByAlarm | BOOLEAN | Boolean | Only meaningful for `WAKE` events |
| alarmPK | INTEGER | (FK, nullable) | → Alarms.PK. The alarm that woke the student |
| studentPK | INTEGER | (FK) | → Students.PK |

#### Alerts → `Alert`

| Column | DB Type | Java Type | Notes |
|--------|---------|-----------|-------|
| PK | INTEGER | Integer | Primary key |
| level | VARCHAR(20) | `AlertLevel` | `YELLOW` or `RED` |
| issuedAt | TIMESTAMP | `java.time.LocalDateTime` | |
| acknowledged | BOOLEAN | Boolean | Student dismissed it |
| studentPK | INTEGER | (FK) | → Students.PK |

#### GuardianNotifications → `GuardianNotification`

| Column | DB Type | Java Type | Notes |
|--------|---------|-----------|-------|
| PK | INTEGER | Integer | Primary key |
| sentAt | TIMESTAMP | `java.time.LocalDateTime` | |
| studentPK | INTEGER | (FK) | → Students.PK |
| contactPK | INTEGER | (FK) | → TrustedContacts.PK |
| alertPK | INTEGER | (FK) | → Alerts.PK. The third red alert that triggered it |

### Relationships

| From | To | ObjRelationship names | Cardinality | Delete rule |
|------|----|-----------------------|-------------|-------------|
| Student | TrustedContact | `trustedContact` / `student` | 1 : 0..1 | Cascade |
| Student | Alarm | `alarms` / `student` | 1 : many | Cascade |
| Student | SleepEvent | `sleepEvents` / `student` | 1 : many | Cascade |
| Student | Alert | `alerts` / `student` | 1 : many | Cascade |
| Student | GuardianNotification | `guardianNotifications` / `student` | 1 : many | Cascade |
| Alarm | SleepEvent | `wakeEvents` / `alarm` | 1 : many | Nullify |
| TrustedContact | GuardianNotification | `notifications` / `contact` | 1 : many | Nullify |
| Alert | GuardianNotification | `notification` / `triggeringAlert` | 1 : 0..1 | Nullify |

Deleting an alarm nullifies the link on past wake events. The sleep history stays intact.

### Enums

Place these in `entities.enums`, next to `State`.

```java
public enum AlarmType      { WAKE, BEDTIME }
public enum SleepEventType { SLEEP, WAKE }
public enum AlertLevel     { YELLOW, RED }
```

## Service Layer

Follow the Assignment 3 pattern. Pages depend on interfaces. Cayenne implementations are bound in `AppModule`. Swapping a mock for Cayenne is one binding change.

| Interface | Key methods | Part |
|-----------|-------------|------|
| `AlarmService` | `listAlarms(student)`, `getNewAlarm()`, `updateAlarm(alarm)`, `deleteAlarm(PK)` | 2 |
| `SleepLogService` | `recordSleep(student)`, `recordWake(student)`, `listEvents(student, from, to)` | 3 |
| `AlertService` | `getCurrentLevel(student)`, `getRedStreak(student)`, `acknowledge(alertPK)` | 5 |
| `ReportService` | `getNights(student, days)`, `getAverageSleep(...)`, `getIgnoredAlarmCount(...)`, `findStreaks(...)` | 7 |
| `NotificationService` | `notifyContact(student, alert)` | 6 |

Example binding:

```java
public static SleepLogService buildSleepLogService(CayenneService cayenneService) {
    return new CayenneSleepLogService(cayenneService);
}
```

### Tapestry pages

| Page | Uses | Requirement covered |
|------|------|---------------------|
| `Tonight` | `SleepLogService`, `AlertService`, `AlarmService` | Log sleep and wake, show alerts |
| `alarms/Index`, `alarms/AddAlarm`, `alarms/EditAlarm` | `AlarmService` | Set wake and bedtime alarms |
| `Patterns` | `ReportService` | Sleep reports, ignored alarms, streaks |
| `Settings` | Student and TrustedContact data | Trusted contact, sleep goal, research opt-in |

All pages carry `@RequiresAuthentication`. A student only ever sees their own data.

## Business Rules

| ID | Rule |
|----|------|
| BR1 | A wake counts as woken by alarm if it is logged within 2 minutes after an enabled wake alarm fires. |
| BR2 | A night is a `SLEEP` event followed by the next `WAKE` event. Hours slept is the time between them. |
| BR3 | A `WAKE` without a prior `SLEEP` is rejected. A second `SLEEP` before a `WAKE` replaces the first. |
| BR4 | An alarm is ignored if it fires and no `WAKE` is logged within 2 minutes. Snoozes that lead to a later wake still count as one ignored alarm per firing. |
| BR5 | Sleep debt is the sum of `max(0, sleepGoal − hoursSlept)` over the last 7 nights. |
| BR6 | Three red alerts in a row trigger a guardian notification. A night at or above `sleepGoal − 0.5` hours resets the streak. |
| BR7 | Only one guardian notification is sent per streak. The next one requires a reset and three new reds. |

## Expected Back-End Behavior

The back-end team builds this. We specify what we expect.

### Alert thresholds

| Level | Trigger (either condition) |
|-------|----------------------------|
| Yellow | Awake 16+ hours, or sleep debt over 5 hours |
| Red | Awake 20+ hours, or sleep debt over 10 hours |

These numbers are placeholders. A sleep researcher should set the real values.

### Interface contract

- **Inbound:** the app posts `SleepEvent` records as JSON, each with a client-generated ID so retries don't create duplicates.
- **Outbound:** the back end pushes `Alert` records to the app. The app acknowledges receipt.
- **Evaluation timing:** the Alert Engine re-evaluates after every `SleepEvent` and on an hourly schedule, so an alert fires even if the student logs nothing.
- **Guardian notification:** sent by SMS or email from the back end, never from the student's device.

### Research export

- Only students with `shareData = true`.
- Records use `researchId`. No names, emails, or contact data.
- Fields: sleep and wake timestamps, woken-by-alarm flag, alert levels and times, ignored-alarm counts.

## Design Considerations

### Offline behavior
Alarms fire from the device, not the server. A dropped connection must never mean a missed alarm. Sleep events queue locally and sync when the connection returns. The brief assumes an always-connected device, but real networks drop, and the cost of a missed wake alarm is high.

### Clock accuracy
The 2-minute woken-by-alarm window depends on accurate timestamps. Record events with the device's time and the server's receipt time. If they differ by more than a minute, trust the device for BR1 and flag the record for researchers.

### Honest logging
The data is only as good as what students tap. Mitigations: a bedtime reminder that asks "Going to sleep?", and a morning prompt if a wake alarm fires but no `WAKE` is logged within 30 minutes.

### Privacy and consent
- Research sharing is opt-in and can be turned off at any time.
- The trusted contact is chosen by the student. The app explains when the contact gets messaged before the student saves one.
- Guardian messages say the student may be sleep-deprived. They include no sleep history.

### Alert fatigue
Too many alerts get ignored. Show at most one alert of each level per 6 hours unless the level rises.

### Accessibility
Large type for the clock and alert status. Alert state is shown with text, not color alone. All controls work by keyboard and screen reader.

### Design principles in play
- **SRP:** each service owns one concern. Streak and debt math live in `ReportService` and `AlertService`, not in page controllers.
- **DIP and OCP:** pages depend on service interfaces. Mock and Cayenne implementations swap in `AppModule`.
- **ISP:** alarm, logging, alert, and report services stay separate, so a page only depends on what it uses.
- **Low coupling:** the app and back end share only a JSON contract, so either team can change internals independently.

## Open Questions

1. Should snoozing count as ignoring an alarm, or only a full dismissal with no wake?
2. What happens when the trusted contact is unreachable? Retry, or notify a campus resource instead?
3. How long does the research store keep data?
4. Who sets the final yellow and red thresholds?

## Alternate Stack Mapping

The same design in Next.js, Prisma, and Postgres:

| Course stack | Alternate stack |
|--------------|-----------------|
| Cayenne DBEntity and ObjEntity | Prisma `model` in `schema.prisma` |
| Enum attribute stored as VARCHAR | Prisma `enum` mapped to a Postgres enum |
| `CayenneService` + service interfaces | Server-side modules called from route handlers or server actions |
| `AppModule` bindings | Module imports, or a small DI container if mocks are needed for tests |
| Tapestry pages and `.tml` templates | App Router pages with React components |
| Tynamo / Shiro security | Auth.js (or similar) with middleware route protection |
| Hand-built UI components | shadcn/ui (`Switch`, `Card`, `Tabs`) on Radix, styled with Tailwind |

The entities, relationships, business rules, and service boundaries carry over unchanged. Only persistence and wiring differ.
