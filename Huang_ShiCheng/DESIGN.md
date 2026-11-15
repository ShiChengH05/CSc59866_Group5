# Autoscaling for Resource Usage (CPU and memory)

**Author:** Shi Cheng Huang  
**Course:** CSC 59866  
**Selected project and option:** Build or improve a distributed system tool (option 1) - a predictive autoscaling for Kubernetes on resource usage to improve existing HPA before threshold is breached.     
**Current assignment:** 2  
**Friday commit deadline:** November 13 at 11:59 p.m. New York time  
**Monday presentation:** November 16 at 6:30 p.m. New York time  
**Last updated:** 11/13/26  
**Document status:** ready for review

## Progress note

<!-- Write 100â€“150 words: changes, learning, main open issue, feedback addressed,
and next step. Link evidence when available. Update at each Friday deliverable. -->
Section 1 is revised by adding more evidence from Cheekuri and Jeong to support the reactive method problem related to autoscaling resource usage. Section 2 defines four functional requirements (F1–F4), one correctness rule (C1), one quality target (Q1), given an ordinary example and unsuccessful example, assumptions with checks, and constraints. Section 3 compares three approaches: reactive HPA (baseline), Kairos operator, AROF, with a source for each in a table format. The comparison table shows whether each approach meets the requirements (F1-F4 and Q1). The final choice and decision record will be made in Assignment 3. The main open question remains whether the prediction interval reduces SLO violations enough to justify its complexity. The next step is Architecture and component interfaces. (11/13/26)

## Contents

- [1 Problem research and scope](#1-problem-research-and-scope)
- [2 Requirements and system specification](#2-requirements-and-system-specification)
- [3 Alternatives and selected solution](#3-alternatives-and-selected-solution)
- [4 Architectural design](#4-architectural-design)
- [5 Component design and interfaces](#5-component-design-and-interfaces)
- [6 Data structure design](#6-data-structure-design)
- [7 Algorithm design and execution](#7-algorithm-design-and-execution)
- [8 Failures and security](#8-failures-and-security)
- [9 Implementation and feasibility](#9-implementation-and-feasibility)
- [10 Testing and verification](#10-testing-and-verification)
- [11 Operation maintenance and future work](#11-operation-maintenance-and-future-work)
- [12 Revision history and references](#12-revision-history-and-references)

## 1 Problem research and scope

<!-- Assignment 1. Describe a person, task, difficulty, and consequence using
a concrete scenario. Label assumptions. Distinguish a larger research project's
aims from your own contribution where relevant. -->

**Problem and affected people:** The problem that I am investigating is the reactive autoscaling for resource usage, which means giving computer resources including CPU (central processing unit that processes instructions), memory (which stores active data) only after a threshold is crossed while working on a workload. As a reactive method, the system watches resource usage and only adds resources when CPU or memory crosses a threshold. This is inefficient when many different workloads must be managed, which leads to consequences such as slowing down the process and performance degradation within distributed computing systems.  
### The affected people are:
| Stakeholders | Type | Responsibility/Need | Source link |
|:---|:---|:---|:---|
| Cloud operators, platform engineers, and the organization | Internal | Responsible for the cluster and for allocating resources to avoid SLO violations | https://trepo.tuni.fi/handle/10024/238698#1 |
| Users and clients | External | Access the service and need fast responses without degradation | https://www.jisem-journal.com/index.php/journal/article/view/13473/6334 |

SLO is a target for service performance and a violation occurs when performance fails to meet that target within a given time window.

This problem matters because it costs money and hurts users' experience when workloads receive the wrong amount of resources. The problem occurs under conditions where resource demand changes so quickly that the system is unable to react, especially with unpredictable workloads. When demand for a workload is steady, the reactive method can function properly, but this is not always the case when demand is bursty.

**Current workflow:** To understand why this problem matters, consider a concrete example. A web service normally receives 100 requests per second. At a certain time, the traffic to the web service jumps to 500 requests per second within 30 seconds. The Horizontal Pod Autoscaler (HPA), a Kubernetes (a system for automating deployment and scaling of containerized applications) component that automatically adds or removes application instances when resource usage rises above or falls below a target threshold, checks metrics every 15–30 seconds. Thirty seconds later, it detects CPU above 80%, and 15 seconds after that, it starts scaling. Each new instance takes 60–120 seconds to start, after which new capacity is available. During the waiting period, existing instances are overloaded and lack resources. Response times increase, and some requests time out. 

![Diagram showing the HPA control loop, where HPA checks metrics every 15–30 seconds, decides whether the threshold is breached, and adds replicas.](assets/workflow.png)     
Figure 1. The HPA control loop checks metrics every 15–30 seconds and adds replicas when the threshold is breached.

### Conditions and difficulty:
- Under normal conditions, reactive autoscaling works when resources are sufficient. 
- It fails when resources are insufficient (under-provisioning), causing slowdowns and timeouts.
- It wastes money when resources are excessive (over-provisioning), even though the service itself continues to function. 

Therefore, the right number of resources is required for service-level objective. 

**Evidence:** 
- Reactive autoscaling is structurally lagging, with a three-to-five-minute timing period spent waiting to scale resources after the threshold is breached. During this window, it leads to tail-latency degradation (the slowest performance) and service-level objective (SLO) violations. [Documented] This claim is documented by the thesis of Tran, Duy Anh. 
- Conventional autoscaling, which is a reactive method, scales workloads based on the resource utilization of CPU and memory, which can lead to a failure to anticipate the burst demand of these workloads. This means that with the reactive method, the resources used for certain workloads cannot handle the burst demand for immediate changes in resources to achieve better performance. [Documented] This claim is documented by the abstract by Karthik Chakravarthy Cheekuri.
- With sudden load spikes, Horizontal Pod Autoscaler will trigger the threshold and wait for the new resource, which can cause cold starts (the delay before a new instance is ready to serve requests), temporary under-provisioning, and frequent SLO violations. [Documented] This claim is documented by the poster abstract by Hyunsu Jeong and others.

**Six-week contribution:** By the end of six weeks, I will demonstrate the prototype with its architecture: 
- The system design on how the Random Forest forecasters with the two decision policies (point prediction and prediction interval), with simulator connects.
- A working prototype that can run on a small subset of the Alibaba 2021 trace.
- Initial forecast output to show that the pipeline functions.

**Semester II scope:** In semester II, I will add to the complete project with 
- full implementation using codes
- running simulations to test two designs policies (point prediction and prediction interval)
- measure the data on the count of SLO violations and additional resources allocated for each design.   

**Exclusions:** I will not build a production-ready system, modify the Kubernetes control plane, address GPU scheduling, train deep learning models, or consider I/O.

<!-- Add an actual image using a relative path, for example:
![Description of the workflow](assets/workflow.png)

Figure 1. Explain what the workflow shows and why it matters.
Do not leave references to nonexistent images. -->

## 2 Requirements and system specification

<!-- Assignment 2. Explain required behavior before implementation methods.
Start with 3â€“5 functions, a correctness rule, and a quality target.
Complete design and test references during Assignments 2 and 3. -->

| ID | Required behavior and conditions | Acceptance check | Design and test reference |
|---|---|---|---|
| F1 | Predict future CPU and memory demand to scale resources before the threshold is crossed. | Given a trace where either resource (CPU or memory) rises above the threshold, check whether the system outputs a forecast and makes a scaling decision before the breach. | Test T1 (normal operation) |
| F2 | Support two decision policies: scale on the point prediction (single number), and scale on the prediction interval (the upper bound of the range). | Both policies will be run on the same trace and produce scaling decisions that can be compared. | Test T2 (policy comparison) |
| F3 | Measure for each policy: SLO violation rate and additional resource cost. | The output reports both numbers for both policies on the same workload. | Test T3 (measurement) |
| F4 | Reject old metric data, such as a sample that is 60 seconds old | A stale sample is rejected, and the system waits for fresh data | Test T4 (stale data) |
| C1 | Use only data from before the prediction time, to avoid data leakage from the future. | Training on the full trace, so the model sees future data which lead to data leakage for testing. | Test T5 (correctness rule) |
| Q1 | Reduce SLO violation rate compared with reactive HPA. Target is proposed until measured. Workload: Alibaba 2021 trace. | Run both policies, measure their SLO violation rate and cost. Label as proposed until results exist. | Mark as target until measured |

**Inputs and outputs:**
- Ordinary example: The system receives a time series of CPU and memory usage. It outputs a forecast (when CPU will be 75%) and a decision (scale up by 2 replicas).
- Unsuccessful example: The system receives a metric sample that is 60 seconds old. It rejects this data and waits for fresh data, instead of deciding on stale information.

**Assumptions:**
| Assumptions | How to check it |
|---|---|
| Kubernetes, HPA, and metrics-server are running | Check that metrics-server and HPA are active in the cluster|
| Prediction will reduce the SLO violation rate compared with reactive HPA | Compare violation rates on the same trace|
| The Alibaba 2021 trace represents bursty workloads | Check the trace's workload description |
| The simulator reproduces the 60-120 second startup time to act like real Kubernetes | Run one test scaling event and time it |

**Constraints:** 
- Time: One student with two semesters to work on. 
- Tools: Python, scikit-learn, Random Forest model, KEMU or KWOK. No production cluster (real).
- Data: Alibaba 2021 trace. No live cluster data.
- Resources: The simulator runs on a laptop, which is not a cloud platform.

## 3 Alternatives and selected solution

<!-- Draft in Assignment 2; complete in Assignment 3. Compare at least two
approaches under the same requirements, including a simple baseline. -->

| Approach | Useful behavior | Limitation | Evidence or assumption |
|---|---|---|---|
| Simple baseline: Reactive HPA | Built into Kubernetes, works for steady workloads | Scales only after the threshold is crossed, with 3-5 minutes lag and causes SLO violations | https://trepo.tuni.fi/handle/10024/238698#1 |
| Kairos operator | Predicts future CPU and memory usage and adjusts resource allocation, using AI, before demand arrives. Allows multiple scaling modes, avoids conflicts with other Kubernetes autoscaling tools. | Needs an external AI service; relies on historical patterns; does not report an SLO violation rate. | https://operatorhub.io/operator/kairos-operator |
| AROF (Adaptive Resource Optimization Framework) | Combines hybrid classification (clustering and tagging) and multi-horizon LSTM forecasting with cost optimization. It uses multi-horizon forecasting, which produces predictions over multiple future time horizons. | Complex model; LSTM increases training cost; long-term predictions are uncertain. Reports SLA violations, instead of SLO violations (which this project requires) | https://www.tandfonline.com/doi/full/10.1080/17445760.2025.2605532?scroll=top&needAccess=true |

LSTM stands for Long Short-Term Memory, a type of neural network that learns patterns in sequences over time.   
SLA stands for Service Level Agreement, a contractual commitment to a customer.

### Comparison Table
| Approach | F1 | F2 | F3 | F4 | Q1 |
|---|---|---|---|---|---|
| Simple baseline: Reactive HPA | No | No | Partial (provides raw metrics, not SLO rate or cost) | No | No |
| Kairos operator | Yes | No (focuses on multiple modes) | No (reports cost estimation without SLO violation rate) | Not documented | Not measured |
| AROF (Adaptive Resource Optimization Framework) | Yes | No (focuses on cost optimization) | No (reports SLA, not SLO) | Not documented | Not measured |

**Requirement key:**
- **F1:** Forecast before the breach
- **F2:** Two decision policies (point vs. interval)
- **F3:** Measure SLO violation rate and cost
- **F4:** Reject stale data
- **Q1:** Reduce SLO violation rate compared with reactive HPA

**Choice and reason:** Connect your choice to requirements and constraints.

### Decision 1

**Context and requirement:** Why is a choice needed?  
**Options:** Plausible alternatives.  
**Decision and reason:** What you selected and why.  
**Consequences:** Benefits, costs, and limitations.  
**Status and revisit condition:** Proposed or accepted; what might change it?

### Decision 2

**Context and requirement:** Replace.  
**Options:** Replace.  
**Decision and reason:** Replace.  
**Consequences:** Replace.  
**Status and revisit condition:** Replace.

## 4 Architectural design

<!-- Assignment 3. Show context and the main running programs and stores.
Embed a diagram from assets and retain editable source. Mermaid is optional. -->
Work to come; they are not due yet.

**Boundary and outside participants:** Replace.

**Main parts and messages:** Explain responsibilities and labeled arrows.

**Where parts run:** Identify shared machines and dependencies.

**What I build reuse or simulate:** Replace.

## 5 Component design and interfaces

<!-- Assignment 3. A component can be a function or module in one program. -->
Work to come; they are not due yet.

| Component | Responsibility | Input | Output or state change |
|---|---|---|---|
| Replace | Replace | Replace | Replace |

**Critical operation:** Name and purpose.

**Input:** Fields, types, allowed values, caller identity, and example.

**Output:** Fields, meanings, and example.

**Errors:** Conditions and appropriate caller actions.

**Repeated or overlapping calls:** Define the behavior.

## 6 Data structure design
Work to come; they are not due yet.

<!-- Assignment 4. Explain information and meaning, not only storage syntax. -->

| Record or structure | Fields and example values | Relationships and rules | Persistence |
|---|---|---|---|
| Replace | Replace | Replace | What survives restart? |

**Ownership:** Identify the authoritative place for important facts.

**Invalid combinations:** Explain how they are prevented or detected.

**Retention and deletion:** Explain how long data is needed and what deletion changes.

## 7 Algorithm design and execution
Work to come; they are not due yet.

<!-- Assignment 4. Replace this block with your pseudocode. Explain any step
that hides a guarantee, such as a protected write. -->

```text
ALGORITHM NAME(inputs)
    replace with input checks
    replace with decisions and state changes
    replace with output and error behavior
```

**Normal sequence:** Numbered steps or an explained sequence diagram.

**Interrupted sequence:** Failure point, surviving state, and recovery.

**Overlapping activity:** A conflicting execution and the mechanism that handles it.

**Resource estimate:** A small justified estimate if it affects the choice.

## 8 Failures and security
Work to come; they are not due yet.

<!-- Assignment 4. Include three adverse scenarios and two misuse cases. -->

| Scenario | State that may survive | Expected behavior | Planned check |
|---|---|---|---|
| Replace | Replace | Replace | Replace |

**Selected adverse case for the prototype:** Name the case and why it matters.

**Excluded conditions and consequences:** Replace.

| Possible misuse and affected asset | Response | Remaining limitation and check |
|---|---|---|
| Replace | Replace | Replace |

## 9 Implementation and feasibility
Work to come; they are not due yet.

<!-- Begin the feasibility experiment and path in Assignment 3; refine in Assignment 4;
report implementation status in Assignments 5â€“6. -->

**Minimal path:** The smallest useful end-to-end operation.

**Feasibility experiment:** Question, procedure, evidence, finding, design consequence.

**Current implementation:** Distinguish implemented, reused, simulated, blocked, and deferred.

**Run instructions and CI:** Link README.md and the recorded check.

## 10 Testing and verification
Work to come; they are not due yet.

<!-- Plan in Assignment 4; add observations in Assignments 5â€“6. Expected outcomes
are not results. Link raw evidence. -->

| Test and requirement | Setup and procedure | Expected result | Status and evidence |
|---|---|---|---|
| T1 normal operation | Replace | Replace | Planned until run |
| T2 correctness rule | Replace | Replace | Planned until run |
| T3 adverse scenario | Replace | Replace | Planned until run |

**Baseline and comparison question:** Replace.

**Workload and environment:** Inputs, counts, timing, overlap, versions, machine, and reset.

**Measurements:** Units and populations, including unsuccessful operations.

**Actual results:** Explain what happened and what the evidence supports.

**Limits and next test:** State remaining uncertainty.

## 11 Operation maintenance and future work
Work to come; they are not due yet.

<!-- Draft in Assignment 4; add operating instructions in Assignment 5;
complete in Assignment 6. -->

**Startup and diagnosis:** How to run the system and recognize a problem.

**Recovery:** Action, preserved state, and verification of recovery.

**One future change:** Effects on components, interfaces, data, and tests.

**Update and rollback:** How to change versions and recover a working configuration.

**Responsible design:** Connect people, data, access, reuse, and resource use to choices.

| Risk | Early sign | Response or fallback |
|---|---|---|
| Replace | Replace | Replace |

| Semester II milestone | Dependency | Completion evidence | Essential or optional |
|---|---|---|---|
| Replace | Replace | Replace | Replace |

## 12 Revision history and references

| Revision date | Change to my design | Reason and evidence |
|---|---|---|
| 11/13/2026 | Add more evidences to show the reactive problem exists in the autoscaling for resource usage (in section 1) | See others have multiple evidences provided |

| Open question | Why it matters | Next action |
|---|---|---|
| Will the prediction interval reduce SLO violations enough, at an acceptable resource cost, to justify its added complexity? This remains to be tested. | It determines whether the interval policy is used in the final design | Run the simulation on the Alibaba 2021 trace and compare SLO violation rates with additional resource used|

**Assistance and reuse:** Attribute sources, code, data, diagrams, and assistance under course policy.

### References

<!-- Use descriptive links with author/organization, title, date/version, and
the claim supported. Also cite sources near relevant explanations. -->
Tran, Duy Anh. "Predictive autoscaling for kubernetes using machine learning workload forecasting", Tampere University, June 2026, https://trepo.tuni.fi/handle/10024/238698#1  
Karthik Chakravarthy Cheekuri. "SLO-First Autoscaling for Multi-Tenant Microservices: A Control-Theoretic Approach to P95/P99 Latency", Journal of Information Systems Engineering and Management, https://www.jisem-journal.com/index.php/journal/article/view/13473/6334  
Jeong et al. "SLO-Aware and Cost-Efficient Predictive Autoscaling for Kubernetes Microservices", Jeju National University, https://www.sc-asia.org/2026/data/poster/post240.pdf#1#1

Kairos Operator. OperatorHub, https://operatorhub.io/operator/kairos-operator   
Patel, Ravi, and Ashwin Makwana. "AROF: adaptive resource optimization framework for Kubernetes cluster using workload forecasting", International Journal of Parallel, Emergent and Distributed Systems, 2025, https://www.tandfonline.com/doi/full/10.1080/17445760.2025.2605532

### Supporting media and evidence

<!-- Link actual images, editable diagram files, logs, data, and optional video.
Explain each item's purpose. Avoid missing files and private-only paths. -->
- Figure 1 (Section 1): workflow diagram of the HPA control loop. The image is stored at `assets/workflow.png` and shows how HPA checks metrics every 15–30 seconds, detects a threshold breach, and adds replicas that take 60–120 seconds to start. This illustrates the three-to-five-minute lag that the project addresses.

### Final document check

- [ ] Terms and examples make the project understandable to a beginner.
- [ ] All twelve sections contain project-specific content.
- [ ] Requirements connect to design and tests.
- [ ] Architecture, components, data, and algorithms agree.
- [ ] Targets and plans are separate from observations.
- [ ] Tables, images, pseudocode, references, and evidence links render correctly.
- [ ] Images have alternative text, captions, and readable labels.
- [ ] Run instructions and evidence support reproduction.
- [ ] I am prepared to present my work and orally review the same selected group mate.
- [ ] Remaining work and maintenance needs are explicit.
- [ ] Instructional placeholders are removed from the final submission.