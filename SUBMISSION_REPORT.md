# EventPass – CI/CD Assignment Report

**Submitting Student:** Sambhav Jain  
**Class Code:** 001  
**Challenge Code:** 00620  

**Project:** EventPass – Campus Event Management System

**Original Project Attribution:** Adapted from the FestPass CI project by its original author, Progyan.

**Evidence Note:** The historical Jenkins build records below are retained from the original project and are not represented as builds performed by the submitting student.

---

## 1. Job Configuration
- **Job Name:** `festpass-ci` (Pipeline from SCM)
- **Repository URL:** `git://127.0.0.1:9418/festpass-java`
- **Branch:** `*/main`
- **Script Path:** `Jenkinsfile`
- **Agent:** `student-laptop` (1 executor, exclusive usage)
- **Built-in Node:** 0 executors
- **Polling Trigger:** `pollSCM('* * * * *')` added to `Jenkinsfile`

---

## 2. Build History

| Build | Trigger Cause | Commit | Result | Tests Passed / Total | Product Stages | Published Artifacts |
|---|---|---|---|---|---|---|
| **#1** | Manual (Bootstrap) | `90a069b` | FAILURE | 25 / 28 | Skipped | `submission-evidence.csv` |
| **#2** | SCM Change (Polling) | `2c71278` | FAILURE | 25 / 28 | Skipped | `submission-evidence.csv` |
| **#3** | SCM Change (Polling) | `3feb392` | SUCCESS | 28 / 28 | Executed | `festpass.jar`, `build-info.txt`, `smoke-test.txt`, `submission-evidence.csv` |

---

## 3. Bug Diagnosis and Fix

### Failing Assertion
In Build #1 and #2, the test suite failed with:
```text
FAIL cancellationFreesSeat: SOLD OUT. Even the developer needs a ticket.
java.lang.IllegalArgumentException: SOLD OUT. Even the developer needs a ticket.
    at BookingService.book(BookingService.java:60)
    at FestPassTest.cancellation(FestPassTest.java:147)
```

### Cause
In `src/main/java/BookingService.java`, the `cancel(String eventId, String studentId)` method verified that the student was registered for the event, but never removed them from `event.students`. Because the roster count was never decremented, subsequent booking requests failed with `SOLD OUT` even after cancellation.

### Fix
Added the missing removal call in `src/main/java/BookingService.java` at line 71:
```java
event.students.remove(studentId);
```
No test files, build scripts, or pipeline stages were modified.

---

## 4. Verification

### Help Command Output
```text
$ java -jar festpass.jar help
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

### Walkthrough Run
```text
$ java -jar festpass.jar add-event java "Java Workshop" 1
EVENT CREATED: java

$ java -jar festpass.jar book java S1
BOOKING ACCEPTED: S1 -> java

$ java -jar festpass.jar book java S2
Invalid input: SOLD OUT. Even the developer needs a ticket.

$ java -jar festpass.jar cancel java S1
BOOKING CANCELLED: S1 -> java

$ java -jar festpass.jar book java S2
BOOKING ACCEPTED: S2 -> java

$ java -jar festpass.jar summary
Events: 1 | Booked: 1 | Available: 0
```

---

## 5. Evidence Files
- Red Build #1: [`evidence/build-1/submission-evidence.csv`](evidence/build-1/submission-evidence.csv)
- Red Build #2: [`evidence/build-2/submission-evidence.csv`](evidence/build-2/submission-evidence.csv)
- Green Build #3: [`evidence/build-3/submission-evidence.csv`](evidence/build-3/submission-evidence.csv)
- Final Executable: [`evidence/build-3/festpass.jar`](evidence/build-3/festpass.jar)
- Smoke Test Output: [`evidence/build-3/smoke-test.txt`](evidence/build-3/smoke-test.txt)
