Shi Cheng Huang 

CSc 59866 

Kurt Brown 

# Problem Background 

The problem related to data science — predicAng future values to make beDer decisions 

with using historical data in distributed compuAng — is the topic of resource usage predicAon for autoscaling. Resource usage predicAon for autoscaling is used to predict future CPU, 

memory, and I/O uAlizaAon so resources can be scaled proacAvely, rather than reacAng aNer a failure occurs. If we give a workload too liDle, it slows down or crashes. If we give it too much, 

we waste resources and money. If we can predict the future demand for resources, then we can give each workload the right amount at the right Ame and will prevent failure due to resource has threshold from happening. 

Cloud operators and plaOorm engineers who run applicaAons on distributed systems like Kubernetes (automates the scaling and management of containerized applicaAons) experience this problem. They manage large numbers of workloads that each require CPU, memory, and I/O resources. They want to give each workload the right amount of resources — too liDle leads to slowdowns and crashes, while too much leads to wasted cost and capacity. It is currently done with a reacAve method, in which the system watches resource usage and only adds resources when CPU or memory crosses a threshold. This is inefficient when many different 

workloads must be managed. The limitaAon is that by the Ame the system reacts, the problem has already occurred. This problem maDers because it costs money and hurts users' experience 

when workloads receive the wrong amount of resources needed. The problem occurs under 

condiAons where resource demand changes so quickly that the system is unable to react, 

especially with unpredictable workloads. 

ReacAve autoscaling is structurally lagging, with a three-to-five-minute Aming period 

spent waiAng to scale resources aNer the threshold is breached. During this window, it leads to tail-latency degradaAon (the slowest performance) and service-level objecAve (SLO) violaAons. This claim is supported by Tran, Duy Anh as documented source. ConvenAonal autoscaling, which is a reacAve method, scales workloads based on the resource uAlizaAon of CPU and memory, which can lead to a failure to anAcipate the burst demand of these workloads. This means that with the reacAve method, the resources used for certain workloads cannot handle the burst demand for immediate changes in resources to achieve beDer performance. This claim is supported by Karthik Chakravarthy Cheekuri as documented source. With sudden load spikes, Horizontal Pod Autoscaler will trigger the threshold and wait for the new resource, which can cause cold starts, temporary under-provisioning, and frequent SLO violaAons. This claim is 

supported by Hyunsu Jeong and others as documented source. 

|Approach|How it addresses the<br>problem|Strength|LimitaAons/QuesAons|
|---|---|---|---|
|Approach A (Kairos)|Use AI models and<br>Ame-series for<br>forecasAng to adjust<br>resource allocaAon.|Intelligent resource<br>allocaAon, mulAple<br>safe scaling mode,<br>and conflict<br>avoidance.|AI dependency and<br>forecast accuracy due<br>to historical data.|
|Approach B (AROF)|Combine hybrid<br>classificaAon<br>(clustering and|MulA-horizon<br>forecasAng to have|Model complexity<br>with mulA-horizon, as<br>well with uncertainty|



|tagging) and mulA-|many predicAons|especially for a long|
|---|---|---|
|horizon LSTM for|over Ame.|Ame into the future.|
|forecasAng with<br>opAmizaAon.|||



The project currently appears closer to OpAon 1 because it is made to improve a distributed system tool by predicAng the resource allocaAon for each workload for more efficient method than the reacAve method, which costs more money and Ame with low performance especially the waiAng Ame window for autoscaling. This is distributed scheduler that decides when and where to run the workload and how many resources they receive, which fits the resource usage predicAon for autoscaling. The project currently appears to involve original design with element of the selecAon design. This is an original design because the task is to build a predicAve autoscaling soluAon, which is existed in network (selecAon design) with like Ame-series model. Distributed compuAng might be important for this problem because the allocaAon of the resources for the workloads is not just only in a single machine sedng, it has resources that are shared across many machines, services and workloads. 

How many workloads are sAll using reacAve autoscaling method for machine resources? 

Answer: search the report that how many workloads are using reacAve and others are proacAve 

methods. Does a model work well in historical trace sAll work well in current workloads? 

Answer: measure the accuracy with the resource allocaAon over Ame for the workloads. How 

do we know is the predicAon of the number of the autoscaling can be trust? Answer: we can 

have both point predicAon and predicAon interval models to count the total number of SLO 

violaAon, and to compare the actual resource required and the predictable resource used. 

The factors that can make the project difficult are difficulty evaluaAng the result, access 

to a realisAc Kubernetes environment, and lack of required data. For difficulty evaluaAng the 

result, we don’t have a controlled experimental environment to prove that predicAve 

autoscaling has an improvement over reacAve autoscaling. For access to a realisAc Kubernetes environment, we are unable to run the experiment without access to a realisAc Kubernetes environment. For lack of required data, since we can only get historical data to train the model, so it might not be appropriate to use in the live workload. 

The discussion of selecAon design changed how I view this project because with using 

the right model that has exist as open-source in the network for my experiment on the resource usage predicAon for autoscaling. The discussion of the principle of independent funcAonality (also called separaAon of concerns) can be used for separaAng predicAon from decision making, and to test each component of the resource independently. The discussion of the risk 

management changed how I view this project because with current sedng and environment, it will be difficult to implement certain parts of the experiments. With how to control the risk that occurs during the experiment. 

Reference: 

Tran, Duy Anh. “PredicAve autoscaling for kubernetes using machine learning workload forecasAng”, Trepo, hDps://trepo.tuni.fi/handle/10024/238698#1 

Karthik Chakravarthy Cheekuri. “SLO-First Autoscaling for MulA-Tenant Microservices: A ControlTheoreAc Approach to P95/P99 Latency”, Journal of InformaAon Systems Engineering and Management, <u>hDps://www.jisemjournal.com/index.php/journal/arAcle/view/13473/6334</u> 

Jeong et al. “SLO-Aware and Cost-Efficient PredicAve Autoscaling for Kubernetes Microservices”, Jeju NaAonal University, <u>hDps://www.sc-asia.org/2026/data/poster/post240.pdf#1#1</u> 

