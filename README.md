# Ramy Maarouf | 041-269-337 | CST8917 | Assignment 1: Serverless Computing - Critical Analysis | February 10, 2026

---

## Part 1: Paper Summary

The central thesis of Hellerstein et al.'s paper, "Serverless Computing: One Step Forward, Two Steps Back," is that while Function-as-a-Service (FaaS) successfully delivers on the promise of autoscaling and simplified administration (the "one step forward"), it simultaneously regresses in its support for data-centric and distributed computing (the "two steps back"). The authors argue that by strictly decoupling compute from storage and enforcing isolation, current serverless platforms make it "crazy expensive/slow" to work with data at scale and nearly impossible to build complex distributed systems.

### Key Limitations Identified

* **Execution Time Constraints**: FaaS functions have limited lifetimes (often 5–15 minutes), requiring awkward workarounds for long-running tasks like machine learning training or complex ETL processes.
* **Communication & Network Limitations**: Functions lack direct addressability; they cannot talk to each other directly and must communicate via slow, high-latency storage intermediaries.
* **The Anti-Pattern (Data-Shipping)**: Current architectures move data to the code rather than code to the data, which is an architectural anti-pattern that leads to high latency and cost for data-heavy apps.
* **Limited Hardware Access**: There is no mechanism to access specialized hardware like GPUs or programmable frameworks, which is critical for modern performance-heavy workloads.
* **Distributed Computing Challenges**: Implementing simple protocols like leader election is prohibitively expensive and slow on FaaS due to the lack of shared state and low-latency synchronization.

### Proposed Future Directions

The authors propose that future cloud programming should include fluid code and data placement, heterogeneous hardware support, and new languages that encourage correct working in small units of both data and computation.

---

## Part 2: Azure Durable Functions Deep Dive

| Topic | Research and Connection to Criticisms |
| :--- | :--- |
| **Orchestration model** | Durable Functions introduces a tripartite model: Client functions (trigger the workflow), Orchestrator functions (manage the sequence and logic), and Activity functions (perform the actual work). Unlike basic FaaS, where functions are triggered independently, the orchestrator acts as a "conductor," ensuring that multiple functions execute in a specific order while maintaining the overall context of the workflow. This model addresses the paper's criticism regarding the difficulty of coordinating distributed tasks by providing a code-first way to manage dependencies. Abstracting the complex state-passing logic, it allows developers to focus on the business sequence rather than the infrastructure plumbing required for multi-step interactions.  |
| **State management** | Durable Functions manages state using an Event Sourcing pattern, where the current state is rebuilt by replaying a sequence of recorded events from an append-only store (Azure Storage). This addresses the paper's criticism that functions are "painfully" stateless by providing a durable, persistent history of the workflow. Behind the scenes, the extension manages checkpoints and restarts for you, so local state isn't lost when a virtual machine reboots or the process recycles. This effectively turns ephemeral functions into "durable" actors that can maintain variables and progress over extremely long periods, solving the coordination challenges identified by Hellerstein et al.  |
| **Execution timeouts** | Orchestrator functions effectively bypass regular timeout limits by "sleeping" (dehydrating) whenever they wait for an activity to finish or a timer to expire. While the activity functions themselves are still subject to the standard function timeouts (e.g., 10 minutes on Consumption), the orchestrator can manage a workflow that spans days, weeks, or even months. This directly addresses the paper's criticism of "execution time constraints" by allowing long-running business processes to exist within a serverless environment. The orchestrator yields control back to the dispatcher during waits, ensuring it doesn't consume billable compute time while idle, which maintains the core value of serverless while removing its lifetime limitations.  |
| **Communication between functions** | Orchestrator and activity functions communicate via internal queues in the Task Hub, typically backed by Azure Storage. While this still involves a storage intermediary, it automates the "slow storage" problem identified in the paper by handling state passing and result retrieval transparently. The orchestrator doesn't need to manually write to a database to pass data to the next step; it simply yields the task, and the framework handles the serialization and messaging. While this doesn't provide the "direct point-to-point" addressability the authors desired, it mitigates the developer burden of manual I/O management and ensures reliable, fault-tolerant message delivery between distributed components.  |
| **Parallel execution (fan-out/fan-in)** | The fan-out/fan-in pattern allows an orchestrator to trigger multiple activity functions simultaneously and then "fan-in" to aggregate the results once they are all finished. This addresses the concern about distributed computing by providing a resilient, managed way to coordinate massive parallel workloads across many instances. In traditional FaaS, fanning in is difficult because you need an external "coordinator" to know when all parallel branches are done; Durable Functions handles this state tracking natively. This allows for efficient MapReduce-style operations within a serverless framework, reducing the time span of large workloads by executing sub-tasks in parallel rather than sequentially.  |

---

## Part 3: Critical Evaluation

### 1. Limitations that remain unresolved

Despite its improvements, Azure Durable Functions still struggles with two major Hellerstein criticisms:

* **The Data Shipping Penalty**: Durable Functions remains a data-shipping architecture. Data must still be serialized and sent to storage (the Task Hub) between every function call, which remains slow and expensive for massive datasets compared to colocated compute.
* **No Direct Point-to-Point Communication**: Functions still cannot address each other directly. They rely on queues and blobs for all coordination, maintaining the high latency "storage as a blackboard" bottleneck that the authors specifically critiqued as an anti-pattern for distributed systems.

### 2. Your Verdict

In my opinion, Azure Durable Functions represents a massive leap in developer productivity but only an incremental step in system physics. It successfully "hides" the limitations of serverless (statelessness and timeouts) behind a brilliant orchestration layer. However, it does not solve the fundamental I/O and networking inefficiencies identified by Hellerstein et al. It is a successful workaround that works within the current cloud's limitations rather than truly revolutionizing the code-data relationship the authors envisioned.

---

## References

* Hellerstein, J. M., et al. (2019). Serverless Computing: One Step Forward, Two Steps Back. [https://www.cidrdb.org/cidr2019/papers/p119-hellerstein-cidr19.pdf](https://www.cidrdb.org/cidr2019/papers/p119-hellerstein-cidr19.pdf)
* Microsoft Learn. (2025). Durable Functions Overview. [https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-overview](https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-overview)
* Burckhardt, S., et al. (2022). Netherite: Efficient Execution of Serverless Workflows. [https://www.vldb.org/pvldb/vol15/p1591-burckhardt.pdf](https://www.vldb.org/pvldb/vol15/p1591-burckhardt.pdf)

---

## AI Disclosure Statement

I used Gemini to assist with this assignment in the following ways:

* **Drafting**: Assisted in expanding the technical deep-dive sections to meet the required word count while maintaining clarity.
* **Research**: Helped identify specific Microsoft documentation links for Durable Functions performance targets and state replay behaviour.
* **Verification**: Used AI to ensure the "fan-out/fan-in" explanation correctly reflected the behavior of the Durable Task Framework.
