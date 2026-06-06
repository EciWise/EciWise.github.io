---
layout: default
title: Actors, Roles & Permissions
---

# Actors, Roles & Permissions

Actors represent the external entities that interact with the ECIWise system. Each actor performs specific actions within the platform according to their role and permissions.

## Table of Actors

The following actors interact with the ECIWise web application:

| Role | Description |
|---|---|
| Student | User who registers in the system to access academic support tools such as tutoring, study resources, forums, and gamification. |
| Monitor | Authorized user who offers academic tutoring sessions to students. |
| Administrator | User responsible for system administration, content management, and institutional statistics. |

## Description of the Actors

### Student
A user who registers in the platform to access academic support tools. Students can search and book tutoring sessions, practice with study tools, participate in forums, send messages, join video calls, and receive personalized AI-based recommendations.

### Monitor
A student authorized by the institution to offer academic tutoring sessions. Monitors can publish and manage their availability, conduct tutoring sessions, register attendance and observations, and evaluate student participation.

### Administrator
The administrator manages system-level operations such as user management, authorized monitor management, institutional statistics, question collection management, and audit monitoring.

---

## Roles & Permissions Matrix

| Functionality | Student | Monitor | Administrator |
|---|:---:|:---:|:---:|
| Search available tutoring sessions | x | x | x |
| Book a tutoring session | x | | |
| Cancel a reservation | x | | |
| Reschedule a tutoring session | x | | |
| Rate a tutoring session | x | | |
| View tutoring history | x | x | x |
| View monitor reputation | x | | x |
| Practice with FlashCards | x | | |
| Participate in Kahoot-style quizzes | x | | |
| Send chat messages | x | x | |
| Join a video call | x | x | |
| Create a forum post | x | x | |
| Reply to a forum post | x | x | |
| View leaderboard | x | | |
| Receive personalized recommendations | x | | |
| Publish availability | | x | |
| Modify availability | | x | |
| Cancel availability | | x | |
| View scheduled tutoring sessions | | x | |
| Register attendance | | x | |
| Register observations | | x | |
| Evaluate student participation | | x | |
| View student reputation | | x | x |
| View institutional statistics | | | x |
| Manage users | | | x |
| Generate reports | | | x |
| Upload question collections | | | x |
| Modify question collections | | | x |
| Delete question collections | | | x |
| View system tutoring history | | | x |
| Manage authorized monitors | | | x |
| Send messages to monitors | | | x |

---

### Use Case Student

![Use Case Student](/assets/usecase-student.png)

### Use Case Monitor

![Use Case Monitor](/assets/usecase-monitor.png)

### Use Case Administrator

![Use Case Administrator](/assets/usecase-admin.png)
