# B01: Traceable Food Orders Across School Meal Suppliers

## What problem are you investigating?

I am investigating how a school, food supplier, and delivery service could maintain an accurate and explainable record of a food order when each participant reports information separately. The main issue is not simply tracking where food is located. It is determining what the system should report when different participants submit delayed, duplicated, incomplete, or contradictory updates about the same order.

A school food order may involve several stages, including ordering products, preparing a shipment, transporting it, and confirming receipt. Different organizations may record these activities in separate systems. Each participant may have information about only part of the process. For example, a supplier may record how much food it shipped, while the school records how much it received. A delivery service may maintain its own record of the delivery. These records describe related events, but they do not necessarily represent the same measurement or observation.

Consider a hypothetical example in which a supplier reports shipping 100 cartons of food, a delivery service reports delivering 100 cartons, and a school records receiving only 80 cartons. These figures are invented to illustrate the problem because I have not observed this particular discrepancy at an actual school. The three reports could refer to different stages of the delivery process, or they could indicate that further investigation is necessary. Simply accepting the most recently received report would not explain why the quantities differ or establish which information should be trusted.

This problem relates to distributed computing. A distributed system consists of multiple computers or software components that communicate over a network rather than relying entirely on one centralized process. Network communication can experience delays or interruptions, and messages may be processed more than once. Different components can therefore have different views of the same activity at a particular time. These are established technical concerns, although their existence does not prove that school food suppliers frequently experience conflicting delivery records.

My proposed project will model a simplified school food delivery process. Each update could contain an event identifier, order identifier, reporting participant, event type, event time, time received by the system, and reported quantity. The event time describes when the participant says an event occurred, while the received time records when the software obtained that report. Keeping both timestamps would allow me to test situations in which an older event arrives after a newer one.

I plan to compare two approaches. The first, a latest-received-status baseline, would use the most recently received update as the current status. This is a deliberately simple experimental baseline, not a claim that existing school procurement systems work this way. The second, an evidence-based reconciliation approach, would preserve the event history, identify repeated event identifiers, check the order of expected workflow events, and flag quantity disagreements or missing receiving confirmations.

My research question is: **Can an evidence-based reconciliation rule produce more accurate and explainable order statuses than a latest-received-status rule when distributed order updates are delayed, duplicated, incomplete, or contradictory?**

The purpose is to evaluate a specific decision-making rule for interpreting order records. I am not claiming to invent supply-chain tracking, reliable messaging, or event histories. Instead, I want to investigate whether a more careful way of interpreting conflicting reports improves the results in a defined school food delivery scenario.

## Who and what are affected?

The groups potentially affected include school staff responsible for receiving food, suppliers responsible for fulfilling orders, delivery personnel responsible for transporting goods, and the software systems used to record their actions.

School staff need to know whether an order has arrived and whether the quantity received matches the expected quantity. If a system reports an order as complete when the recorded quantities disagree, staff may need to spend additional time checking documents, contacting the supplier, or investigating the discrepancy. If an order remains marked as incomplete after it has been received, staff may also have to resolve an unnecessary administrative issue. These are plausible consequences of inaccurate records; I have not yet obtained operational data demonstrating how often they occur in school meal programs.

Suppliers and delivery services also depend on accurate records. A supplier may record an order as shipped, while the delivery service records a delivery attempt or completion. The school’s receiving record may provide another perspective. Each record describes an event from a particular participant’s point of view, and the participants may not immediately agree on the final status. A system that overwrites earlier updates with the newest one could hide useful evidence about how the disagreement occurred.

From a distributed-systems perspective, the affected parties depend on information exchanged among independent components. The challenge is not necessarily that one participant is reporting false information. The participants may have different observations, different timestamps, or different definitions of completion. A shipment record, for instance, is not automatically proof that the school received the entire shipment.

The project will model these separate perspectives rather than assume that every participant has identical information. It will also distinguish between a message being successfully processed and a real-world delivery being confirmed. A message acknowledgment can indicate that software received or handled a message according to a particular protocol, but it does not independently prove that the physical goods arrived.

The intended beneficiaries of a more reliable reconciliation process would be people who need to review an order’s status and understand why the system reached that conclusion. However, this project will not claim that the proposed approach will reduce school food waste, prevent shortages, or save money unless further research provides evidence for those outcomes. Its immediate focus is to determine whether the system can reconstruct and explain an order’s history more accurately under controlled test conditions.

## What evidence do you currently have that this is a real problem?

At this stage, I have sources establishing that school food procurement is a real activity and that distributed messaging and supply-chain traceability are established technical areas. However, I do not yet have direct evidence establishing how frequently school meal suppliers experience duplicate updates, delayed messages, or conflicting delivery records. Evidence that a supply chain exists does not automatically prove that the specific software problem I am investigating is common or that a new solution is necessary.

**Documented evidence: school food procurement.** Sourcewell, a public-sector cooperative purchasing organization, has published a procurement opportunity titled “Food Products and Distribution for School Food Authorities with Related Supplies and Services.” The opportunity requests proposals for food products and distribution for eligible participating organizations. This establishes a real procurement context involving school food products and distribution. However, the opportunity does not establish that schools have experienced duplicated messages, inaccurate delivery histories, or conflicting quantities. It provides context for the proposed project, not proof that the specific failure I describe has occurred.

Source: https://proportal.sourcewell-mn.gov/Module/Tenders/en/Tender/Detail/7f28ce4f-318d-466e-b261-c2094f72634a

**Documented evidence: distributed messaging reliability.** RabbitMQ is an existing message broker that allows software components to exchange messages. Its reliability documentation explains that network or system failures can interrupt communication and that messages may need to be retransmitted. It also explains that consumers must be prepared to handle messages they have received previously. RabbitMQ provides mechanisms, including publisher confirms and consumer acknowledgments, to help applications manage message delivery and processing.

These sources demonstrate that duplicate processing and uncertain message delivery are recognized concerns in distributed systems. They do not demonstrate that RabbitMQ itself has failed in a school food supply chain. In fact, RabbitMQ already provides mechanisms to help developers handle these reliability concerns. My project is not intended to repair RabbitMQ or claim that it cannot handle duplicate messages. Instead, I will investigate a separate question: how an application should interpret order records when different participants report conflicting information, even if the messages themselves are delivered successfully.

Sources:  
https://www.rabbitmq.com/docs/reliability  
https://www.rabbitmq.com/docs/confirms

**Documented evidence: supply-chain traceability.** GS1’s Electronic Product Code Information Services (EPCIS) is an established standard for capturing and sharing supply-chain event information within and across organizations. It provides a common way to represent information about events, including the status, location, movement, and chain of custody of products. This demonstrates that supply-chain event recording and information sharing are already established technical approaches. It also means that I cannot claim that creating an event history across organizations is a new idea.

However, the existence of EPCIS does not prove that every organization uses it or that it resolves every disagreement about the physical quantities delivered. My project will investigate a narrow reconciliation experiment rather than claim to replace an established traceability standard.

Source: https://www.gs1.org/standards/epcis

**Observed evidence:** I have not yet collected real school delivery records, observed a delivery-reconciliation process, or conducted experiments showing that the proposed failure occurs in practice.

**Reported evidence:** The technical documentation describes known messaging reliability challenges and existing traceability capabilities. I have not yet collected interviews or firsthand reports from school staff, suppliers, or delivery personnel about the frequency or impact of conflicting order records.

**Hypothesized problem:** If independent participants submit delayed, duplicated, incomplete, or contradictory updates, a system that relies only on the latest received status may produce a misleading order history. This is a hypothesis that I will test with simulated data, not an established finding about school food distribution.

The next step in establishing the real-world significance of the problem is to investigate existing research, documented operational incidents, and, if feasible, firsthand accounts from people involved in school food procurement and receiving.

## What solutions already exist?

Several existing approaches address parts of this problem. My project needs to distinguish those established approaches from the narrower question it will test.

One possible approach is manual reconciliation. In general, a person can compare an order, a supplier’s shipment record, a delivery document, and a school’s receiving record to investigate a disagreement. This approach does not require a new distributed system, and it may be sufficient for a small number of orders. However, it depends on the records being available and understandable. I have not established which manual reconciliation procedures a particular school uses, so I will treat this as a general approach rather than a documented description of school-specific practice.

A second approach is reliable messaging. RabbitMQ provides mechanisms such as acknowledgments and publisher confirms to help applications manage message delivery and processing. Applications can also be designed to handle duplicate messages safely. These mechanisms are relevant if order updates are sent between separate software components. Nevertheless, reliable message delivery and accurate order interpretation are different problems. A system can successfully deliver a message reporting 100 cartons while the school’s receiving record reports 80. Messaging reliability alone does not determine which quantity reflects what physically occurred.

A third approach is supply-chain event standardization through GS1 EPCIS. EPCIS supports a shared structure for representing and exchanging events across different organizations and systems. This can help separate systems communicate using more consistent information. However, standardizing event formats does not automatically guarantee that the original reports are correct, nor does a common event format alone determine how an application should resolve every contradictory report. The standard is an existing solution that my project should acknowledge rather than attempt to replace.

My proposed experiment will focus on the interpretation of records after they are available to the application. The latest-received-status baseline will select the most recently received update as the current status. The evidence-based approach will retain the full event history, ignore repeated event identifiers when they represent duplicate transmissions, check expected workflow steps, and flag disagreements that cannot be resolved from the available records.

The evidence-based approach will not automatically declare that one participant is correct simply because its report arrived later. For example, a supplier’s shipment quantity and a school’s receiving quantity refer to different stages of the process. If those quantities differ, the software should preserve the reports and mark the discrepancy for review instead of silently treating one value as the truth. Similarly, a missing receiving confirmation should not be interpreted as proof that a delivery failed; it should be represented as an unresolved state.

I plan to evaluate both approaches using the same simulated input cases. The comparison will measure whether each approach identifies the expected final status, handles duplicate updates without incorrectly repeating their effects, detects quantity conflicts, preserves an understandable history, and processes the test data successfully. If time permits, I may also test how the system behaves after a restart.

This project does not assume that the evidence-based approach will win in every category. It may detect more conflicts while leaving more orders unresolved. That could be an appropriate result if the available evidence is insufficient to support a definitive status. The aim is to compare the trade-offs rather than claim that one approach is universally better.

## What is the most important thing you still do not know?

The most important thing I still do not know is whether evidence-based reconciliation can produce a more accurate and explainable order status than a latest-received-status rule when updates are delayed, duplicated, incomplete, or contradictory.

This question has both a technical and a practical dimension. Technically, the system must decide how to process messages whose arrival order differs from their event timestamps. It must avoid applying the same event repeatedly, preserve an audit history, and handle disagreements without silently discarding evidence. Practically, it must determine what information is sufficient to mark an order as shipped, delivered, partially received, or unresolved. A delivery report alone may not establish that the receiving organization confirmed the full quantity.

To investigate this question, I plan to create a controlled dataset of invented order events. The test cases could include a normal order, a repeated transmission, a delayed update, an out-of-order update, a quantity disagreement, and an order with no receiving confirmation. I would also like to test a correction to an earlier record to determine whether the system can preserve the original report while recording the correction.

I plan to evaluate the rules using several measures: whether duplicate transmissions are handled correctly; whether the final status agrees with the expected result for each test case; whether quantity conflicts are identified; whether the system preserves an understandable event history; and how long each rule takes to process the test data. If the implementation supports persistent storage, I may also test whether the stored history remains available after the application restarts.

The expected results are not known in advance. Evidence-based reconciliation may perform better on conflict detection and explanation, but it could also leave more orders unresolved because it refuses to infer a conclusion without sufficient evidence. That may be appropriate in some situations, but it could be inconvenient if the rules are too strict. The experiment must therefore evaluate both incorrect conclusions and unresolved cases.

Another important unknown is whether this is a sufficiently important problem in actual school food supply chains to justify further development. The technical experiment can demonstrate behavior under simulated conditions, but it cannot establish how common the problem is in real operations. That would require further research into existing systems, documented incidents, and possibly interviews with people involved in school food procurement and receiving.

## Individual Scope

My individual project will focus on a small prototype representing one school, one supplier, one delivery service, and one simplified food-order workflow. I plan to use invented data to represent order creation, shipment reporting, delivery reporting, and receiving confirmation. I have not finalized the implementation technologies yet, so the following tools are possible options rather than fixed requirements.

One possible approach is to use Python to implement the two reconciliation rules and run the test cases. SQLite could provide a simple way to store event records and preserve their history. If I find that demonstrating message exchange between separate software components is feasible within the project timeline, I may consider adding RabbitMQ. RabbitMQ could simulate updates being passed between components, but it may not be necessary for the first version of the prototype. I will decide on the final technology choices based on the project requirements, feasibility, and the time available.

The essential deliverable will be a controlled comparison between the latest-received-status baseline and the evidence-based reconciliation rule. I plan to generate the test cases, define their expected outcomes, implement both approaches, run the same inputs through each, and report the results using the measures described above. I will document the assumptions and explain cases where the available evidence does not support a definitive status.

The project will not connect to a real school, supplier, or delivery company’s system. It will not use private student information or actual procurement records. It will not implement a complete commercial ordering platform, optimize delivery routes, manage payments, or replace GS1 EPCIS. If RabbitMQ is included, its role will be to support a limited experiment involving communication between software components. Using it locally would not, by itself, prove the reliability of a production system deployed across multiple organizations.

The expected contribution is a small, measurable evaluation of how two status-reconciliation rules respond to known types of inconsistent event data. It is not a claim that existing supply-chain systems lack traceability or that the proposed method is entirely new. By limiting the project to a specific workflow and controlled tests, I can investigate a concrete distributed-systems question while keeping the implementation achievable for an individual project.

Ultimately, the project will examine whether preserving event history and explicitly flagging uncertainty improves the quality of order-status decisions in simulated scenarios. A positive result would support further investigation, not prove that the same benefits would automatically occur in real school procurement. A mixed or negative result would still be useful because it would show the limitations of the proposed rule and identify where additional information or a different reconciliation method may be needed.
