## Distributed Task Scheduler

A fault-tolerant and scalable distributed task scheduling system that dynamically assigns tasks to worker nodes. The system uses Apache ZooKeeper for worker coordination, leader election, task assignment, and failure detection. It supports multiple worker-selection strategies and provides mechanisms for recovering from worker failures.

## Tech Stack Used
Java 17
Apache ZooKeeper
Apache Curator
Dropwizard
Maven
JUnit
Lombok

## How It Works

The Distributed Task Scheduler follows a leader-based architecture in which Apache ZooKeeper acts as the central coordination layer between clients and worker nodes. A client submits a task through the REST API, after which the task is serialized and stored in ZooKeeper under the /jobs/{job-id} path. The scheduler monitors this job registry and the elected leader is responsible for assigning newly submitted jobs to available workers. Worker selection is handled through the WorkerPickerStrategy interface, with implementations such as RandomWorker and RoundRobinWorker providing different assignment strategies.

Once a worker is selected, the job is placed under /assignments/{worker-id}/{job-id}. The corresponding worker monitors its assignment path, retrieves the task, deserializes it, and executes it using the worker service. After execution, the worker updates the task status under /status/{job-id}. ZooKeeper watchers allow the different components to respond to changes in jobs, assignments, and worker availability without requiring direct communication between every component.

The system also incorporates failure handling through ZooKeeper's coordination mechanisms. Worker nodes are monitored using ZooKeeper, allowing failures to be detected when a worker becomes unavailable. The leader can then identify incomplete tasks associated with the failed worker and reassign them to another available worker. Similarly, if the current leader fails, ZooKeeper can initiate a new leader-election process so that another scheduler instance can continue coordinating task assignment. This architecture provides distributed task execution with an at-least-once execution model while allowing additional workers and worker-selection strategies to be introduced as the system scales.

## Code Flow

The code starts from `App.java`, which acts as the entry point and initializes the application and its required configuration. The client interacts with the scheduler through the REST resources, with `Client.java` and `Job.java` handling task-related requests. These requests are passed to `ClientService`, where the submitted job is prepared and serialized before being stored in ZooKeeper through `ZKDao`.

Once a job is placed under the `/jobs` path, `JobsListener` monitors ZooKeeper for the newly created job. The job is then passed to `JobAssigner`, which obtains the available workers and uses the `WorkerPickerStrategy` to select one. Depending on the configured strategy, either `RandomWorker` or `RoundRobinWorker` determines the worker that receives the job. The assignment is then created under the corresponding worker's assignment path.

On the worker side, `WorkerService` and `AssignmentListener` monitor the worker's assignment path for new tasks. When an assignment is detected, the worker retrieves and deserializes the job, executes it, and updates its status in ZooKeeper. `WorkersListener` continuously monitors worker availability, allowing the system to react when a worker becomes unavailable. ZooKeeper also handles the coordination required for leader election and failure detection.

Therefore, the overall code flow can be viewed as: **Client → REST Resource → ClientService → ZooKeeper → JobsListener → JobAssigner → Worker Selection Strategy → WorkerService → Task Execution → Status Update**. This separation of responsibilities allows task submission, scheduling, worker selection, execution, and failure handling to remain as independent components within the scheduler.


## Future Scope
Task prioritization based on client-defined rules
Docker-based task execution
Prometheus and Grafana integration for monitoring
Improved fault detection and worker recovery
Support for job dependencies

## Attribution

This project is based on the original Distributed Task Scheduler implementation and tutorial by Snehasish Roy.

Original tutorial: https://snehasishroy.com/build-a-distributed-task-scheduler-using-zookeeper

The original implementation, architecture, and design concepts have been retained and adapted for this repository.
