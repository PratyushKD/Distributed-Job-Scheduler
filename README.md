# Distributed Task Scheduler

A fault-tolerant distributed task scheduler built with **Java, Apache ZooKeeper, Apache Curator, and Dropwizard**. The system coordinates multiple worker instances, elects a leader, assigns submitted jobs to workers, tracks execution status, and handles worker or leader failures.

> **Implementation note:** This repository is a reference-based implementation derived from the original project/article. The original attribution and licensing requirements should be retained.

## Overview

The scheduler follows a leader-based coordination model:

- Clients submit jobs through a REST API.
- Worker instances register with ZooKeeper.
- ZooKeeper coordinates worker membership and leader election.
- The elected leader assigns jobs to available workers.
- Workers execute jobs asynchronously and update their status.
- Worker failures are detected through ZooKeeper and affected jobs can be reassigned.

The project demonstrates practical distributed-systems concepts including **leader election, service coordination, ephemeral nodes, watchers, task assignment, failure detection, and at-least-once execution**.

## Key Features

- **Distributed task scheduling** across multiple worker instances
- **Leader election** using ZooKeeper
- **Worker registration and monitoring**
- **Task assignment** using configurable worker-selection strategies
- **Round-robin and random worker selection**
- **Failure detection and task reassignment**
- **At-least-once task execution**
- **REST API** for client/task interaction
- **Asynchronous task execution** using worker thread pools
- **ZooKeeper watchers** for coordination and state changes

## System Architecture

```text
                         ┌─────────────────┐
                         │      Client     │
                         │    REST API     │
                         └────────┬────────┘
                                  │
                                  ▼
                     ┌────────────────────────┐
                     │       ZooKeeper        │
                     │                        │
                     │  Leader Election       │
                     │  Worker Registry       │
                     │  Job Coordination      │
                     │  Failure Detection     │
                     └───────────┬────────────┘
                                 │
                         Leader Worker
                                 │
              ┌──────────────────┼──────────────────┐
              ▼                  ▼                  ▼
       ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
       │   Worker 1  │    │   Worker 2  │    │   Worker N  │
       │             │    │             │    │             │
       │ Execute Job │    │ Execute Job │    │ Execute Job │
       └─────────────┘    └─────────────┘    └─────────────┘
Task Execution Flow
1. Client submits a job
          │
          ▼
2. Job is registered in ZooKeeper
          │
          ▼
3. Leader monitors available jobs
          │
          ▼
4. Leader selects a worker
          │
          ▼
5. Assignment is created for that worker
          │
          ▼
6. Worker detects the assignment
          │
          ▼
7. Worker executes the job
          │
          ▼
8. Worker updates job status
ZooKeeper Coordination

The scheduler uses ZooKeeper as the coordination layer.

Conceptually, the system uses paths such as:

/jobs
    └── {job-id}

/assignments
    └── {worker-id}
          └── {job-id}

/status
    └── {job-id}

/workers
    └── {worker-id}

ZooKeeper is responsible for coordinating:

Worker membership
Leader election
Job registration
Job assignment
Worker failure detection
Task status updates

The project uses CuratorFramework to simplify ZooKeeper operations such as node management, leader election, and watcher handling.

Worker Selection

The scheduler provides an extensible WorkerPickerStrategy interface.

Currently implemented strategies include:

Random Worker

Selects an available worker randomly.

Round Robin Worker

Cycles through available workers to distribute assignments across workers.

The strategy-based design allows additional worker-selection policies to be added without changing the core scheduling flow.

Failure Handling
Worker Failure

Worker membership is tracked through ZooKeeper. When a worker disappears, the scheduler can detect the failure and reassign affected work to another available worker.

Leader Failure

If the current leader fails, ZooKeeper's leader-election mechanism allows another worker to become the leader and continue coordinating task assignment.

Task Reassignment

The failure-recovery flow is designed to prevent tasks from being permanently lost when a worker becomes unavailable.

Project Structure
src/
├── main/
│   ├── java/
│   │   └── com.umar.taskscheduler/
│   │       ├── App.java
│   │       ├── AppConfiguration.java
│   │       ├── JobDetail.java
│   │       │
│   │       ├── callbacks/
│   │       │   ├── AssignmentListener.java
│   │       │   ├── JobAssigner.java
│   │       │   ├── JobsListener.java
│   │       │   └── WorkersListener.java
│   │       │
│   │       ├── core/
│   │       │   └── ZKDao.java
│   │       │
│   │       ├── module/
│   │       │   └── GuiceModule.java
│   │       │
│   │       ├── resources/
│   │       │   ├── Client.java
│   │       │   ├── Job.java
│   │       │   └── Worker.java
│   │       │
│   │       ├── service/
│   │       │   ├── ClientService.java
│   │       │   └── WorkerService.java
│   │       │
│   │       ├── strategy/
│   │       │   ├── RandomWorker.java
│   │       │   ├── RoundRobinWorker.java
│   │       │   └── WorkerPickerStrategy.java
│   │       │
│   │       └── util/
│   │           └── ZKUtils.java
│   │
│   └── resources/
│       └── log4j.properties
│
└── test/
    └── java/
        ├── AppTest.java
        └── RoundRobinWorkerTest.java
Core Components
Component	Responsibility
App.java	Application entry point
AppConfiguration.java	Application configuration
ClientService	Handles client requests and job submission
WorkerService	Handles worker-side execution
JobAssigner	Assigns jobs to workers
JobsListener	Detects newly submitted jobs
WorkersListener	Monitors worker availability
AssignmentListener	Handles worker assignments
ZKDao	Provides ZooKeeper data-access operations
WorkerPickerStrategy	Interface for worker-selection strategies
RandomWorker	Random worker selection
RoundRobinWorker	Round-robin worker selection
ZKUtils	ZooKeeper paths and utility operations
Low-Level Execution Model

The main scheduling path is:

ClientService
     │
     │ submit job
     ▼
 ZooKeeper
   /jobs
     │
     ▼
JobsListener
     │
     ▼
JobAssigner
     │
     │ choose worker
     ▼
ZooKeeper
/assignments/{worker-id}
     │
     ▼
WorkerService
     │
     │ execute
     ▼
ZooKeeper
/status/{job-id}

Jobs are serialized and stored in ZooKeeper, where the relevant listeners and services react to changes in the coordination state.

Technologies
Java 17
Apache ZooKeeper
Apache Curator
Dropwizard
JUnit
Lombok
Maven
Prerequisites

Install/configure:

Java 17
Maven
Apache ZooKeeper 3.5.4 or later
ZooKeeper running on port 2181

The original implementation uses ZooKeeper TTL/extended node functionality, so the ZooKeeper configuration must enable the required extended types support.

Running the Project
1. Start ZooKeeper

Configure ZooKeeper with a client port such as:

clientPort = 2181

For installations requiring extended types, enable:

-Dzookeeper.extendedTypesEnabled=true

Then start the ZooKeeper server.

2. Build the Project

From the project root:

mvn clean install
3. Run the Application

The main application class is:

com.umar.taskscheduler.App

The application can be started with:

server local.yml

Multiple application instances can be started with different configured ports to simulate multiple workers/nodes.

Testing

The project contains tests for core application behavior and worker-selection logic, including the round-robin strategy.

Run the test suite with:

mvn test
Distributed Systems Concepts Demonstrated

This project provides practical exposure to:

Distributed coordination
Leader election
Worker registration
Ephemeral ZooKeeper nodes
ZooKeeper watchers
Task scheduling
Task state management
Failure detection
Task reassignment
At-least-once execution
Strategy-based worker selection
Asynchronous task execution
Future Improvements

Potential extensions include:

Task prioritization
Job dependencies
More sophisticated worker-selection/load-balancing strategies
Docker-based task isolation
Prometheus/Grafana metrics
Improved failure recovery and retry policies
Persistent task history
Authentication and authorization for the REST API
Attribution

This repository is based on the original Distributed Task Scheduler implementation and accompanying article by Snehasish Roy.
