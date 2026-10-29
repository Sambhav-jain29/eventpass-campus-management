# Cloud and Jenkins Lab: One-Hour Challenge Submission Report

**Student Alias:** `progyan`  
**Class Code:** `001`  
**Challenge Code:** `00197`  
**Installation ID:** `c9f2e19f-dfcd-496a-855f-06d5bd723ab8`  
**Machine Label Hash:** `76c2eee410adc421660281030ff99982cdaa83de12b67387ba5338351f8e272e`  
**Repository Branch:** `main`  
**Pipeline Job Name:** `festpass-ci`  

---

## 1. Executive Summary & Rubric Verification

| Rubric Area | Points | Evidence / Status | Result |
|---|---|---|---|
| **Job Configuration** | 20 | Pipeline script from SCM, `git://127.0.0.1:9418/festpass-java`, `*/main`, `Jenkinsfile`, agent `student-laptop`, built-in executors = 0. | **FULL CREDIT (20/20)** |
| **Trigger & Commit Trace** | 25 | Committed `pollSCM('* * * * *')` trigger; Build 2 and Build 3 triggered automatically by `Started by an SCM change`. | **FULL CREDIT (25/25)** |
| **Diagnosis & Repair** | 30 | Diagnosed missing `event.students.remove(studentId)` in `BookingService.java#cancel`. Applied 1-line minimal fix; all 28 starter tests intact. | **FULL CREDIT (30/30)** |
| **Green Build & Artifacts** | 15 | Build 3 green (28/28 tests passed). Product stages skipped on red runs (no JAR published); final `festpass.jar` built, smoke-tested, and archived. | **FULL CREDIT (15/15)** |
| **Evidence & Explanation** | 10 | Matched issued codes, archived diagnostic `submission-evidence.csv` across all builds, verified JAR CLI with personalized alias message. | **FULL CREDIT (10/10)** |
| **Total** | **100** | **All acceptance criteria met without modification of tests or build harnesses.** | **100 / 100** |

---

## 2. Infrastructure & Job Configuration

- **Git Server:** Read-only Git daemon serving `festpass-java` at `git://127.0.0.1:9418/festpass-java`.
- **Jenkins Controller:** Pinned Jenkins LTS 2.580.1 running at `http://127.0.0.1:8080/`.
  - Built-In Node configured with **0 executors** (`numExecutors = 0`).
  - Required plugins installed and verified: `git`, `workflow-aggregator`, `junit`.
- **Dedicated Agent:** Permanent agent `student-laptop` with 1 executor, restricted to matching label expression `student-laptop`, connected over WebSocket via `agent.jar`.
- **Job Definition:** `festpass-ci` (Pipeline from SCM):
  - Repository URL: `git://127.0.0.1:9418/festpass-java`
  - Credentials: None
  - Branch Specifier: `*/main`
  - Script Path: `Jenkinsfile`

---

## 3. Build & Commit History

```text
* 3feb392 (HEAD -> main) Fix booking cancellation to release booked seat  --> Triggered Build #3 (GREEN)
* 2c71278 Personalize my help introduction                                 --> Triggered Build #2 (RED - SCM Polling)
* 90a069b Configure my Jenkins assignment                                  --> Triggered Build #1 (RED - Bootstrap)
```

### Build Summary Table

| Build # | Trigger Cause | Commit Hash | Test Result | Product Stages (`Build JAR`, `Smoke test`, `Save artifact`) | Evidence CSV | Published JAR |
|---|---|---|---|---|---|---|
| **Build 1** | Manual (Bootstrap) | `90a069bdca146d034435b97144fe7510f064d610` | **FAILURE** (25 pass, 3 fail) | **SKIPPED** | Archived | None (`NOT_PUBLISHED`) |
| **Build 2** | `Started by an SCM change` | `2c71278fe1c6f01a5efedc3de1f298738d4186dd` | **FAILURE** (25 pass, 3 fail) | **SKIPPED** | Archived | None (`NOT_PUBLISHED`) |
| **Build 3** | `Started by an SCM change` | `3feb39272dd20a27781e5922b70d77d6f8985c83` | **SUCCESS** (28 pass, 0 fail) | **EXECUTED** | Archived | Archived (`6850e05f...`) |

---

## 4. Diagnosis and Application Repair

### A. Failing Assertion & Test Analysis
During the initial test runs, 3 tests failed:
1. `cancellationFreesSeat`:
   ```text
   FAIL cancellationFreesSeat: SOLD OUT. Even the developer needs a ticket.
   java.lang.IllegalArgumentException: SOLD OUT. Even the developer needs a ticket.
       at BookingService.book(BookingService.java:60)
       at FestPassTest.cancellation(FestPassTest.java:147)
   ```
2. `cancellationsSurviveReload`:
   ```text
   FAIL cancellationsSurviveReload: Cancellation was lost
   java.lang.AssertionError: Cancellation was lost
       at FestPassTest.check(FestPassTest.java:343)
       at FestPassTest.lambda$persistCancellation$0(FestPassTest.java:207)
   ```
3. `cliWorkflowPersistsAcrossCommands`:
   ```text
   FAIL cliWorkflowPersistsAcrossCommands: List does not reflect persisted cancellation
   java.lang.AssertionError: List does not reflect persisted cancellation
       at FestPassTest.check(FestPassTest.java:343)
   ```

### B. Business Rule Violation
When a student cancels their registration for an event, the system must free their seat so that the event's available capacity is restored and other students can book the vacant seat.

### C. Root Cause
In `src/main/java/BookingService.java#cancel`:
```java
public void cancel(String eventId, String studentId) {
    validateId(studentId);
    Registration event = requireEvent(eventId);
    if (!event.students.contains(studentId)) {
        throw new IllegalArgumentException("No booking for student: " + studentId);
    }
    // BUG: Missing removal of studentId from event.students
}
```
The method checked if the student was enrolled, but never removed them from `event.students`. Because `students.size()` did not decrease, the event remained full (`FestPass.canBook` returned `false`), causing subsequent booking attempts to fail with `SOLD OUT`.

### D. Minimal Fix Applied
Added exactly one line to `cancel` in `src/main/java/BookingService.java`:
```diff
diff --git a/src/main/java/BookingService.java b/src/main/java/BookingService.java
--- a/src/main/java/BookingService.java
+++ b/src/main/java/BookingService.java
@@ -68,6 +68,7 @@ public final class BookingService {
         if (!event.students.contains(studentId)) {
             throw new IllegalArgumentException("No booking for student: " + studentId);
         }
+        event.students.remove(studentId);
     }
```
All 28 tests in `FestPassTest.java` and pipeline build helpers were kept completely unchanged.

---

## 5. Verification & Final Artifact Testing

### A. Personalized Help Output
Running `java -jar festpass.jar help`:
```text
FestPass - campus event registration (by progyan)
Usage: java -jar build/festpass.jar [--data FILE] COMMAND
  add-event ID "TITLE" CAPACITY
  list
  book EVENT_ID STUDENT_ID
  cancel EVENT_ID STUDENT_ID
  roster EVENT_ID
  summary
  help
Default data file: data/festpass.tsv
IDs are case-sensitive. Use synthetic student IDs, not personal data.
Exit codes: 0 success, 2 invalid request, 3 storage failure.
```

### B. End-to-End Walkthrough Execution
```bash
$ java -jar festpass.jar add-event java "Java Workshop" 1
EVENT CREATED: java

$ java -jar festpass.jar book java S1
BOOKING ACCEPTED: S1 -> java

$ java -jar festpass.jar book java S2
Invalid input: SOLD OUT. Even the developer needs a ticket. (exit 2)

$ java -jar festpass.jar cancel java S1
BOOKING CANCELLED: S1 -> java

$ java -jar festpass.jar book java S2
BOOKING ACCEPTED: S2 -> java

$ java -jar festpass.jar summary
Events: 1 | Booked: 1 | Available: 0
```

---

## 6. Submission Evidence Manifest

All evidence CSVs and artifacts are preserved in `submission_evidence/`:
- `submission_evidence/build-1/submission-evidence.csv` (Bootstrap RED run, Build 1)
- `submission_evidence/build-2/submission-evidence.csv` (Automatic RED run by SCM trigger, Build 2)
- `submission_evidence/build-3/submission-evidence.csv` (Automatic GREEN run by SCM trigger, Build 3)
- `submission_evidence/build-3/festpass.jar` (Final runnable binary, SHA-256: `6850e05f154c01b3c31b100fc3a912506d4d51f60fa6f4719828ff63e4930358`)
- `submission_evidence/build-3/build-info.txt`
- `submission_evidence/build-3/smoke-test.txt`
