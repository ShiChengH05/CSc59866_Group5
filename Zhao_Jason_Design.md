Your Project Design
Author: Jason Zhao
Course: CSC 59866
Selected project and option: Orchestrating Generative AI-based Agents for Personalized and Scalable Computing Education 
Current assignment: 1
Friday commit deadline: Select November 6, 2026, at 11:59 p.m. New York time
Monday presentation: Select November 9, 2026, at 6:30 p.m. New York time
Last updated: 11/6/2026
Document status: Proposed 

Progress note

Currently, I still need to have proposal to be reviewed and approved. 

Contents
1 Problem research and scope
2 Requirements and system specification
3 Alternatives and selected solution
4 Architectural design
5 Component design and interfaces
6 Data structure design
7 Algorithm design and execution
8 Failures and security
9 Implementation and feasibility
10 Testing and verification
11 Operation maintenance and future work
12 Revision history and references
1 Problem research and scope

Orchestrating Generative AI-based Agents for Personalized and Scalable Computing Education 

 

What problem is being investigated? 

Computing education can be difficult to scale because students learn programming at different speeds and struggle with different concepts. In a large introductory programming course, one instructor and a limited number of teaching assistants may not be able to provide immediate, individualized feedback to every student. For example, one student may struggle with understanding loops, while another may understand loops but have difficulty with functions, variables, or debugging. Providing the same explanation to every student does not necessarily address these different learning needs. 

Generative artificial intelligence (AI), particularly large language models (LLMs), could help address this problem by providing students with assistance whenever an instructor or teaching assistant is unavailable. An AI tutor can answer questions, explain programming concepts, identify errors in code, and suggest possible corrections. However, having access to AI does not automatically mean students are learning. For example, a student might ask an AI to write an entire Python function, copy the answer into their assignment, and submit it without understanding how the code works. Although the assignment may be completed successfully, the student might struggle to solve a similar problem independently. 

This creates two related problems: providing students with personalized assistance and determining whether that assistance is actually helping them learn. AI feedback can also be inaccurate, repetitive, or confusing. Beginners may not have enough programming experience to recognize when an AI-generated explanation is incorrect. Therefore, a system needs to do more than simply provide answers. It should support students while also giving instructors useful information about their learning progress and interactions with AI. 

The National Science Foundation (NSF) project, "Orchestrating Generative AI-based Agents for Personalized and Scalable Computing Education," investigates a framework involving teaching-assistant agents, instructor-facing analytics, and an orchestration agent. These components are intended to work together to provide personalized student support while allowing instructors to monitor and adjust the system. My investigation focuses on how these separate components can communicate and coordinate reliably rather than treating the project as one chatbot that answers programming questions. 

In this project, the student interface, tutoring service, code-evaluation service, interaction-analysis service, and instructor dashboard could operate separate services. They must exchange information, handle multiple students using the system simultaneously, and continue functioning when one component is delayed or unavailable. The challenge is to design reliable communication, coordination, data handling, and recovery between these components. 

Source: National Science Foundation, Award 2540315, "Orchestrating Generative AI-based Agents for Personalized and Scalable Computing Education." 

https://www.nsf.gov/awardsearch/show-award/?AWD_ID=2540315 

 

Who and what are affected by it? 

Students are directly affected because the quality and timing of feedback can influence their understanding of programming concepts. An AI tutor could provide helpful explanations and hints, but it could also give incorrect information or complete solutions that students do not fully understand. This creates a risk that students become dependent on AI rather than developing their own problem-solving abilities. 

Instructors and teaching assistants are also affected. In a large class, it can be difficult to determine which students need additional help. Assignment grades do not always explain how students arrived at their answers. Two students might receive the same grade even though one understands the programming concepts while the other relied heavily on AI to complete the assignment. Interaction records could provide additional context about how students approach problems. 

Teaching assistants may benefit if an AI tutor can answer routine questions, allowing them to focus on more complicated problems. However, instructors and teaching assistants would still need to review incorrect feedback and situations where students appear to be struggling. 

Educational institutions must also consider privacy, academic integrity, accessibility, and instructional quality. The system should not automatically label students as cheating or struggling simply because they use AI frequently. Instead, its analytics should provide information that instructors can review before making decisions. 

The distributed architecture affects all these groups because the different services depend on shared information. For example, the tutoring service may need a student's latest code, the evaluation service may return test results, and the analytics service may need both the conversation and evaluation history. If information is delayed, duplicated, or lost, the instructor's dashboard may display incomplete or misleading information. Reliable communication between services is therefore important for providing useful educational support. 

 

What evidence do you currently have that is a real problem? 

Published research suggests that AI feedback can help students in certain situations, but its effectiveness depends heavily on the quality and design of that feedback. 

Qian, Liu, and Zhu's 2026 study, "Exploring the Quality and Effectiveness of AI-Generated Feedback in Introductory Programming," examined AI feedback in an introductory Python course involving 60 undergraduate students across two cohorts. The researchers compared AI-generated feedback with data-driven feedback and examined expert ratings, student perceptions, debugging performance, and examination results. They reported that some AI feedback was repetitive, overly complicated, or misleading. The average expert quality rating was 1.84 out of 5, and more than 40 percent of the feedback received the lowest rating. Students who received AI-generated feedback also performed worse on debugging and final-examination assessments than students who received data-driven feedback. This suggests that simply providing AI-generated assistance does not guarantee better learning outcomes. 

Source: Qian, Yizhou, Meishan Liu, and Liye Zhu. "Exploring the Quality and Effectiveness of AI-Generated Feedback in Introductory Programming." Humanities and Social Sciences Communications, 2026. 

https://www.nature.com/articles/s41599-026-07920-7 

Another study by Heickal and Lan, "A Classroom Study of LLM-Generated Feedback Intervention in Introductory Programming," involved 215 students consenting across 17 programming labs and examined 6,693 programming submissions. The researchers compared natural-language AI hints, AI-generated failing test cases, and assignments without AI feedback. They found that natural-language feedback was associated with higher assignment completion rates and faster progress toward correct solutions. However, the results for generated test cases depended on whether those test cases were valid. This provides a more balanced perspective because it shows that AI can help students, but the type and reliability of the feedback matter. 

Source: Heickal, Hasnain, and Andrew Lan. "A Classroom Study of LLM-Generated Feedback Intervention in Introductory Programming." Proceedings of Machine Learning Research, 2026. 

https://proceedings.mlr.press/v339/heickal26a.html 

A 2026 systematic review by Katona and Katonane Gyonyoru examined 76 studies involving AI in programming education. The review identified potential benefits involving programming performance, motivation, engagement, and problem-solving. However, the results varied depending on students' prior knowledge, assignment types, and system design. It also discussed problems such as inaccurate responses, overreliance on AI, academic-integrity concerns, unequal access, and difficulty verifying generated solutions. 

Together, these studies demonstrate that AI has potential in programming education, but there are still important questions about how it should be implemented and monitored. They support investigating not only whether students use AI, but also what kind of assistance they receive and whether that assistance contributes to actual learning. 

However, these studies do not establish that a distributed architecture will solve educational problems. My engineering question is whether separate services can exchange and preserve student interaction information reliably enough to support instructor monitoring. 

 

What solutions already exist? 

One existing approach is learning analytics. Moodle, an open-source learning management system, provides analytics that use course information, such as participation and assignment activity, to identify patterns and make predictions. Its documentation describes machine-learning models, which learn patterns from historical data, and rule-based models, which apply predefined conditions. These tools can help instructors identify students who may need additional attention. 

However, activity measurements do not necessarily explain why a student is struggling. Two students may submit incorrect programming assignments for completely different reasons. One might misunderstand a programming concept, while another might understand the concept but struggle with the programming environment. Traditional course analytics also does not necessarily interpret conversations between students and AI tutors. 

Source: Moodle, "Using Analytics." 

https://docs.moodle.org/502/en/Using_analytics 

Another existing approach is using general-purpose LLMs as programming assistants. OpenAI's documentation describes function calling and agent orchestration, which allows AI models to use external tools and coordinate specialized agents. For example, one agent could explain a programming concept, another could retrieve course materials, and a separate service could evaluate submitted code. These technologies provide useful building blocks, although their technical capabilities do not prove that a particular arrangement improves student learning. 

Sources: 

OpenAI, "Agents: Orchestration." https://developers.openai.com/api/docs/guides/agents/orchestration 

OpenAI, "Function Calling." https://developers.openai.com/api/docs/guides/function-calling 

These approaches address different parts of the problem. Learning analytics summarizes course activity; programming assistants respond to individual questions, and orchestration tools coordinate different AI models and services. The NSF project combines related capabilities around personalized student assistance and instructor oversight. 

My project will not attempt to recreate the entire NSF framework. Instead, I will examine a smaller distributed implementation focused on monitoring student-AI interactions. 

The proposed design consists of several services. A student-facing application sends questions to a tutoring service, which uses an LLM API to provide hints and explanations rather than immediately returning complete assignment solutions. A separate code-evaluation service can run permitted tests against submitted code and return results. An interaction-analysis service receives conversation and evaluation of events and examines patterns using predefined rules or an LLM. Finally, an instructor's dashboard displays summaries and explains why certain interactions were flagged. 

The services could communicate through APIs and a message queue. After a tutoring exchange, the application could send an event to a queue for background analysis. This allows the student to continue working without waiting for the analytics service to finish processing the interaction. If the analytics service becomes unavailable, the queue can temporarily retain events until the service recovers. Unique event identifiers and safe retry handling can also help prevent duplicate records. 

This design introduces several distributed systems concepts. Concurrency is important because multiple students may use the tutor simultaneously. Fault tolerance matters because one unavailable service should not necessarily stop the entire application. Data consistency is important because the instructor's dashboard should eventually reflect the interactions accepted by the tutoring service. Scalability also matters because individual services may require additional resources as the number of students increases. 

 

What is the most important thing you still do not know? 

The most important uncertainty is whether student-AI interaction records can accurately distinguish productive learning behavior from interactions that may require instructor attention. Two students may ask many questions for completely different reasons. One student might ask follow-up questions because they are trying to understand a programming concept, while another might repeatedly request complete solutions without attempting to understand the explanations. Simply counting the number of questions, a student asks cannot reliably distinguish between these situations. 

Another major uncertainty is how difficult it is to measure whether students are actually learning. Completing an assignment correctly does not necessarily mean a student understands the programming concepts behind the solution, especially when AI assistance is involved. For example, a student could use AI to generate an entire Python program, submit it successfully, and still be unable to write a similar program independently. On the other hand, a student who frequently asks AI questions might be learning effectively by exploring different approaches and asking for explanations. This creates a challenge for the proposed system because AI usage, assignment grades, and completion rates are not necessarily accurate measurements of student understanding. 

I need to investigate what indicators could provide meaningful evidence of learning rather than simply measuring how often students use AI or whether they complete their assignments. Some possible indicators include whether students can correct their own mistakes, explain the reasoning behind their code, apply previously learned concepts to new problems, and solve similar programming exercises without assistance. However, even these indicators have limitations. A student might understand a concept but make a mistake due to a small programming error. Similarly, a student might successfully complete a new problem without fully understanding every concept involved. Therefore, the system should not automatically determine whether a student understands something based on a single interaction or assignment. 

I also do not know how much coordination is needed to make the monitoring system reliable. For example, if the tutoring service successfully answers a student's question, but the analytics service is delayed, the instructor's dashboard might temporarily show an incomplete interaction history. The system needs to preserve information during service interruptions, recover when services become available again, and handle events that are delivered more than once. 

To investigate these uncertainties, I will create simulated introductory Python interactions. These will include students asking for explanations, repeatedly requesting complete solutions, following incorrect suggestions, correcting their own mistakes, and attempting similar problems after receiving feedback. I will compare rule-based analysis with LLM-based analysis to determine how effectively each method identifies different interaction patterns. 

For the educational portion, I will examine indicators such as repeated errors, requests for complete solutions, changes to submitted code, and performance on later similar problems. I will also consider whether students can explain or apply concepts independently. These indicators will help demonstrate the difference between completing an assignment and developing an understanding of the material. However, they should only be treated as evidence for instructor review rather than definitive proof of a student's abilities. 

For the distributed-systems portion, I will test whether events are preserved during an analytics outage, whether retries create duplicate records, and whether multiple interactions can be processed simultaneously. I will compare the events submitted to the system with the records displayed on the instructor's dashboard and measure processing delays. 

One limitation is that simulated data cannot prove that the system accurately identifies real student learning difficulties or improves classroom outcomes. It can only test the analysis methods and system behavior under the scenarios I create. A real classroom evaluation with appropriate privacy protections would eventually be necessary to determine whether the system provides meaningful educational benefits. 

 

Individual Scope 

My individual project will be a small distributed prototype for monitoring potentially ineffective student-AI interactions in an introductory Python course. I will not attempt to build a complete learning platform or train a new language model. Instead, the prototype will contain a student interaction interface, an interaction-recording service, an analysis service, and an instructor's dashboard. A code-evaluation component may also be included in a limited form if time permits. 

I will create fictional student interaction records and compare two different analysis methods. The first method will use rule-based analysis with predefined conditions, such as repeated requests for complete solutions or recurring programming errors. The second method will use an LLM to examine the context of conversations and identify potentially concerning patterns. Comparing these methods will help determine whether a more context-aware approach provides useful information beyond simple predefined rules. 

The system will use Python, a database, an LLM API, and a dashboard. A message broker such as RabbitMQ could be used to pass interaction events between services asynchronously. The instructor's dashboard will display interaction summaries and explain why certain behaviors were flagged. It will not automatically label students as cheating or assume that frequent AI usage means they are struggling. 

My evaluation will examine both educational-analysis behavior and distributed-system behavior. For the analysis portion, I will compare the system's classifications against the categories I assigned to the simulated scenarios. I will examine false positives, where the system flags normal learning behavior, and false negatives, where it fails to identify a potentially concerning interaction. For the distributed-systems portion, I will test concurrent events, delayed processing, service interruptions, retries, and duplicate messages. I will also examine whether all accepted interaction records eventually appear accurately on the dashboard and how long processing takes. 

This project is manageable because it uses simulated data, a limited number of services, and an existing LLM API instead of requiring a completely new AI model. The main contribution is not developing another AI tool that solves programming assignments. Instead, it is investigating how separate computing services can coordinate to record and analyze student-AI interactions reliably while helping instructors better understand how students are using AI during the learning process. 

 ![Alt text for the image]("C:\Users\Jayhe\OneDrive\Pictures\Screenshots\Screenshot 2026-11-06 220500.png")

Assumptions: Conditions you rely on and how to check them.

Constraints: Limits on time, tools, data, and resources.

3 Alternatives and selected solution
Approach and source	Useful behavior	Limitation	Evidence or assumption
Simple baseline	Replace	Replace	Replace
Candidate alternative	Replace	Replace	Replace
Choice and reason: Connect your choice to requirements and constraints.

Decision 1
Context and requirement: Why is a choice needed?
Options: Plausible alternatives.
Decision and reason: What you selected and why.
Consequences: Benefits, costs, and limitations.
Status and revisit condition: Proposed or accepted; what might change it?

Decision 2
Context and requirement: Replace.
Options: Replace.
Decision and reason: Replace.
Consequences: Replace.
Status and revisit condition: Replace.

4 Architectural design
Boundary and outside participants: Replace.

Main parts and messages: Explain responsibilities and labeled arrows.

Where parts run: Identify shared machines and dependencies.

What I build reuse or simulate: Replace.

5 Component design and interfaces
Component	Responsibility	Input	Output or state change
Replace	Replace	Replace	Replace
Critical operation: Name and purpose.

Input: Fields, types, allowed values, caller identity, and example.

Output: Fields, meanings, and example.

Errors: Conditions and appropriate caller actions.

Repeated or overlapping calls: Define the behavior.

6 Data structure design
Record or structure	Fields and example values	Relationships and rules	Persistence
Replace	Replace	Replace	What survives restart?
Ownership: Identify the authoritative place for important facts.

Invalid combinations: Explain how they are prevented or detected.

Retention and deletion: Explain how long data is needed and what deletion changes.

7 Algorithm design and execution
ALGORITHM NAME(inputs)
    replace with input checks
    replace with decisions and state changes
    replace with output and error behavior
Normal sequence: Numbered steps or an explained sequence diagram.

Interrupted sequence: Failure point, surviving state, and recovery.

Overlapping activity: A conflicting execution and the mechanism that handles it.

Resource estimate: A small justified estimate if it affects the choice.

8 Failures and security
Scenario	State that may survive	Expected behavior	Planned check
Replace	Replace	Replace	Replace
Selected adverse case for the prototype: Name the case and why it matters.

Excluded conditions and consequences: Replace.

Possible misuse and affected asset	Response	Remaining limitation and check
Replace	Replace	Replace
9 Implementation and feasibility
Minimal path: The smallest useful end-to-end operation.

Feasibility experiment: Question, procedure, evidence, finding, design consequence.

Current implementation: Distinguish implemented, reused, simulated, blocked, and deferred.

Run instructions and CI: Link README.md and the recorded check.

10 Testing and verification
Test and requirement	Setup and procedure	Expected result	Status and evidence
T1 normal operation	Replace	Replace	Planned until run
T2 correctness rule	Replace	Replace	Planned until run
T3 adverse scenario	Replace	Replace	Planned until run
Baseline and comparison question: Replace.

Workload and environment: Inputs, counts, timing, overlap, versions, machine, and reset.

Measurements: Units and populations, including unsuccessful operations.

Actual results: Explain what happened and what the evidence supports.

Limits and next test: State remaining uncertainty.

11 Operation maintenance and future work
Startup and diagnosis: How to run the system and recognize a problem.

Recovery: Action, preserved state, and verification of recovery.

One future change: Effects on components, interfaces, data, and tests.

Update and rollback: How to change versions and recover a working configuration.

Responsible design: Connect people, data, access, reuse, and resource use to choices.

Risk	Early sign	Response or fallback
Replace	Replace	Replace
Semester II milestone	Dependency	Completion evidence	Essential or optional
Replace	Replace	Replace	Replace
12 Revision history and references
Revision date	Change to my design	Reason and evidence
Replace	Replace	Replace
Open question	Why it matters	Next action
Replace	Replace	Replace
Assistance and reuse: Attribute sources, code, data, diagrams, and assistance under course policy.

References
Supporting media and evidence
Final document check
 Terms and examples make the project understandable to a beginner.
 All twelve sections contain project-specific content.
 Requirements connect to design and tests.
 Architecture, components, data, and algorithms agree.
 Targets and plans are separate from observations.
 Tables, images, pseudocode, references, and evidence links render correctly.
 Images have alternative text, captions, and readable labels.
 Run instructions and evidence support reproduction.
 I am prepared to present my work and orally review the same selected group mate.
 Remaining work and maintenance needs are explicit.
 Instructional placeholders are removed from the final submission.