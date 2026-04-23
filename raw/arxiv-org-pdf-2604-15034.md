---
title: "Autogenesis: A Self-Evolving Agent Protocol"
source: https://arxiv.org/pdf/2604.15034
date: 2026-04-23
fetched: 2026-04-23
type: paper
status: compiled
fetch_method: pdftotext
compiled_to:
  - wiki/self-evolving-agents.md
  - wiki/autoresearch-and-self-improvement.md
compiled_date: 2026-04-23
---

# Autogenesis: A Self-Evolving Agent Protocol

**Source:** https://arxiv.org/pdf/2604.15034

---

Autogenesis: A Self-Evolving Agent Protocol

arXiv:2604.15034v2 [cs.AI] 21 Apr 2026

Wentao Zhang 1 * Zhe Zhao 2 * Haibin Wen 3 * Yingcheng Wu 2 Ming Yin 4  Bo An 1  Mengdi Wang 4 

Abstract
Recent advances in LLM based agent systems have shown promise in tackling complex, long horizon tasks. However, existing agent protocols (e.g., A2A and MCP) under specify cross entity lifecycle and context management, version tracking, and evolution safe update interfaces, which encourages monolithic compositions and brittle glue code. We introduce AUTOGENESIS PROTOCOL (AGP), a self evolution protocol that decouples what evolves from how evolution occurs. Its Resource Substrate Protocol Layer (RSPL) models prompts, agents, tools, environments, and memory as protocol registered resources1 with explicit state, lifecycle, and versioned interfaces. Its Self Evolution Protocol Layer (SEPL) specifies a closed loop operator interface for proposing, assessing, and committing improvements with auditable lineage and rollback. Building on AGP, we present AUTOGENESIS SYSTEM (AGS), a self-evolving multi-agent system that dynamically instantiates, retrieves, and refines protocolregistered resources during execution. We evaluate AGS on multiple challenging benchmarks that require long horizon planning and tool use across heterogeneous resources. The results demonstrate consistent improvements over strong baselines, supporting the effectiveness of agent resource management and closed loop self evolution. The code is available at https://github.com/ DVampire/Autogenesis.
*Equal contribution. Corresponding authors. 1Nanyang Technological University, Singapore 2Stanford University, Stanford, CA, USA 3City University of Hong Kong, Hong Kong, China 4Princeton University, Princeton, NJ, USA. Correspondence to: Ming Yin <mingyin0312@gmail.com>, Bo An <boan@ntu.edu.sg>, Mengdi Wang <mengdiw@princeton.edu>.
1Unless otherwise specified, resources refer to instances of the five RSPL entity types: prompt, agent, tool, environment, memory with agent outputs.

1. Introduction
Recent advances in LLM-based agent systems have demonstrated significant potential in tackling complex, longhorizon tasks (Yao et al., 2022; Wei et al., 2022; Brown et al., 2020). However, static agent designs often prove insufficient when facing the diversity and stochasticity of real-world environments. To overcome this limitation, endowing agents with self-evolution capabilities--enabling them to automatically adjust strategies, refine instructions, and update tools based on environmental feedback--has emerged as a critical avenue for achieving robust autonomy. This transition from predefined execution to dynamic adaptation represents a fundamental shift in agentic system design.
Despite the growing interest in self-evolving agents, implementations remain largely fragmented and ad hoc. Existing systems often lack shared standards, rendering the evolution process neither composable nor auditable. Developers are frequently forced to rely on brittle glue code, leading to monolithic architectures that are difficult to maintain. Furthermore, without explicit lifecycle management and safe update interfaces, self-modification introduces significant risks of runtime instability. To address these issues, it is necessary to elevate development from ad hoc engineering practices to the protocol level, decoupling "what evolves" from "how evolution occurs" via a standardized framework to ensure modular, traceable, and safe evolution.
While protocols such as Anthropic's Model Context Protocol (MCP) (Anthropic, 2025b) and Google's Agent-toAgent (A2A) (Google, 2025) have standardized connectivity, applying them directly to self-evolution scenarios presents a conceptual mismatch. These protocols are primarily designed to resolve connectivity challenges--specifically, model-tool invocation (MCP) or inter-agent communication (A2A). However, the core of self-evolution lies not in invocation, but in state mutation and management.
Existing connectivity protocols lack native support for entity Lifecycle and Version Lineage. In a closed-loop evolutionary system, if the creation, update, and destruction of components are not precisely defined, the optimizer cannot safely apply modifications. Moreover, the absence of version tracking and rollback mechanisms means that erroneous updates can lead to irrecoverable errors. Consequently, re-

1

Autogenesis: A Self-Evolving Agent Protocol

lying solely on communication protocols is insufficient; a novel protocol capable of managing the dynamics of mutation is required.
To bridge the gap from connectivity to evolution, a specialized protocol must address three essential problems:
� Decoupling: Resources such as prompts, tools, and memory must be abstracted from the agent's core logic, transforming them into passive, independently managed entities rather than tightly coupled code blocks.
� Safety & Auditability: Strict version control and rollback mechanisms must be introduced to ensure that every evolutionary step is traceable and reversible.
� Formalism: A set of standardized operators (e.g., reflect, propose, verify) needs to be defined to strictly govern the evolution process, converting heuristic text modifications into a rigorous control loop.
To address these challenges, we introduce AUTOGENESIS. Far from being merely a utility library, AUTOGENESIS is a two-layer protocol architecture designed to strictly decouple the evolutionary substrate from the evolutionary logic. Our core motivation is to standardize underlying resource representations, enabling the same optimization algorithms (Yuksekgonul et al., 2025; Shao et al., 2024; Hu, 2025b) to be seamlessly applied across diverse agent components.
� Layer 1: Resource Substrate Protocol Layer (RSPL). This layer defines the substrate of evolution, modeling Prompts, Agents, Tools, Environments, and Memory as Protocol-registered Resources. RSPL endows these resources with explicit state, lifecycle, and versioned interfaces, rendering them standardized objects amenable to observation and manipulation.
� Layer 2: Self-Evolution Protocol Layer (SEPL). This layer establishes a closed-loop operator interface grounded in control theory. It defines atomic operations--Reflect, Select, Improve, Evaluate, and Commit--to formally execute the evolution cycle, ensuring that every self-modification is documented and adheres to strict safety constraints.
Building on this protocol, we present AUTOGENESISAGENT, a reasoning-and-acting tool-calling agent. Instead of relying on hard-coded components, it dynamically instantiates, retrieves, and refines resources via protocol interfaces during execution. We evaluated this system on multiple challenging benchmarks, including GPQA (Rein et al., 2024), AIME, GAIA (Mialon et al., 2023), and LeetCode (LeetCode). The results demonstrate that by leveraging standardized resource management and closed-loop evolution, AUTOGENESIS-AGENT consistently achieves significant improvements over strong baselines.
The significance of this work extends beyond performance gains; it illustrates a potential shift from manual prompt en-

gineering to automated protocol engineering. By equipping agents with standardized self-repair and evolution capabilities, AUTOGENESIS provides a foundational paradigm for building next-generation agent systems capable of sustained autonomous adaptation in complex environments.
2. Related Work
2.1. LLM-based Agent Systems and Tool Use
Recent progress in large language model (LLM) based agent systems has demonstrated their ability to address complex, long-horizon tasks that require multi-step reasoning and external tool interaction (Rein et al., 2024; Mialon et al., 2023; Yao et al., 2022; Wei et al., 2022; Schick et al., 2023). In these systems, LLMs typically serve as centralized decisionmaking modules that interpret observations, decompose tasks, and invoke tools to affect the environment(Yao et al., 2022; Wei et al., 2022). Benchmarks such as GAIA (Mialon et al., 2023) have further highlighted the importance of structured tool use and planning capabilities in agent design.
Most existing agent frameworks adopt architectures in which prompts, tools, and memory are embedded as tightly coupled internal components. Tools are commonly treated as fixed functional modules that are manually curated and integrated into the agent pipeline (Qin et al., 2023; Schick et al., 2023; Chen et al., 2021).. While effective for bounded tasks, this design limits systematic reuse and controlled adaptation of tools as task requirements evolve. In contrast, our approach models tools (including native scripts, MCP tools (Anthropic, 2025a), and agent skills (Anthropic, 2025b)) as protocol-registered resources with explicit interfaces and state representations, enabling dynamic instantiation and controlled refinement during execution.
2.2. Connectivity and Interoperability Protocols
As agent-based systems grow in scale and complexity, several protocol-level efforts have emerged to standardize model�tool interaction and inter-agent communication. Anthropic's Model Context Protocol (MCP) (Anthropic, 2025a) provides a unified interface for connecting language models to external tools and data sources. Similarly, Google's Agent-to-Agent (A2A) protocol aims to standardize communication primitives that support collaboration among multiple agents.
These protocols primarily address interoperability at the level of invocation and message passing. They specify how agents and tools interact, but largely leave the internal state of agents and resources opaque. In particular, they do not define mechanisms for managing resource lifecycles, tracking version lineage, or constraining state mutations over time. As a result, while connectivity protocols simplify integration, they do not directly support the persistent state

2

Autogenesis: A Self-Evolving Agent Protocol

evolution required by self-modifying agent systems.
2.3. Self-Correction and Optimization Mechanisms
A parallel line of work investigates mechanisms that enable agents to improve their performance through self-correction and optimization. Methods such as TextGrad (Yuksekgonul et al., 2025) interpret natural language feedback as a signal analogous to gradients, enabling iterative updates to string-valued components such as prompts (Pryzant et al., 2023; Zhou et al., 2022). . Reinforcement learning based approaches have also been applied to agent improvement. Techniques including Reinforce++ (Hu, 2025a) and GRPO (Shao et al., 2024) frame agent components as policies and use evaluation signals as rewards to guide optimization (Shinn et al., 2023; Madaan et al., 2023; Zelikman et al., 2022).
While these methods demonstrate that agent behaviors can be iteratively improved, they are typically applied within narrowly scoped settings and lack a shared abstraction for managing heterogeneous agent components. Updates are often applied directly to prompts or policies without explicit lifecycle control, version tracking, or rollback support. AUTOGENESIS provides a protocol-level abstraction that accommodates these optimization strategies by exposing agent components as standardized, evolvable resources and defining operator-level interfaces through which different optimization methods can be applied in a controlled manner.
2.4. Summary
Existing work on agent systems, interoperability protocols, and self-optimization has laid important foundations for autonomous behavior. However, these efforts do not provide a unified protocol for managing the persistent state evolution of agent-internal resources. In particular, current connectivity protocols emphasize interaction but do not address lifecycle management or versioned state mutation. AUTOGENESIS addresses this gap by introducing a two-layer protocol architecture that separates the definition of evolvable resources from the mechanisms that govern their evolution, enabling modular, traceable, and auditable self-evolution in multi-agent systems.
3. Autogenesis
Despite growing interest in self-evolving agents (Gao et al., 2025), most systems remain engineered in an ad hoc manner and lack a shared protocol standard that makes evolution composable, auditable, and interoperable. We introduce AGP, a two-layer self-evolution protocol. The Resource Substrate Protocol Layer (RSPL) specifies the evolvable substrate, namely which resources may change and how they are represented, versioned, and accessed. The Self-Evolution

Protocol Layer (SEPL) specifies the evolution logic, namely how updates are proposed, assessed, and committed through a safe operator interface. Inspired by interface standardization efforts in agent tooling (e.g., the Model Context Protocol), this separation cleanly decouples what evolves from how evolution occurs, enabling modularity, traceability, and safety-preserving evolution across components.

3.1. Layer 1: Resource Substrate Protocol Layer
The Resource Substrate Protocol Layer (RSPL) defines the evolvable substrate as a set of protocol-registered resources2 with explicit state, lifecycle, and version lineage. In this paper, these resources comprise (i) instructions (Prompt), (ii) decision policies (Agent), (iii) actuation interfaces (Tool), which encompass native tool scripts, MCP tools (Anthropic, 2025a), and agent skills (Anthropic, 2025b), (iv) task/world dynamics (Environment), and (v) persistent state (Memory). Crucially, resources in RSPL are passive: they encapsulate no optimization logic and cannot self-modify; all observations and state transitions occur only through controlled, interface-mediated operations invoked by higher layers.

3.1.1. CORE ENTITIES
We focus on these five entity types as a minimal yet expressive substrate for agentic systems. This choice is not intended to be exhaustive, but rather to identify a common denominator across modern agent stacks and provide a uniform target space on which SEPL can operate.
Definition 3.1 (Resource Entity). A resource entity of type  and its type-level collection can be represented as:

e,i = (n,i, d,i, ,i, g,i, m,i),

(1)

E = { e,i | i  I },

where T = {PROMPT, AGENT, TOOL, ENV, MEM} denotes the set of RSPL entity types,   T indexes the entity type, I is the index set of resource instances of type  , and i  I indexes an individual instance. Here n,i is a unique resource name, d,i is a short description, ,i : X  Y is an input-to-output mapping, g,i  {0, 1} is the trainable marker that indicates whether the resource is evolvable, and m,i is an auxiliary metadata dictionary.

A key motivation for making prompt, tool, and memory explicit RSPL resources is decoupling. Many agent systems package prompts, tools, and memory as internal components of an agent, which entangles agent logic with task-specific instructions and capability bundles, increasing maintenance and limiting transfer(Wu et al., 2024; Hong et al., 2023; Chen et al., 2023). By externalizing them as first-class, ver-
2Unless otherwise specified, we use resources to refer to instances of the five RSPL entity types: prompt, agent, tool, environment, and memory.

3

Autogenesis: A Self-Evolving Agent Protocol

Protocol
Layer 1: Resource Substrate Protocol Layer (RSPL) Layer 2: Self-Evolution Protocol Layer (SEPL)

Core Resources

Evolvable Variables (Vevo)

Prompt ( ) Prompt

Agent (Agent)

Tool Environment Memory

(Tool)

(Env)

(Men)

Server Interface & Context Manager
Registration
Lifecycle Control Version Lineage
State Access

Infrastructure Services

Version

Model

Manager Manager

Dynamic Manager

Tracer Module

V1.0.1

V1.0.0

V1.0.3

V1.0.6

V1.0.0

V1.0.1

Operator Algebra & Evolutionary Loop

1. Generate
Answer Initialization

5. Commit
Improvement Commit

Multi-Agent Optimization
Cycle

2. Reflect
Proposal Generation

4. Evaluate
Answer Evaluation

3. Improve
Variables Improvement

Application
Multi-agent System

Reporter Agent

Add Content

Export Report

Browser Use Agent

Decide Actions

Browser Actions

Record Results

User Objectives Actions

Tools

Planning Agent

create Create a new plan delete Delete the plan update Update the plan mark Mark step as completed

Planning Tool Create, update, and manage plans for complex tasks simultaneously Track execution states

Planning
Interprete user tasks

Decompose into manageable
sub-tasks

Assign to specialized
sub-agents

Feedback

sub-agent A sub-agent B tool C
......

Objective Shifts (Update Plans)

& Unexpected Errors

Deep Researcher Agent

Deep Analyzer Agent

Tool Generator Agent

Optimize Queries

Search Tools

Refine Insight

Organize

Reason and

Diverse Formats Summarize

Tool Retrieval

Tool Creation

Tool Reuse

Game Playing

Math Problem Solving Computer Use Trading

Brower Use

Figure 1. The Autogenesis architecture.

sioned resources with standardized interfaces, the same toolcalling agent policy can be paired with different prompts and tool sets, and deployed unchanged across tasks and environments.

To support resource registration, unified management, and instantiation, RSPL stores a serializable registration record for each resource instance.
Definition 3.2 (Resource Registration Record). A resource registration record and its type-level collection can be represented as:

c,i = (e,i, v,i, ,i, ,i, F,i),

(2)

C = { c,i | i  I },

where   T indexes the entity type and i  I indexes an individual instance. Here e,i is the resource entity tuple defined in Theorem 3.1, v,i  V is a version string, ,i is an implementation descriptor (e.g., import path, class definition, or source-code string), ,i are instantiation parameters (e.g., constructor arguments), and F,i is a set of exported representations used by LLMs to interact with the resource (e.g., function-calling schema, natural-language text, and structured argument schema).
Definition 3.3 (Protocol-registered resource). For each entity type  , let R denote the type-specific registry of protocol-registered resources, and let R =  R denote the global registry. RSPL binds each entity type  to a dedicated context manager M and a server-exposed interface A . We represent the type-level registered resource as

r = (C , M , A ),

(3)

where each c,i  C is a registration record in Theorem 3.2. The context manager M maintains the collection C , the

version lineage for type  , and implements lifecycle and update operations over these records; the server-exposed interface A encapsulates M and exposes a unified external interface by delegating requests to the corresponding context-manager routines.
Context manager. The context manager implements the management plane for each resource type. Beyond lifecycle control and dependency constraints, it maintains (i) an active registry of materialized resources and (ii) a versioned history for restoration. Its exported API can be viewed as a small set of functionally grouped operators for lifecycle and registration (e.g., init, build), retrieval and inspection (e.g., list, get state), evolution and versioning (e.g., update, restore), execution and contract (e.g., run, load contract), and serialization and deserialization (e.g., save to json, load from json). The manager explicitly supports contract generation, producing a consolidated capability and constraint specification for the managed entities, which provides stable, up-to-date descriptions that improve reliability and reduce prompt bloat, enabling systematic context engineering via controlled prompt injection. For instance, for tools (which may be native scripts, MCP tools, or agent skills) the contract can take a skills.mdstyle form (Anthropic, 2025b) that enumerates tool actions, arguments, preconditions, and usage constraints. Concrete interface instantiations can be found in Section C.1.2.
Server interface. The server is introduced to encapsulate the context manager's internal complexity and present a stable, simplified interface for external callers. It packages heterogeneous management routines behind a uniform set of endpoints with consistent request/response semantics, while delegating the implementation details to the context manager. This separation isolates clients from internal design

4

Autogenesis: A Self-Evolving Agent Protocol

changes, reduces coupling, and provides a single control plane through which the protocol mediates safe, versionaware interactions with RSPL resources.
3.1.2. INFRASTRUCTURE SERVICES
RSPL further includes cross-cutting services that support reliable evolution, including reproducibility, safe deployment, and versioned recovery:
Model manager. A unified model-API layer that standardizes calls across providers (e.g., OpenAI, Anthropic, Google, and OpenRouter, etc.), while supporting routing, fallback, and cost-aware selection to keep model access consistent as components evolve.
Version manager. Maintains version lineage for each resource, enabling rollback, branching, and diffing. Versions are auto-incremented identifiers (e.g., semantic versions) assigned on register or update, each referencing an immutable snapshot of the configuration record and associated artifacts for auditability and reproducibility.
Dynamic manager. Handles serialization or deserialization of resource configurations for persistence and transfer, enabling safe hot-swapping of resources at runtime without restarting the agent system.
Tracer Module. A module that captures fine-grained execution traces (inputs, outputs, intermediate decisions, tool interactions, etc.) for interpretability and debugging, and as training signals for dataset synthesis and retrospective improvement.
3.2. Layer 2: Self-Evolution Protocol Layer (SEPL)
The Self-Evolution Protocol Layer (SEPL) establishes a control-theoretic formalism for agentic system evolution. It conceptualizes the continuous improvement of an agentic system as a generalized optimization problem defined over a heterogeneous state space. Formally, SEPL models evolutionary dynamics as a state transition function governed by a strictly typed operator algebra.
By mediating all state mutations through standardized RSPL interfaces, the protocol guarantees that evolution is traceable, reversible, and safe-by-construction. While this paper focuses on the reflection-driven optimizer as the primary instantiation, our implementation also supports other optimization strategies, including TextGrad (Yuksekgonul et al., 2025), GRPO (Shao et al., 2024), and Reinforce++ (Hu, 2025b), utilizing the same state manipulation primitives. Further details on these alternative implementations are provided in Section C.2.

3.2.1. EVOLVABLE VARIABLES
To transition from heuristic adaptation to a systematic evolution protocol, we introduce the concept of variable lifting. This abstraction projects discrete, heterogeneous RSPL resources (e.g., tool code, system prompts) onto a unified representation of evolvable variables. This formalism offers significant theoretical advantages by homogenizing the interaction surface for evolutionary operators and rigorously delineating the trainable subspace via an explicit learnability mask.
Definition 3.4 (Evolvable Variable Set). We define the universal set of evolvable variables, Vevo, as the union of all managed resource entities and execution artifacts:

Vevo =

E  {y},

(4)

 T

where E denotes the set of resource entities of type  governed by the RSPL. The element y encapsulates execution artifacts, specifically final outputs and reasoning traces, which constitute the observational basis for retrospective optimization. Furthermore, each variable v  Vevo is associated with a binary learnability constraint gv  {0, 1}, thereby strictly defining the trainable parameter subspace  = {v  Vevo | gv = 1}.

3.2.2. OPERATOR ALGEBRA
To formalize the evolutionary trajectory as a rigorous control process, we decompose the state transition function into atomic operations that correspond to the canonical phases of iterative optimization: observation, attribution, proposal, verification, and commit. Consequently, we establish five necessary auxiliary spaces to ensure the process is mathematically well-defined. The trace space Z guarantees system observability; the hypothesis space H provides the basis for semantic error attribution; the modification space D formalizes the modification primitives; the objective specification G defines the optimization landscape; and the evaluation space S encapsulates performance metrics and safety status. These components constitute the minimal sufficiency required to close the self-evolution loop.
Reflect (). Defined as  : Z � Vevo  (H), this operator bridges the gap between raw observation and optimization direction. It approximates the "semantic gradient" of the system by mapping high-dimensional execution traces to specific, causal failure hypotheses within the variable space.
Select (). Formulated as  : Vevo �(H)  (D), this operator acts as the generative policy. It translates diagnostic hypotheses into concrete update proposals, sampling candidate modifications D designed to minimize the identified error signal subject to structural constraints.
Improve (). The mutation operator,  : Vevo�(D)  Vevo,

5

Autogenesis: A Self-Evolving Agent Protocol

executes the physical state transition. It applies discrete updates D via standardized RSPL interfaces to yield a provisional candidate state.
Evaluate (). Specified as  : Vevo � G  S, this operator serves as the objective function. It maps the candidate state and goal specification to the evaluation space S (comprising quantitative scores and strict safety invariants).
Commit (). Operating as  : Vevo � S  Vevo, this function acts as a conditional gating mechanism. It utilizes the evaluation signals in S to govern state transition, rigorously enforcing safety invariants and performance monotonicity by accepting the candidate Vevo only when specific success criteria are met.
3.2.3. THE EVOLUTIONARY LOOP
The atomic operators defined above are orchestrated into a rigorous closed-loop process, summarized in Algorithm 1. Starting from an initial state Ve(v0o), SEPL iteratively executes the system to generate observational traces (Z), derives causal failure hypotheses (H), and synthesizes modification primitives (D).
Crucially, the loop is closed via the evaluation space S and the commit operator . This design ensures that selfevolution is not a random walk, but a directed trajectory that is grounded in execution data, traceable through versioned updates, and monotonically improving under strictly defined safety invariants.
4. AGS and Optimization Strategies
This section presents the concrete instantiation of the AGP protocol, demonstrating its practical usability as a selfevolving agent system.
4.1. AGS Architecture
Building on AGP, we instantiate the two-layer protocol into AGS, a self-evolving multi-agent system organized around an Agent Bus architecture (Wu et al., 2024; Hong et al., 2023). Rather than relying on a monolithic controller or a rigid pipeline, AGS uses a shared message bus as the central coordination backbone: all agents communicate exclusively through standardized bus messages, enabling loose coupling, transparent observability, and concurrent sub-agent execution. Throughout all configurations, prompts, tools (including native scripts, MCP tools, and agent skills), and memory are treated as first-class RSPL resources with explicit lifecycle and version lineage, rather than hard-coded internal components. The system operates through three interleaved mechanisms:
Orchestration via Plan Generation. Upon receiving a task from the Agent Bus, the Orchestrator is responsible

Algorithm 1 SEPL Evolutionary Loop

Input: Agentic System A, Objective G, Budget T

Output: Optimized state Vevo

1: Initialization: 2: Ve(v0o)  VariableLifting(A)  Project resources to

optimization manifold

3: Z(0)  Execute(A, Ve(v0o))

 Obtain initial

observational trace

4: Optimization Cycle:

5: for t = 0, 1, . . . , T - 1 do

6: // Phase 1: Diagnosis & Proposal 7: H(t)  (Z(t), Ve(vto))  Reflect: Compute semantic

gradients

8: D(t)  (Ve(vto), H(t))

 Select: Generate

modification primitives

9: // Phase 2: Mutation & Verification 10: Ve(vto+1)  (Ve(vto), D(t))  Improve: Apply updates

to candidate

11: S(t+1)  (Ve(vto+1), G)

 Evaluate: Map to

evaluation space

12: // Phase 3: Gating & Transition 13: Ve(vto+1)  (Ve(vto+1), S(t+1))
Conditional state transition

 Commit:

14: // Phase 4: Next Iteration 15: Z(t+1)  Execute(A, Ve(vto+1)) 16: if Converged(S(t+1)) then 17: break 18: end if 19: end for 20: return Ve(vto)

solely for planning and coordination; it does not execute subtasks directly. Concretely, the Orchestrator produces a structured plan.md artifact that records the overall task decomposition: a human-readable flowchart of the execution graph, an ordered list of subtask steps, and the assignment of each subtask to a designated sub-agent (e.g., deep researcher, browser-use agent, tool-calling agent, or tool generator). This plan is registered as a versioned RSPL resource, making the coordination structure itself inspectable and evolvable. The Orchestrator then broadcasts each subtask together with its specification to the corresponding sub-agents via the bus.
Concurrent Sub-Agent Execution and Iterative Replanning. Upon receiving a broadcast subtask, each subagent independently retrieves the relevant prompt and tool resources from the RSPL registry via semantic search, executes tool calls to interact with the environment, and writes intermediate results and reasoning traces to shared memory as persistent, queryable state. Sub-agents operate concurrently: the bus decouples task dispatch from task completion,

6

Autogenesis: A Self-Evolving Agent Protocol

so multiple sub-agents may execute in parallel without synchronization overhead. Once all sub-agents in the current round have completed, the Orchestrator collects their outputs via the bus, summarizes the aggregated results, and updates plan.md with the current execution state. Based on this global view, the Orchestrator decides whether the task is complete or whether a further round of subtask decomposition and broadcast is required. This collect-and-replan loop repeats until the termination condition is satisfied, enabling the system to handle tasks of arbitrary depth and branching complexity. As a complementary pattern, AGS also supports agent-as-tool composition, in which a sub-agent is wrapped behind a standard RSPL tool schema and directly invoked by a tool-calling agent alongside conventional tools, MCP services, and skills, enabling lightweight multi-agent collaboration without bus-level orchestration.
Self-Evolution. Interleaved with the bus coordination loop, AGS triggers the SEPL evolutionary loop (Algorithm 1) whenever observational traces signal correctable failures or suboptimal performance. Concretely, the agent (i) reflects on execution traces Z (tool outputs, errors, latencies, reward signals, and task progress) to derive causal failure hypotheses H, (ii) selects targeted modification proposals D over evolvable variables (e.g., prompt text, tool source code for native scripts, MCP tool configurations, skill definitions, or the plan structure itself), (iii) applies candidate updates to produce a provisional state Vevo, (iv) evaluates the candidate against the objective G, and (v) commits accepted modifications as versioned transitions with auditable lineage and rollback. Failed evolution attempts are rolled back without side effects, and successful ones become immediately available to all sub-agents in subsequent bus rounds. This tight integration ensures that evolution is always safe, traceable, and composable across the full lifetime of the agent network.

operator  then translates these hypotheses into concrete modification proposals D(t), such as appending constraint clauses to the system prompt or rewriting a function body. The Improve operator  applies these proposals through the RSPL set variables interface to produce a candidate state. The Evaluate operator  re-executes the task under the candidate state and compares performance against the objective G. Finally, the Commit operator  accepts the update only if performance improves or safety invariants are preserved, otherwise rolling back to the previous version. This reflection-driven loop is repeated for a fixed budget of T rounds.
Alternative Strategies. Beyond reflection, our implementation supports additional optimization strategies that map naturally onto the same SEPL operator interface:
TextGrad (Yuksekgonul et al., 2025) treats the naturallanguage feedback produced by  as a "textual gradient" and applies gradient-descent-like updates to string-valued variables (prompts, code). Within AGP, TextGrad instantiates  as a gradient-informed proposal generator and  as a string-level edit operator, while reusing the standard  and  for evaluation and gating.
Reinforce++ / GRPO (Hu, 2025b; ?; Ziegler et al., 2019; Schulman et al., 2017) adopt a reinforcement-learning perspective, treating the evolvable variables as a policy and the evaluation signal as a reward. Here,  samples multiple candidate trajectories,  ranks them by reward,  updates the policy parameters (e.g., prompt weights or LoRA adapters) via policy-gradient estimates, and  commits only if the updated policy exceeds a baseline return threshold. These strategies demonstrate that the SEPL operator algebra is sufficiently general to accommodate both inference-time text optimization and gradient-based parameter updates within a unified protocol.

4.2. Instantiating the Optimizer
The AGP protocol is agnostic to the specific optimization strategy: any procedure that conforms to the five-operator SEPL interface (, , , , ) can serve as the evolutionary engine. We describe the primary instantiation used in our experiments and briefly outline alternative strategies supported by our implementation.
Reflection Optimizer. The default optimizer in our experiments implements the SEPL loop through natural-language reflection (Shinn et al., 2023; Madaan et al., 2023). Given an execution trace Z(t) and the current evolvable state Ve(vto), the Reflect operator  prompts the backbone LLM to analyze failures and generate structured diagnostic hypotheses H(t) in natural language (e.g., "the prompt lacks explicit instruction for edge-case handling" or "the sorting algorithm has O(n2) complexity on the critical path"). The Select

5. Empirical Studies
In this section, we present empirical results of deploying AGS across various challenging benchmarks with AGP protocol to demonstrate its comprehensive capabilities.
Benchmark Instruction. For GPQA-Diamond (198 questions), we adopt a closed-book, non-retrieval evaluation protocol. The agent is presented with a graduate-level STEM multiple-choice question (covering biology, chemistry, and physics) and must output exactly one option as the final answer. GPQA-Diamond is designed to be Google-proof, such that simple web search is insufficient and success typically requires difficult, multi-step scientific reasoning beyond factual recall. Overall, this benchmark measures the agent's deep scientific understanding and closed-book reasoning ability. For AIME, we use problems from the 2024 and 2025 American Invitational Mathematics Examination (AIME24

7

Autogenesis: A Self-Evolving Agent Protocol

and AIME25), each consisting 30 problems. Each instance requires the agent to solve a competition-level problem and output a single integer answer. We evaluate performance by exact-match accuracy, which primarily measures the agent's long-horizon symbolic reasoning and arithmetic precision. For GAIA, we evaluate on the GAIA Test split (300 tasks). Each task specifies a real-world, multi-step objective that typically requires planning and tool use (e.g., web browsing and document/file operations). We measure performance by task success (completion), which primarily reflects the agent's long-horizon planning and reliable tool-use execution. For LeetCode, we construct an in-house, LeetCode multi-language programming benchmark to evaluate executable code generation under reduced data contamination. To mitigate potential training-data contamination from widely circulated legacy problems, we intentionally select recently released problems across diverse categories (e.g., arrays, trees, linked lists, etc.) and split them into 200 training problems and 100 test problems. The agent solves each problem in one of multiple languages (Python, C++, Java, Go, etc.), and we report multiple metrics including overall score (acceptance), test-case pass rate, and runtime, which together measure algorithmic reasoning, implementation correctness, and efficiency.
5.1. Experiments on Scientific and Mathematical Benchmarks
5.1.1. EXPERIMENT SETTING
To validate our self-evolving agent AGS based on the AGP protocol, we conduct experiments across GPQADiamond, AIME24, and AIME25, focusing on evolving prompts and agent outputs. These benchmarks represent standard reasoning tasks where evolution of agent architecture, memory systems, environments, and tools is relatively less critical compared to instruction refinement and solution quality. To isolate the self-evolution capability on prompts and solutions, we deliberately do not equip AGS with any external tools in this setting, and compare three evolution strategies: evolve prompt only, evolve solution only, and the combined evolve prompt+solution. To ensure comprehensive coverage across model capabilities, we evaluate using multiple backbone models: lower-performing models (gpt-4o, gpt-4.1), a mediumperforming model (claude-sonnet-4.5), and a high-performing model (gemini-3-flash-preview, grok-4.1-fast). Our self-evolution algorithm primarily employs the reflection optimizer with a maximum of 3 optimization rounds, after which the agent output is taken as the final solution.
Metrics. We measure performance by exact-match accuracy: for GPQA-Diamond, the agent's selected option must match the ground-truth multiple-choice answer; for

AIME24 and AIME25, the agent's numerical output must exactly match the reference integer answer.

5.1.2. RESULTS AND ANALYSIS

Table 1. Results on GPQA-Diamond, AIME24 and AIME25.

Approach

GPQA-Diamond AIME24 AIME25

vanilla evolve prompt evolve solution evolve prompt + solution

gpt-4o 47.98 53.81 53.53 58.08

13.34 13.34 16.67 16.67

6.67 13.34 13.34 13.34

Improvement(%)

21.05

24.97 100

vanilla evolve prompt evolve solution evolve prompt + solution

gpt-4.1 65.15 68.68 68.68 67.67

23.34 33.33 36.67 40.00

20.00 23.33 30.00 33.33

Improvement(%)

3.87

71.38 66.65

grok-4.1-fast

vanilla

83.33

evolve prompt

83.84

evolve solution

87.81

evolve prompt + solution

89.34

96.67 96.67 96.67 96.67

90.00 93.33 90.00 96.67

Improvement(%)

7.21

0.00 7.41

claude-sonnet-4.5

vanilla

78.28

evolve prompt

79.79

evolve solution

80.30

evolve prompt + solution

81.44

76.67 86.67 80.00 86.67

73.33 90.00 90.00 90.00

Improvement(%)

4.04

13.04 22.73

gemini-3-flash-preview

vanilla

88.38

evolve prompt

88.89

evolve solution

87.88

evolve prompt + solution

90.40

83.33 93.33 93.33 93.33

83.33 86.67 90.00 93.33

Improvement(%)

2.28

12.00 12.00

The results in Table 1 reveal four key observations across models and evolution strategies. (1) Weak models gain more; strong models gain less. gpt-4.1, with lower vanilla baselines (23.3% on AIME24, 20.0% on AIME25), improves by 71.4% on AIME24 and 66.7% on AIME25 under evolve prompt+solution. In contrast, gemini-3-flash-preview starts at 83�88% and improves by 2.3% on GPQA-Diamond and 12.0% on both AIME benchmarks. The reason is straightforward: evolution corrects errors exposed during reflection; weaker models make more correctable mistakes, whereas stronger models already operate near ceiling. claude-sonnet-4.5 occupies a middle tier (76�78% vanilla) and improves by 4.0%, 13.0%, and 22.7% on GPQA, AIME24, and AIME25, respectively, confirming that headroom correlates with evolution benefit. (2) Combined evolution dominates prompt-only and solution-only. Across all models, evolve prompt+solution consistently yields the best scores. For gpt-4.1 on AIME24, evolve prompt reaches 33.3% and evolve solution 36.7%, whereas the combined approach

8

Autogenesis: A Self-Evolving Agent Protocol

reaches 40.0%; on AIME25, the respective scores are 23.3%, 30.0%, and 33.3%. claude-sonnet-4.5 shows similar patterns: evolve prompt+solution outperforms either single strategy on all three benchmarks. This suggests that instruction refinement and solution refinement address complementary failure modes; combining both closes more errors than either alone. (3) Math benchmarks respond more strongly than science QA. AIME24 and AIME25 exhibit larger relative gains than GPQA-Diamond. For gpt-4.1, GPQA improves by 3.9% while AIME24 improves by 71.4%; for gemini-3-flash-preview, GPQA improves by 2.3% while both AIME benchmarks improve by 12.0%. Long-horizon symbolic reasoning (multi-step derivations, arithmetic chains) exposes more intermediate failure points that reflection can target; closed-book science QA, by contrast, relies more on factual recall where prompt/solution refinement offers fewer levers. (4) Ceiling effects cap evolution on saturated benchmarks. grok-4.1-fast reaches 96.7% on AIME24 with vanilla, leaving minimal headroom; evolution yields no gain there. It still improves GPQA and AIME25 by 7.2% and 7.4%, respectively, where baselines are lower. This reinforces that self-evolution is most effective when both model capability and benchmark difficulty leave room for improvement.
In summary, AGS delivers consistent gains across diverse model capabilities and benchmarks throughout our experiments. Stronger models improve modestly but reliably; weaker models improve substantially when sufficient headroom exists. The combined prompt+solution evolution strategy consistently outperforms single-strategy evolution, and math benchmarks benefit more strongly than science QA from iterative refinement.
5.2. Experiments on General Agent Benchmark
5.2.1. EXPERIMENT SETTING
For GAIA, we focus on evolving tools, as GAIA tasks primarily depend on tool capabilities rather than pure reasoning. Our system architecture consists of a top-level planner agent (m = 50) and multiple specialized subagents: a deep researcher (m = 3), a browser-use agent (m = 3), a report agent, a tool generator (m = 3), and a deep analyzer agent (m = 3). All agents utilize gemini-3-flash-preview as the backbone model, where m denotes the maximum number of reasoning steps per agent. The self-evolution of tools is primarily driven by the tool generator agent: given a subtask, it first retrieves candidate tools from the managed tool registry via semantic search; if a suitable tool is found, the agent attempts to execute it, and upon encountering errors, iteratively refines the tool's source code through reflection; if no suitable tool exists, the agent synthesizes a new tool from scratch and registers it as a versioned RSPL resource for future reuse.

Metrics. We adopt the Pass@1 score on the GAIA Test split and report task-completion accuracy at each difficulty tier (Level 1, Level 2, Level 3) as well as the overall average.

Table 2. Performance results for agents on GAIA Test benchmark.

Agent
o4-mini-DR (OpenAI, 2025) JoyAgent (Liu et al., 2025) o3-DR (OpenAI, 2025) Langfun (Google, 2024) Alita (Qiu et al., 2025) DeSearch (Desearch-ai, 2024) h2o (H2O.ai, 2025) Su-Zero-Ultra AWorld (Yu et al., 2025) HALO (Hou et al., 2025) ToolOrchestra (Su et al., 2025)
vanilla evolve tool
Improvement(%)

Level1 Level2 Level3 Average

67.59 77.42 79.42 84.95 92.47 91.40 89.25 93.55 95.70 94.62 95.70

59.10 67.30 68.97 73.58 71.70 75.47 79.87 77.36 81.13 84.91 82.39

44.28 46.94 47.48 48.98 55.10 61.22 61.22 65.31 57.14 69.39 87.76

59.30 67.11 68.70 73.09 75.42 78.07 79.73 80.40 81.73 85.38 87.38

91.40 77.36 61.22 79.07 98.92 85.53 81.63 89.04

8.23 10.56 33.34 12.61

The results in Table 2 reveal three key observations. (1) AGS achieves state-of-the-art performance. With an average score of 89.04%, AGS surpasses all public leaderboard entries, outperforming the next-best agent ToolOrchestra (87.38%) by 1.66 percentage points. This advantage is especially pronounced on the hardest tier: AGS scores 81.63% on Level 3, compared to 69.39% for HALO and 57.14% for AWorld, demonstrating that evolution-driven adaptation provides the largest gains where task complexity is highest. On the easier tiers, the gap narrows but remains consistent: Level 1 reaches 98.92% (vs. 95.70% for ToolOrchestra) and Level 2 reaches 85.53% (vs. 84.91% for HALO), indicating that tool evolution provides broad-spectrum improvement rather than being limited to a single difficulty regime. (2) Tool evolution yields large gains on hard tasks. Compared to the vanilla baseline (79.07% avg.), evolve tool improves performance by 12.6% overall. The improvement is strongly skewed toward difficulty: Level 1 gains 8.2%, Level 2 gains 10.6%, and Level 3 gains 33.3%. This pattern mirrors the headroom effect observed in the math benchmarks: harder tasks expose more correctable failure modes, which the reflection-driven tool evolution can target. Notably, the 33.3% gain on Level 3 represents the single largest relative improvement across all benchmarks in our study, underscoring that tool evolution is particularly effective when tasks demand complex multi-step tool chains that static toolkits cannot adequately cover. (3) Hierarchical resource management mitigates planning complexity. GAIA's multi-domain tasks require temporal and cross-modal state coherence, and many baselines degrade during domain transitions (e.g., from browser retrieval to local file analysis). By treating prompts, tools, and environments as first-class RSPL resources with explicit lifecycle management, AGS preserves session-critical

9

Autogenesis: A Self-Evolving Agent Protocol

state across agent boundaries, reducing contextual forgetting and enabling compositional generalization on Level 2 and Level 3 scenarios. Furthermore, when the planning agent encounters novel subtasks, it invokes the tool generator to synthesize context-specific functionalities on the fly, bypassing the fixed-capability bottleneck of static agent toolkits. This dynamic tool creation and refinement loop, mediated entirely through the SEPL operator interface, ensures that new capabilities are version-tracked and reusable across subsequent tasks.
In summary, GAIA confirms that AGP's self-evolution protocol extends beyond pure reasoning tasks to complex, toolintensive agent scenarios. The largest gains emerge on the hardest task tiers, where iterative tool refinement and hierarchical resource management provide the most leverage.
5.3. Experiments on Algorithmic Coding Benchmark
5.3.1. EXPERIMENT SETTING
Benchmark design rationale. Our benchmark construction is driven by three motivations: (i) evaluating inference-time self-evolution on executable code, (ii) calibrating agent performance against the distribution of human submissions, and (iii) assessing cross-language robustness under long-tail language usage. We build on top of the LeetCode online judge, which provides an execution-based evaluation interface and rich feedback signals(Chen et al., 2021; Jimenez et al., 2023). Specifically, acceptance status and per-testcase pass rates enable fine-grained assessment of functional correctness beyond binary success. For accepted submissions, the platform reports runtime and memory usage along with percentile-based runtime beats and memory beats statistics computed against the distribution of human submissions, which directly supports human-referenced evaluation. Finally, LeetCode provides standardized starter code across many programming languages, enabling consistent and reproducible multi-language evaluation under a unified protocol.
Data collection. We collect the full set of 3,822 programming problems available on LeetCode at the time of crawling. For each problem, we extract the natural-language statement, official input�output examples, and languagespecific starter code templates. Each problem is annotated with its platform-provided difficulty label (Easy, Medium, Hard) and topical tags describing required algorithmic concepts (e.g., arrays, trees, dynamic programming). We perform quality checks including filtering malformed records, removing duplicates, and validating successful parsing of statements, examples, and templates. From the full pool, we select 100 recently released test problems across diverse categories to mitigate training-data contamination.
Evaluation protocol. We compare a vanilla baseline against

AGS with evolve solution enabled. For the vanilla baseline, the agent presents a fixed input representation to the model, deterministically extracts executable source code, and submits it to the execution-based judge in a single pass. For AGS, the agent iteratively refines solutions through the SEPL reflection optimizer within a fixed revision budget of 3 rounds, while keeping the task specification and evaluation interface unchanged. This controlled setup enables direct comparison between one-shot generation and inference-time self-evolution on solution quality. We evaluate across five languages (Python3, C++, Java, Go, Kotlin) using multiple backbone models and report multi-dimensional metrics.

Table 4. Evaluation metrics for the algorithmic coding benchmark.

Metric
PR TLE MLE CE RE WA TO RpE
AR AM APC
ARB AMB

Description
Capability metrics Number of problems passing all test cases within
time and memory limits. Number of problems exceeding the allowed exe-
cution time limit. Number of problems exceeding the allowed
memory usage. Number of problems where generated code
failed to compile. Number of problems encountering a runtime er-
ror during execution. Number of problems producing incorrect output. Number of problems where the model failed to
respond within the timeout. Number of problems where the model returned
an invalid or unparseable response.
Efficiency metrics Mean runtime (ms) of accepted solutions. Mean memory (MB) of accepted solutions. Mean test cases passed before failure.
Human-referenced metrics Percentage of accepted solutions whose runtime
outperforms human submissions. Percentage of accepted solutions whose memory
usage outperforms human submissions.

Metrics. As shown in Table 4, we report three groups of metrics that capture complementary aspects of coding performance. First, capability metrics measure functional correctness and failure modes under the judge constraints. Second, efficiency metrics summarize runtime and memory cost for accepted submissions. Third, human metrics quantify how often accepted submissions outperform the distribution of human solutions in runtime and memory.
The results in Table 3 and Figure 2 reveal four key findings across coding capability, efficiency, and human-referenced dimensions. (1) Self-evolution consistently improves pass rate across all languages. The evolve solution agent achieves relative pass-rate improvements ranging from 10.1% (Python3) to 26.7% (Kotlin), with compiled languages benefiting most: C++ reaches 99 and Java 98 out of 100 problems. These gains are accompanied by broad reductions in execution-blocking errors; compile error, runtime

10

Autogenesis: A Self-Evolving Agent Protocol

Table 3. Results based on gemini-3-flash-preview. Vanilla and evolve solution are reported; Improvement (%) denotes relative change.  indicates gain;  indicates degradation.

Agent

Capability metrics

Efficiency metrics

Human metrics

PR TLE MLE CE RE WA TO RpE AR (ms) AM (MB) APC ARB (%) AMB (%)

vanilla evolve solution

Python3 79 4 0 0 2 14 1 0 1376.19 56.59 750.89 73.28 87 3 0 0 1 9 0 0 1269.39 59.08 750.98 70.29

Improvement (%) 10.1 25.0 0 0 50 35.7 100 0 7.8

4.4 0.0 4.1

vanilla evolve solution

C++ 84 2 0 2 1 10 0 1 266.04 168.93 743.31 68.02 99 0 0 0 0 1 0 0 142.60 148.43 749.86 88.99

Improvement (%) 17.9 100 0 100 100 90 0 100 46.4 12.1 0.9 30.8

36.62 42.15 15.1
59.24 73.14 23.5

vanilla evolve solution

84 0 98 1

Improvement (%) 16.7 0

Java 0 2 2 9 1 2 125.04 0 0 0 1 0 0 96.30 0 100 100 88.9 100 100 23.0

126.09 752.86 71.03 120.00 751.09 88.33 4.8 0.2 24.4

59.18 72.38 22.3

vanilla evolve solution

Go 82 1 0 9 0 7 0 1 139.22 95 0 0 0 0 5 0 0 111.64

Improvement (%) 15.9 100 0 100 0 28.6 0 100 19.8

22.01 18.35 16.6

739.46 754.17 2.0

76.22 81.52 7.0

63.48 67.94 7.0

vanilla evolve solution

75 2 95 1

Improvement (%) 26.7 50

Kotlin 0 8 1 10 2 2 171.99 0 0 0 4 0 0 122.83 0 100 100 60 100 100 28.6

72.80 77.88 7.0

760.43 749.38 1.5

83.49 83.58 0.1

79.07 67.21 15.0

error, timeout, and response error frequently drop to zero, indicating that iterative refinement effectively repairs format and tooling issues that cause outright failures. Figure 2 (first row) corroborates this finding, showing consistently higher pass-rate trajectories for the evolving agent as problems accumulate. (2) Evolution improves runtime efficiency but shows mixed memory effects. Average runtime decreases in every language, with reductions of 7.8% in Python3 and 19.8�46.4% in compiled languages. Figure 2 (second row) confirms this trend: the evolving agent accumulates substantially lower cumulative runtime, and the gap widens as tasks accumulate. This pattern aligns with reductions in TLE errors, suggesting that reflection helps replace suboptimal algorithms with more efficient ones. Memory usage, however, shows a mixed trend: it decreases in C++, Java, and Go but increases modestly in Python3 and Kotlin, plausibly because the evolving agent introduces auxiliary data structures to ensure correctness or improve speed. (3) Evolved solutions become more competitive against human submissions. Runtime beats (ARB) increase strongly in compiled languages, with gains of 30.8% in C++ and 24.4% in Java, and smaller gains in Go (7.0%) and Kotlin (0.1%). Memory beats (AMB) increase in Python3, C++, Java, and Go, but decrease in Kotlin (15.0%). Figure 2 (third row) shows that the evolving agent sustains higher ARB and AMB trajectories than the vanilla agent in most settings, indicating that competitiveness against human submissions improves consistently over the inference trajectory. The Kotlin divergence mirrors the absolute memory trend and suggests that in long-tail languages the evolving agent may trade memory for correctness or speed. (4) Within-inference trajectories

ARB & AMB (%) Cumul. Runtime (ms)

PR (%)

Python 3 100 80 60 40 20
0 4 �105

C++ 100 80 60 40 20
0 8 �104

Java 100 80 60 40 20
0 1.5 �105

Go 100 80 60 40 20
0 1.5 �105

100 80 60 40 20 0 2.0 �105

Kotlin

3

6

1.0

1.0

1.5

2

4

1.0

1

2

0.5

0.5

0.5

0

0

0.0

0.0

0.0

100

100

100

100

100

80

80

80

80

80

60

60

60

60

60

40

40

40

40

40

20

20

20

20

20

0 0 Task5C0ount 100 0 0 Task5C0ount 100 0 0 Task5C0ount 100 0 0 Task5C0ount 100 0 0 Task5C0ount 100

Evolving Vanllia Runtime Beats Memory Beats

Figure 2. Performance comparison of evolving and vanilla agents within-inference.

reveal compounding improvement dynamics. Beyond endpoint metrics, Figure 2 enables trajectory-level analysis of self-evolution. Across all three metric groups, the gap between evolving and vanilla agents widens as problems accumulate rather than plateauing, suggesting that the reflection-driven optimizer continues to find correctable failure modes throughout the evaluation. This compounding behavior is most pronounced in the runtime panel, where cumulative efficiency gains accelerate in later problems.
In summary, self-evolution on the algorithmic coding benchmark delivers consistent improvements in functional correctness and runtime efficiency across all five languages, with the largest gains in compiled languages where the type system and compiler feedback provide richer signals for reflection. Human-referenced metrics confirm that these gains

11

Autogenesis: A Self-Evolving Agent Protocol translate into solutions that are increasingly competitive with human submissions. The within-inference trajectory analysis further demonstrates that AGP not only improves endpoint scores but also enables fine-grained visibility into when and how self-evolution provides the most leverage during a single inference episode.
6. Conclusion
We presented AGP, a two-layer self-evolution protocol that decouples what evolves from how evolution occurs. The Resource Substrate Protocol Layer (RSPL) models prompts, agents, tools, environments, and memory as first-class, versioned resources with explicit lifecycle and interface contracts. The Self-Evolution Protocol Layer (SEPL) specifies a closed-loop operator algebra for proposing, evaluating, and committing improvements with auditable lineage and rollback. Building on this protocol, we instantiated AGS, a thinking-and-action agent that dynamically retrieves, refines, and evolves heterogeneous resources during execution. We believe this protocol-level approach to self-evolution provides a principled foundation for building modular, traceable, and safely improvable agentic systems.
12

Autogenesis: A Self-Evolving Agent Protocol

References

Anthropic. Equipping agents for the real world with

agent skills. https://www.anthropic.com/

engineering/equipping-agents-for-the-

real-world-with-agent-skills,

2025a.

Accessed October 2025.

Anthropic. Introduction to agent skills. https:// anthropic.skilljar.com/introductionto-agent-skills, October 2025b.

Brown, T., Mann, B., Ryder, N., Subbiah, M., Kaplan, J. D., Dhariwal, P., Neelakantan, A., Shyam, P., Sastry, G., Askell, A., et al. Language models are few-shot learners. Advances in neural information processing systems, 33: 1877�1901, 2020.

Chen, M., Tworek, J., Jun, H., Yuan, Q., Pinto, H. P. D. O., Kaplan, J., Edwards, H., Burda, Y., Joseph, N., Brockman, G., et al. Evaluating large language models trained on code. arXiv preprint arXiv:2107.03374, 2021.

Chen, W., Su, Y., Zuo, J., Yang, C., Yuan, C., Chan, C.-M., Yu, H., Lu, Y., Hung, Y.-H., Qian, C., et al. Agentverse: Facilitating multi-agent collaboration and exploring emergent behaviors. In The Twelfth International Conference on Learning Representations, 2023.

Chen, Z., Deng, Y., Yuan, H., Ji, K., and Gu, Q. Self-play fine-tuning converts weak language models to strong language models. arXiv preprint arXiv:2401.01335, 2024.

Desearch-ai. desearch.py: Official Async Python SDK for the Desearch API. https://github.com/ Desearch-ai/desearch.py, 2024.

Gao, H.-a., Geng, J., Hua, W., Hu, M., Juan, X., Liu, H., Liu, S., Qiu, J., Qi, X., Wu, Y., et al. A survey of selfevolving agents: What, when, how, and where to evolve on the path to artificial super intelligence. arXiv preprint arXiv:2507.21046, 2025.

Google. Langfun: Object-Oriented Programming for Language Models. https://github.com/google/ langfun, 2024.

Google. A2a: A new era of agent interoperability. https: //developers.googleblog.com/en/a2a-anew-era-of-agent-interoperability/, April 2025. Google Developers Blog. Accessed: 2026-04-20.

H2O.ai. Enterprise h2oGPTe: Agentic AI for Generative and Predictive Intelligence. https://h2o.ai/ platform/enterprise-h2ogpte/, 2025.

Hong, S., Zhuge, M., Chen, J., Zheng, X., Cheng, Y., Wang, J., Zhang, C., Wang, Z., Yau, S. K. S., Lin, Z., et al. Metagpt: Meta programming for a multi-agent collaborative framework. In The twelfth international conference on learning representations, 2023.
Hou, Z., Tang, J., and Wang, Y. Halo: Hierarchical autonomous logic-oriented orchestration for multi-agent llm systems. arXiv preprint arXiv:2505.13516, 2025.
Hu, J. Reinforce++: A simple and efficient approach for aligning large language models. arXiv preprint arXiv:2501.03262, 2025a.
Hu, J. Reinforce++: A simple and efficient approach for aligning large language models. arXiv preprint arXiv:2501.03262, 2025b.
Jimenez, C. E., Yang, J., Wettig, A., Yao, S., Pei, K., Press, O., and Narasimhan, K. Swe-bench: Can language models resolve real-world github issues? arXiv preprint arXiv:2310.06770, 2023.
LeetCode. Leetcode online judge. https://leetcode. com. Accessed 2025.
Liu, J., Xu, S., Liu, S., Li, Y., Liu, W., Liu, M., Zhou, X., Wang, H., Jia, S., Tian, S., et al. Joyagent-jdgenie: Technical report on the gaia. arXiv preprint arXiv:2510.00510, 2025.
Madaan, A., Tandon, N., Gupta, P., Hallinan, S., Gao, L., Wiegreffe, S., Alon, U., Dziri, N., Prabhumoye, S., Yang, Y., et al. Self-refine: Iterative refinement with selffeedback. Advances in neural information processing systems, 36:46534�46594, 2023.
Mialon, G., Fourrier, C., Wolf, T., LeCun, Y., and Scialom, T. Gaia: a benchmark for general ai assistants. In The Twelfth International Conference on Learning Representations, 2023.
OpenAI. Introducing Deep Research. https://openai. com/index/introducing-deep-research/, February 2025.
Ouyang, L., Wu, J., Jiang, X., Almeida, D., Wainwright, C., Mishkin, P., Zhang, C., Agarwal, S., Slama, K., Ray, A., et al. Training language models to follow instructions with human feedback. Advances in neural information processing systems, 35:27730�27744, 2022.
Pryzant, R., Iter, D., Li, J., Lee, Y., Zhu, C., and Zeng, M. Automatic prompt optimization with "gradient descent" and beam search. In Proceedings of the 2023 conference on empirical methods in natural language processing, pp. 7957�7968, 2023.

13

Autogenesis: A Self-Evolving Agent Protocol

Qin, Y., Liang, S., Ye, Y., Zhu, K., Yan, L., Lu, Y., Lin, Y., Cong, X., Tang, X., Qian, B., et al. Toolllm: Facilitating large language models to master 16000+ real-world apis. arXiv preprint arXiv:2307.16789, 2023.
Qiu, J., Qi, X., Zhang, T., Juan, X., Guo, J., Lu, Y., Wang, Y., Yao, Z., Ren, Q., Jiang, X., et al. Alita: Generalist agent enabling scalable agentic reasoning with minimal predefinition and maximal self-evolution. arXiv preprint arXiv:2505.20286, 2025.
Rein, D., Hou, B. L., Stickland, A. C., Petty, J., Pang, R. Y., Dirani, J., Michael, J., and Bowman, S. R. Gpqa: A graduate-level google-proof q&a benchmark. In First Conference on Language Modeling, 2024.
Schick, T., Dwivedi-Yu, J., Dess`i, R., Raileanu, R., Lomeli, M., Hambro, E., Zettlemoyer, L., Cancedda, N., and Scialom, T. Toolformer: Language models can teach themselves to use tools. Advances in neural information processing systems, 36:68539�68551, 2023.
Schulman, J., Wolski, F., Dhariwal, P., Radford, A., and Klimov, O. Proximal policy optimization algorithms. arXiv preprint arXiv:1707.06347, 2017.
Shao, Z., Wang, P., Zhu, Q., Xu, R., Song, J., Bi, X., Zhang, H., Zhang, M., Li, Y., Wu, Y., et al. Deepseekmath: Pushing the limits of mathematical reasoning in open language models. arXiv preprint arXiv:2402.03300, 2024.
Shinn, N., Cassano, F., Gopinath, A., Narasimhan, K., and Yao, S. Reflexion: Language agents with verbal reinforcement learning. Advances in neural information processing systems, 36:8634�8652, 2023.
Su, H., Diao, S., Lu, X., Liu, M., Xu, J., Dong, X., Fu, Y., Belcak, P., Ye, H., Yin, H., et al. Toolorchestra: Elevating intelligence via efficient model and tool orchestration. arXiv preprint arXiv:2511.21689, 2025.
Touvron, H., Lavril, T., Izacard, G., Martinet, X., Lachaux, M.-A., Lacroix, T., Rozie`re, B., Goyal, N., Hambro, E., Azhar, F., et al. Llama: Open and efficient foundation language models. arXiv preprint arXiv:2302.13971, 2023.
Wang, Y., Yang, L., Li, G., Wang, M., and Aragam, B. Scoreflow: Mastering llm agent workflows via score-based preference optimization. arXiv preprint arXiv:2502.04306, 2025.
Wei, J., Wang, X., Schuurmans, D., Bosma, M., Xia, F., Chi, E., Le, Q. V., Zhou, D., et al. Chain-of-thought prompting elicits reasoning in large language models. Advances in neural information processing systems, 35:24824�24837, 2022.

Wu, Q., Bansal, G., Zhang, J., Wu, Y., Li, B., Zhu, E., Jiang, L., Zhang, X., Zhang, S., Liu, J., et al. Autogen: Enabling next-gen llm applications via multi-agent conversations. In First conference on language modeling, 2024.
Yao, S., Zhao, J., Yu, D., Du, N., Shafran, I., Narasimhan, K. R., and Cao, Y. React: Synergizing reasoning and acting in language models. In The eleventh international conference on learning representations, 2022.
Yu, C., Lu, S., Zhuang, C., Wang, D., Wu, Q., Li, Z., Gan, R., Wang, C., Hou, S., Huang, G., et al. Aworld: Orchestrating the training recipe for agentic ai. arXiv preprint arXiv:2508.20404, 2025.
Yuksekgonul, M., Bianchi, F., Boen, J., Liu, S., Lu, P., Huang, Z., Guestrin, C., and Zou, J. Optimizing generative ai by backpropagating language model feedback. Nature, 639(8055):609�616, 2025.
Zelikman, E., Wu, Y., Mu, J., and Goodman, N. Star: Bootstrapping reasoning with reasoning. Advances in Neural Information Processing Systems, 35:15476�15488, 2022.
Zhou, Y., Muresanu, A. I., Han, Z., Paster, K., Pitis, S., Chan, H., and Ba, J. Large language models are humanlevel prompt engineers. In The eleventh international conference on learning representations, 2022.
Ziegler, D. M., Stiennon, N., Wu, J., Brown, T. B., Radford, A., Amodei, D., Christiano, P., and Irving, G. Fine-tuning language models from human preferences. arXiv preprint arXiv:1909.08593, 2019.

14

Autogenesis: A Self-Evolving Agent Protocol

A. Notation
We summarize the main mathematical symbols and their meanings in Table 5. For readability, the notation is grouped by functional categories (grey rows), covering the RSPL substrate (resource entities, registration records, and registries) and the SEPL layer (evolvable variables, auxiliary spaces, and operator definitions used in the optimization loop).

Table 5. Notation used in the paper. Grey rows indicate categories.

Symbol
T  I i V (�)
e,i n,i d,i ,i : X  Y g,i m,i E
c,i C v,i ,i ,i F,i
R R M A r
Vevo v gv  y Z H D G S , , , , 
A T t Ve(vto) Z (t) H(t) D(t) Ve(vto+1) S (t+1)

Description
Indexing and Sets Set of RSPL entity types, {PROMPT, AGENT, TOOL, ENV, MEM}. Entity type index,   T . Index set of resource instances of type  . Instance index, i  I . Space of version strings. Power set operator.
RSPL Resource Entity (Def. C.1) Resource entity tuple (n,i, d,i, ,i, g,i, m,i). Unique resource name. Short description. Input-to-output mapping of the resource. Trainable marker indicating whether the resource is evolvable. Auxiliary metadata dictionary. Set of resource entities of type  .
RSPL Registration Record (Def. C.2) Registration record (e,i, v,i, ,i, ,i, F,i). Set of registration records for type  . Version string of the resource instance. Implementation descriptor (e.g., import path, class, or source). Instantiation parameters (e.g., constructor arguments). Exported representations for LLM interaction (schemas/text/structured args).
Protocol-registered Resource (Def. C.3) Type-specific registry of protocol-registered resources. Global registry,  R . Context manager for type  (maintains registry and version lineage). Server-exposed interface for type  (delegates to M ). Type-level registered resource triple (C , M , A ).
SEPL Variables, Spaces, and Operators Universal set of evolvable variables (all managed entities plus execution artifacts). A variable in Vevo. Learnability constraint for variable v (binary). Trainable subspace, {v  Vevo | gv = 1}. Execution artifacts (e.g., outputs and reasoning traces). Trace space. Hypothesis space. Modification space. Objective specification. Evaluation space (metrics and safety status). Reflect, Select, Improve, Evaluate, and Commit operators.
Optimization Loop (Alg. 1) Agentic system. Optimization budget (number of iterations). Iteration index.
Evolvable state at iteration t. Observational trace at iteration t. Hypotheses at iteration t. Proposed modifications at iteration t.
Candidate state after applying modifications. Evaluation result for the candidate state.

15

Autogenesis: A Self-Evolving Agent Protocol

B. Comparison with Other Protocols
We provide a structured comparison between Autogenesis, Google A2A, and Anthropic MCP in Table 6. The goal of this comparison is to position Autogenesis relative to widely used protocol abstractions in agent tooling, and to clarify which protocol-level primitives are required to make self-evolution composable, auditable, and safe in practice. Accordingly, the comparison is organized into four high-level dimensions (grey rows): Basic Information, Agent and System Capabilities, Evolvable Resource Management, and Self-Evolution Mechanism. Blue-highlighted entries emphasize the specific capabilities that enable closed-loop improvement (e.g., lifecycle control, version lineage, contract generation, and operatorized updates), which are not directly addressed by communication- or invocation-centric protocols.

Table 6. Protocol-level comparison: Autogenesis vs. Google A2A vs. Anthropic MCP across key dimensions for agentic systems and selfevolution. Symbols:  = Supported,  = Partial, � = Not supported. Highlighted rows (blue background) emphasize evolution-enabling capabilities.

Dimension
Proposer Protocol Focus Entity Scope
Agent First-Class Multi-Agent Tracer Memory as Resource
Lifecycle Ops Versioning and Rollback Registry and Retrieval Contract Generation
Closed-Loop Evolution Operatorized Updates Auditability
Model-Agnostic Scalability Open Ecosystem

Autogenesis

A2A

Basic Information

Our work

Google

Self-evolution Agentic System

Multi-agent System Collaboration

Prompt/Agent/Tool/Env/Memory Agent/Tool

Agent and System Capabilities















�

Evolvable Resource Management







�









Self-Evolution Mechanism



�



�





 O(log n) 

General and Ecosystem
 O(n2) 

MCP
Anthropic Tool Tool
� � � �
� �  �
� � 
 O(n) 

B.1. Basic Information
Proposer: This dimension identifies the originating organization and design context of each protocol. Google's A2A is introduced as part of an agent communication framework, focusing on enabling agents to collaborate via standardized interaction primitives. Anthropic's MCP (Model Context Protocol) is designed to standardize how LLMs connect to external tools and resources. Autogenesis is proposed in this work as a protocol for systematic self-evolution, targeting composable, auditable, and updateable agentic systems.
Protocol Focus: This dimension describes the primary interaction patterns and control plane each protocol standardizes. Autogenesis focuses on enabling closed-loop improvement of agentic systems by organizing resources and updates through protocol operators and versioned state. A2A focuses on multi-agent collaboration and communication. MCP focuses on standardizing model-to-tool (and resource) invocation interfaces.
Entity Scope: This dimension defines what is treated as first-class, protocol-governed components. Autogenesis explicitly manages heterogeneous entities (e.g., prompts, agents, tools, environments, and memory) as protocol-registered resources with explicit state and lineage, which is necessary for component-level evolution (e.g., prompt refinement, tool/code updates). A2A centers around agents (and their interactions), and typically does not establish tools/environments/memory as unified
16

Autogenesis: A Self-Evolving Agent Protocol
managed entities. MCP treats tools/resources as callable interfaces for LLMs, but does not natively model them as evolvable components with lifecycle and version lineage.
B.2. Agent and System Capabilities
Agent First-Class: First-class support means agents are modeled as managed protocol components with explicit schemas, metadata, and lifecycle hooks (enabling registration, discovery, orchestration, and controlled updates). Autogenesis supports agents as first-class resources. A2A provides agent-centric collaboration but often treats agents as service endpoints without unified lifecycle/version lineage. MCP does not define agents as protocol components, focusing instead on model-to-tool connectivity.
Multi-Agent: This dimension captures whether the protocol natively supports multi-agent composition beyond ad-hoc application logic. Autogenesis supports multi-agent configurations as part of a broader system substrate, enabling coordinated execution with traceability and evolution-ready state. A2A provides direct support for agent-to-agent collaboration. MCP does not address multi-agent orchestration as a protocol concern.
Tracer/Observability: Observability refers to whether the protocol provides native mechanisms to record execution traces (inputs/outputs, intermediate decisions, tool calls, state transitions) for debugging, evaluation, and learning signals. Autogenesis includes protocol-level tracing to support auditable evolution. A2A and MCP typically leave tracing to application-level implementations, which can lead to inconsistent observability.
Memory as Resource: This dimension reflects whether memory is explicitly modeled and managed as a protocol-level component. Autogenesis treats memory as a first-class resource (e.g., readable/writable state with explicit interfaces), enabling persistent improvement and reproducible evolution. A2A and MCP generally do not prescribe a memory management protocol, leaving memory to external systems.
B.3. Evolvable Resource Management
Lifecycle Ops: Lifecycle operations refer to standardized procedures for initializing, registering, constructing, and decommissioning protocol-managed components. Autogenesis provides explicit lifecycle operators so that updates can be applied safely to well-defined targets. A2A and MCP do not provide comprehensive lifecycle management across heterogeneous component types.
Versioning and Rollback: Version lineage and rollback provide the foundation for safe evolution: every update yields an auditable snapshot, supports comparison, and enables restoration when regressions occur. Autogenesis integrates version management as a protocol capability. A2A and MCP do not natively support version lineage for protocol-managed components, making systematic evolution difficult.
Registry and Retrieval: This dimension captures whether the protocol supports unified registration, listing, and retrieval of components (optionally via semantic search) to enable reuse and scalable coordination. Autogenesis maintains a registry of protocol-registered components and supports retrieval to reduce duplication and improve composability. A2A and MCP provide partial discovery mechanisms but do not define a unified management plane over heterogeneous components.
Contract Generation: Contract generation refers to producing consolidated, up-to-date capability and constraint specifications (e.g., tool actions, arguments, preconditions, usage constraints) for reliable orchestration and reduced prompt bloat. Autogenesis supports contract generation as a systematic form of context engineering. A2A and MCP generally rely on static descriptions or application-layer documentation without protocol-level contract aggregation.
B.4. Self-Evolution Mechanism
Closed-Loop Evolution: Closed-loop evolution means the protocol supports an iterative improvement loop (execute  diagnose  propose  verify  commit) rather than one-off adaptation. Autogenesis is explicitly designed around this loop to enable sustained improvement. A2A and MCP do not provide a native self-evolution loop.
Operatorized Updates: This dimension captures whether system updates are expressed as a typed, composable operator interface (rather than ad-hoc scripts), enabling controlled state transitions and repeatable evolution. Autogenesis defines self-evolution as operator-mediated transitions over protocol-managed resources. A2A and MCP do not define an operator algebra for evolution.
17

Autogenesis: A Self-Evolving Agent Protocol
Auditability: Auditability means that system changes are traceable and reviewable: what changed, why it changed, under what evidence, and with what evaluation outcome. Autogenesis emphasizes auditability through versioned lineage and trace-based evaluation signals. A2A and MCP provide only partial audit trails via external tooling rather than protocol-level guarantees.
B.5. General and Ecosystem
Model-Agnostic: This dimension captures whether the protocol can work across different LLM backends and providers. Autogenesis is model-agnostic by design via a unified model interface layer. A2A and MCP are also broadly model-agnostic as they define interaction standards rather than binding to a specific model.
Scalability: Scalability reflects how coordination and discovery behave as the number of components grows. Autogenesis supports scalable management by treating heterogeneous components as registry-governed resources with retrieval mechanisms, enabling efficient lookup and controlled orchestration. A2A may face coordination overhead as interactions densify in large multi-agent settings. MCP standardizes tool interfaces but may still rely on application-level orchestration for large tool/resource sets.
Open Ecosystem: Open ecosystem support refers to whether the protocol can enable a reusable ecosystem of interoperable components. Autogenesis provides a full protocol stack for managing, evolving, and auditing agentic components, which supports component sharing and safe integration. A2A and MCP offer partial ecosystem enablement focused on interoperability or tool interfaces, typically requiring additional layers for evolution-ready management.

C. Details of Self-Evolution Protocol
C.1. Layer 1: Resource Substrate Protocol Layer
The Resource Substrate Protocol Layer (RSPL) defines the evolvable substrate as a set of protocol-registered resources with explicit state, lifecycle, and version lineage. In this paper, these resources comprise (i) instructions (Prompt), (ii) decision policies (Agent), (iii) actuation interfaces (Tool), which encompass native tool scripts, MCP tools (Anthropic, 2025a), and agent skills (Anthropic, 2025b), (iv) task/world dynamics (Environment), and (v) persistent state (Memory). Crucially, resources in RSPL are passive: they encapsulate no optimization logic and cannot self-modify; all observations and state transitions occur only through controlled, interface-mediated operations invoked by higher layers.

C.1.1. CORE ENTITIES

We focus on these five entity types as a minimal yet expressive substrate for agentic systems. This choice is not intended to be exhaustive, but rather to identify a common denominator across modern agent stacks and provide a uniform target space on which SEPL can operate.
Definition C.1 (Resource Entity). A resource entity of type  and its type-level collection can be represented as:

e,i = (n,i, d,i, ,i, g,i, m,i),

(5)

E = { e,i | i  I },

where T = {PROMPT, AGENT, TOOL, ENV, MEM} denotes the set of RSPL entity types,   T indexes the entity type, I is the index set of resource instances of type  , and i  I indexes an individual instance. Here n,i is a unique resource name, d,i is a short description, ,i : X  Y is an input-to-output mapping, g,i  {0, 1} is the trainable marker that indicates whether the resource is evolvable, and m,i is an auxiliary metadata dictionary.

A key motivation for making prompt, tool, and memory explicit RSPL resources is decoupling. Many agent systems package prompts, tools, and memory as internal components of an agent, which entangles agent logic with task-specific instructions and capability bundles, increasing maintenance and limiting transfer. By externalizing them as first-class, versioned resources with standardized interfaces, the same tool-calling agent policy can be paired with different prompts and tool sets, and deployed unchanged across tasks and environments.
To support resource registration, unified management, and instantiation, RSPL stores a serializable registration record for each resource instance.
Definition C.2 (Resource Registration Record). A resource registration record and its type-level collection can be represented

18

Autogenesis: A Self-Evolving Agent Protocol

as:

c,i = (e,i, v,i, ,i, ,i, F,i),

(6)

C = { c,i | i  I },

where   T indexes the entity type and i  I indexes an individual instance. Here e,i is the resource entity tuple defined in Theorem C.1, v,i  V is a version string, ,i is an implementation descriptor (e.g., import path, class definition, or sourcecode string), ,i are instantiation parameters (e.g., constructor arguments), and F,i is a set of exported representations used by LLMs to interact with the resource (e.g., function-calling schema, natural-language text, and structured argument
schema).

Definition C.3 (Protocol-registered resource). For each entity type  , let R denote the type-specific registry of protocol-
registered resources, and let R =  R denote the global registry. RSPL binds each entity type  to a dedicated context manager M and a server-exposed interface A . We represent the type-level registered resource as

r = (C , M , A ),

(7)

where each c,i  C is a registration record in Theorem C.2. The context manager M maintains the collection C , the version lineage for type  , and implements lifecycle and update operations over these records; the server-exposed interface A encapsulates M and exposes a unified external interface by delegating requests to the corresponding context-manager routines.

C.1.2. CONTEXT MANAGER
The context manager implements the management plane for each resource type. Beyond lifecycle control and dependency constraints, it maintains (i) an active registry of materialized resources and (ii) a versioned history for restoration. Its exported API can be viewed as a small set of functionally grouped operators for lifecycle and registration (e.g., init, build), retrieval and inspection (e.g., list, get state), evolution and versioning (e.g., update, restore), execution and contract (e.g., run, load contract), and serialization and deserialization (e.g., save to json, load from json). The manager explicitly supports contract generation, producing a consolidated capability and constraint specification for the managed entities, which provides stable, up-to-date descriptions that improve reliability and reduce prompt bloat, enabling systematic context engineering via controlled prompt injection. For instance, for tools (which may be native tool scripts, MCP-connected tools (Anthropic, 2025a), or agent skills) the contract can take a skills.md-style form (Anthropic, 2025b) that enumerates tool actions, arguments, preconditions, and usage constraints. The exported management interface implemented by M and exposed by A are as follows:

C.1.3. SERVER INTERFACE
The server is introduced to encapsulate the context manager's internal complexity and present a stable, simplified interface for external callers. It packages heterogeneous management routines behind a uniform set of endpoints with consistent request/response semantics, while delegating the implementation details to the context manager. This separation isolates clients from internal design changes, reduces coupling, and provides a single control plane through which the protocol mediates safe, version-aware interactions with RSPL resources.

C.1.4. INFRASTRUCTURE SERVICES
RSPL further includes cross-cutting services that support reliable evolution, including reproducibility, safe deployment, and versioned recovery:
Model manager. A unified model-API layer that standardizes calls across providers (e.g., OpenAI, Anthropic, Google, and OpenRouter, etc.), while supporting routing, fallback, and cost-aware selection to keep model access consistent as components evolve.
Version manager. Maintains version lineage for each resource, enabling rollback, branching, and diffing. Versions are auto-incremented identifiers (e.g., semantic versions) assigned on register or update, each referencing an immutable snapshot of the configuration record and associated artifacts for auditability and reproducibility.
Dynamic manager. Handles serialization or deserialization of resource configurations for persistence and transfer, enabling safe hot-swapping of resources at runtime without restarting the agent system.

19

Autogenesis: A Self-Evolving Agent Protocol

Table 7. Operator set of Context Manager and Server Interface.

Operator
init build register unregister
get get info list retrieve get state
update copy restore get variables set variables
run save contract load contract
save to json load from json

Description
Lifecycle & Registration Auto discover resources and register the resource configuration to the registry. Build a resource instance from code and configuration. Register a new resource instance with a unique name and version. Unregister a resource instance from the active registry and version history.
Retrieval & Inspection Retrieve a resource instance by name from the active registry. Retrieve a resource configuration by name from the active registry. List all registered resource names. Retrieve similar resources via semantic search when supported. Get the current state of a resource instance when supported.
Evolution & Versioning Update a resource implementation and generate a new version. Duplicate a resource with an optional new name and version. Restore a specific historical version by name and version string. Expose resource code/configuration as evolvable variables. Update resource variables and generate a new version.
Execution & Contract Run a resource instance with structured input. Save the contract of a resource instance to a file. Load the contract of a resource instance from a file.
Serialization & Deserialization Serialize configurations and version history to a JSON file. Deserialize configurations and version history from a JSON file.

Tracer Module. A module that captures fine-grained execution traces (inputs, outputs, intermediate decisions, tool interactions, etc.) for interpretability and debugging, and as training signals for dataset synthesis and retrospective improvement.
C.2. Layer 2: Self-Evolution Protocol Layer
The Self-Evolution Protocol Layer (SEPL) specifies how an agentic system can improve itself through a principled closedloop operator interface. SEPL frames self-improvement as iterative state transitions over a heterogeneous evolvable state, while routing all modifications through standardized RSPL interfaces so that updates remain auditable (versioned), reversible (restorable), and safe by construction.
C.2.1. OVERVIEW
SEPL conceptualizes continuous improvement as a generalized optimization problem over a structured evolvable state space. Formally, SEPL treats evolutionary dynamics as state transitions governed by a strictly typed operator algebra, enabling different optimization strategies to share the same mutation surface and safety/verification gates. In our system, SEPL admits multiple instantiations--including reflection-driven optimization (our default), TextGrad (Yuksekgonul et al., 2025), GRPO (Shao et al., 2024), and Reinforce++ (Hu, 2025b). We do not expand their full mechanics in this overview; instead, we summarize their variables, operators, and loop procedures in dedicated subsections below.
C.2.2. EVOLVABLE VARIABLES
SEPL relies on variable lifting to project heterogeneous RSPL resources (e.g., prompts, tool implementations (native scripts, MCP tools, or agent skills), and memory modules) into a unified evolvable variable space. This abstraction provides a common interface for all evolution operators and makes the learnable subspace explicit via a binary learnability mask. We refer readers to the main text (SEPL, Evolvable Variables, Definition "Evolvable Variable Set") for the formal definition of Vevo and the associated learnability constraint.

20

Autogenesis: A Self-Evolving Agent Protocol
C.2.3. OPERATOR ALGEBRA
SEPL formalizes evolution as a composition of typed operators over auxiliary spaces, aligning with the canonical phases of iterative optimization (observation, attribution, proposal, verification, and commit). We adopt the reflectiondriven instantiation in the main text as the canonical example: it specifies a minimal operator suite {, , , , } (Reflect/Select/Improve/Evaluate/Commit) operating over the trace, hypothesis, modification, objective, and evaluation spaces (Z, H, D, G, S). We refer readers to the main text (Operator Algebra) for the formal operator signatures and their semantics; below we provide method-specific operatorizations for TextGrad, GRPO, and Reinforce++ in addition to reflection.
C.2.4. REFLECTION OPTIMIZER
Evolvable Variables. In the reflection-driven instantiation, the evolvable state is given by the lifted variable set Vevo introduced in the main text (SEPL, Evolvable Variables). Concretely, Vevo includes RSPL-managed resources (e.g., prompts, tools, memories, and agent components) together with execution artifacts (e.g., the produced answer and reasoning trace). A binary learnability mask specifies which variables may be modified, allowing the optimizer to target only authorized components while keeping non-learnable resources fixed.
Operator Algebra. We instantiate SEPL with the canonical reflection-driven operator suite in the main text (Operator Algebra). For completeness, we restate the operator signatures and their intended roles below.
� Reflect (). Defined as  : Z � Vevo  (H), this operator bridges the gap between raw observation and optimization direction. It approximates the "semantic gradient" of the system by mapping high-dimensional execution traces to specific, causal failure hypotheses within the variable space.
� Select (). Formulated as  : Vevo � (H)  (D), this operator acts as the generative policy. It translates diagnostic hypotheses into concrete update proposals, sampling candidate modifications D designed to minimize the identified error signal subject to structural constraints.
� Improve (). The mutation operator,  : Vevo � (D)  Vevo, executes the physical state transition. It applies discrete updates D via standardized RSPL interfaces to yield a provisional candidate state.
� Evaluate (). Specified as  : Vevo � G  S, this operator serves as the objective function. It maps the candidate state and goal specification to the evaluation space S (comprising quantitative scores and strict safety invariants).
� Commit (). Operating as  : Vevo � S  Vevo, this function acts as a conditional gating mechanism. It utilizes the evaluation signals in S to govern state transition, rigorously enforcing safety invariants and performance monotonicity by accepting the candidate Vevo only when specific success criteria are met.
The Evolutionary Loop. These operators are composed into the reflection-driven closed-loop procedure shown in Algorithm 2. Starting from an initial lifted state Ve(v0o), the agent first executes to collect an observational trace Z (tool outputs, intermediate decisions, failures, and progress signals). The reflect operator  maps Z to a set of causal hypotheses H, which are then translated by  into concrete modification primitives D (e.g., prompt edits, tool adjustments, or memory updates) over the learnable subset of Vevo. The improve operator  applies D via RSPL interfaces to obtain a candidate state, which is evaluated by  to produce S capturing both performance metrics and safety constraints. Finally, the commit operator  gates the transition by accepting only candidates that satisfy the predefined criteria, recording each accepted change as a versioned resource update with auditable lineage and enabling rollback when necessary.
C.2.5. TEXTGRAD OPTIMIZER
Evolvable Variables. In the TextGrad instantiation, the evolvable variables are restricted to a subset of prompt variables marked as optimizable and lifted into TextGrad variables with explicit role descriptions. In our implementation, each optimizable prompt module is represented as a TextGrad variable whose value is the current prompt text and whose role description specifies the prompt's function, enabling the optimizer to condition updates on its intended semantics.
Operator Algebra. TextGrad instantiates SEPL with a prompt-level operatorization in which "gradients" are naturallanguage critiques produced by an LLM evaluator and updates are implemented as constrained prompt rewrites. Following the standard TextGrad view, we express the method with five core operators, namely Execute, Loss, Backward, Improve, and Commit, where the "gradient" is a piece of text (a critique) rather than a numeric vector:
� Execute (tg). tg : (A, Vevo, x, f )  Z runs the agent under the current prompt variables and produces an execution trace/outcome.
� Loss (tg). tg : Z  Gtg, where Gtg is a space of natural-language critiques (textual gradients). In our implementation,
21

Autogenesis: A Self-Evolving Agent Protocol

Algorithm 2 Reflection Optimizer Evolutionary Loop

Input: Agentic System A, Objective G, Budget T Output: Optimized state Vevo 1: Initialization: 2: Ve(v0o)  VariableLifting(A) 3: Z(0)  Execute(A, Ve(v0o))

4: Optimization Cycle:

5: for t = 0, 1, . . . , T - 1 do

6: // Phase 1: Diagnosis & Proposal 7: H(t)  (Z(t), Ve(vto)) 8: D(t)  (Ve(vto), H(t))

9: // Phase 2: Mutation & Verification 10: Ve(vto+1)  (Ve(vto), D(t)) 11: S(t+1)  (Ve(vto+1), G)

12: // Phase 3: Gating & Transition 13: if Accept(S(t+1)) then

14: // Accept: safe & non-degrading

15:

Ve(vto+1)  (Ve(vto+1), S(t+1))

16: else

17: // Reject: rollback / keep previous state

18:

Ve(vto+1)  Ve(vto)

19: end if

20: // Phase 4: Next Iteration 21: Z(t+1)  Execute(A, Ve(vto+1)) 22: if Converged(S(t+1)) then 23: break 24: end if 25: end for 26: return Ve(vto)

 Project resources to optimization manifold  Trace: tool I/O, failures, latencies, progress
 Reflect: attribute failures / inefficiencies  Select: propose edits over learnable variables  Improve: apply proposed updates (candidate)
 Evaluate: metrics + safety invariants
 Commit: versioned update
 Re-run under updated resources

tg is realized by TextLoss, which queries an evaluator LLM and returns critique feedback. � Backward (tg). tg : Vevo � Gtg  Vevo assigns textual gradients to optimizable prompt variables by storing the critique
(optionally with context) in a per-variable gradient buffer. In our current implementation, we distribute the same critique to each optimizable prompt variable for stability. � Improve (tg). tg : Vevo  Vevo rewrites prompt variables via a textual-gradient-descent step: it constructs an update instruction from each variable's role description, current value, and accumulated textual gradients, then queries an optimizer LLM and extracts the improved variable text from a constrained output format. � Commit (tg). tg : Vevo  Vevo synchronizes the updated prompt variables back into the running agent and clears caches, completing the state transition.
The Evolutionary Loop. Algorithm 3 presents the full TextGrad optimization cycle in operator form. At each iteration, the agent is executed under the current prompt variables to obtain a trace Z via tg, an LLM-based evaluator produces a natural-language critique g  Gtg via tg, the critique is assigned as a textual gradient to the optimizable prompt variables via tg, the prompt variables are improved via tg using textual-gradient-descent, and the candidate state is committed via tg to synchronize the updated prompts back into the running agent (and clear caches) before the next iteration.
C.2.6. REINFORCE++ OPTIMIZER
Evolvable Variables. Reinforce++ optimizes a trainable subset of RSPL resources, focusing on prompt variables and tool implementations (native scripts, MCP tools (Anthropic, 2025a), and agent skills (Anthropic, 2025b)), and optionally refining the produced solution text. Our implementation follows a two stage structure: (i) update trainable variables that govern behavior (e.g., prompts and tools), and (ii) update the solution itself when enabled.
22

Autogenesis: A Self-Evolving Agent Protocol

Algorithm 3 TextGrad Prompt Optimization Loop

Input: Agentic System A, task x, attachments f (optional), Budget K, evaluator/optimizer LLMs Meval, Mopt Output: Updated state Vevo (prompt variables updated via TextGrad)
1: // Phase 0: Setup

2: Set backward engine to Meval 3: Ve(v0o)  VariableLifting(A)
4: Initialize textual optimizer with Mopt

 Evaluator used by TextLoss  Lift optimizable prompts to TextGrad variables
 TextualGradientDescent over prompt vars

5: // Optimization Cycle

6: for k = 0, 1, . . . , K - 1 do

7: // Phase 1: Execute (Forward) 8: Z(k)  tg(A, Ve(vko), x, f )

 Run agent with current prompts

9: // Phase 2: Loss (Textual Gradient) 10: Build evaluation instruction from Z(k) 11: g(k)  tg(Z(k))

 Condition on success/error  TextLoss produces critique string

12: // Phase 3: Backward (Assign Gradients) 13: Ve(vko)  tg(Ve(vko), g(k))

 Assign critique to gradient buffers

14: // Phase 4: Improve (Textual Gradient Descent) 15: Ve(vko+1)  tg(Ve(vko))

 Rewrite prompts via textual GD

16: // Phase 5: Commit & Next Iteration 17: Ve(vko+1)  tg(Ve(vko+1)) 18: if Converged(g(k)) then

 Sync back; clear caches

19: break

20: end if

21: end for 22: return Ve(vko)

Operator Algebra. Reinforce++ is characterized by a clipped objective with an explicit penalty to a reference solution, while using reflection to translate RL signals into concrete edits. We group the method into a small set of core operators:
� Sample (rpp). rpp : (A, Vevo, x, f )  Z samples a rollout under the current resources and yields an execution trace containing the produced answer.
� Reward (rpp). rpp : (y(t), y(t-1), y, ysft)  (r(t), A(t), J (t), (t)) computes the RL signal tuple from the current solution y(t). Here r(t) is a task reward comparing y(t) with y, and (t) is a ratio surrogate defined by a text similarity (�, �) as (t)  (y(t-1), y(t)). We define a penalty to a reference solution ysft as pen(t)   log max((ysft, y(t)), 0) and set A(t)  r(t) - pen(t). The clipped Reinforce++ objective is
J (t)  min (t)A(t), �(t)A(t) , �(t)  clip((t), 1 - , 1 + ).
� Reflection (rpp). rpp : (Z, Vtrain, r(t), A(t), J (t), (t))  H produces an edit oriented diagnosis that is explicitly conditioned on the RL metrics and the execution trace.
� Improve (rpp). rpp : (V, H)  Vevo applies RL informed edits to either (i) the trainable resources Vtrain such as prompts and tools, or (ii) the solution variable itself when solution refinement is enabled, yielding a candidate state.
� Commit (rpp). rpp : Vevo  Vevo applies accepted updates back to RSPL resources, completing the state transition.
The Evolutionary Loop. Algorithm 4 summarizes the Reinforce++ loop in a phased form. Each iteration (i) computes Reinforce++ signals via the clipped objective and the penalty to the reference solution, (ii) improves trainable resources through RL conditioned reflection and edits, (iii) optionally improves the solution text, and (iv) applies an early stopping evaluation.
C.2.7. GRPO OPTIMIZER
Evolvable Variables. GRPO optimizes a trainable subset of RSPL resources, focusing on prompt variables and tool implementations (native scripts, MCP tools (Anthropic, 2025a), and agent skills (Anthropic, 2025b)), and optionally refining the produced solution text. Similar to Reinforce++, our implementation follows a two stage structure: (i) update trainable
23

Autogenesis: A Self-Evolving Agent Protocol

Algorithm 4 Reinforce++ Optimization Loop

Input: Agentic System A, task x, ground truth y, reference solution ysft, Budget T Output: Final solution y(t) and updated trainable resources Vtrain
1: // Initialization 2: Ve(v0o)  VariableLifting(A) 3: Z(0)  rpp(A, Ve(v0o), x, f ) 4: Extract solution y(0) from Z(0) 5: y(-1)  y(0)
6: for t = 0, 1, . . . , T - 1 do

 Lift trainable resources  Sample once
 Initialize previous solution

7: // Phase 1: Reinforce++ reward and objective 8: (r(t), A(t), J (t), (t))  rpp(y(t), y(t-1), y, ysft)
9: // Phase 2: Improve trainable resources (prompt and tool) 10: Vt(rta)in  GetTrainables(Ve(vto)) 11: Ht(rta)in  rpp(Z(t), Vt(rta)in, r(t), A(t), J (t), (t)) 12: Vt(rta+in1)  rpp(Vt(rta)in, Ht(rta)in) 13: Vt(rta+in1)  rpp(Vt(rta+in1))
14: // Phase 3: Re run under updated resources 15: Z(t+1)  rpp(A, Ve(vto)  Vt(rta+in1), x, f ) 16: Extract solution y(t+1) from Z(t+1)

 Reward, penalty, clipped objective
 Reflection conditioned on RL signals  Apply edits to trainables (candidate)
 Commit updates

17: // Phase 4: Optional solution refinement 18: Hs(otl)  rpp(Z(t+1), {y(t+1)}, r(t), A(t), J (t), (t)) 19: y(t+1)  rpp(y(t+1), Hs(otl)) 20: y(t+1)  rpp(y(t+1))
21: // Phase 5: Early stopping 22: if Satisfied(Z(t+1)) then
23: break
24: end if 25: y(t)  y(t+1)
26: end for 27: return y(t)

 Reflect on solution quality  Edit solution text (candidate)
 Commit solution update
 Advance current solution

variables that govern behavior (e.g., prompts and tools), and (ii) update the solution itself when enabled.
Operator Algebra. GRPO is characterized by sampling multiple candidate solutions per step and using group normalized advantages with a clipped objective. We formalize the method with the following core operators:
� Sample (grpo). grpo : (A, Vevo, x, f, K)  {Zi}Ki=1 samples K independent rollouts under the current resources, yielding K execution traces each containing a candidate solution yi.
� Reward (grpo). grpo : ({yi}Ki=1, y, y(t-1))  ({ri}Ki=1, {Ai}iK=1, {Ji}Ki=1, {i}iK=1) computes RL signals for all K candidates. For each candidate yi, we compute a task reward ri comparing yi with y, a policy ratio surrogate i  (y(t-1), yi) using text similarity (�, �), and a group normalized advantage Ai by normalizing rewards across the candidate set: Ai = (ri - r�)/r where r� and r are the mean and standard deviation of {ri}iK=1. The GRPO clipped objective for each candidate is

Ji  min iAi, �iAi ,

�i 

min(i, 1 + ) max(i, 1 - )

if Ai  0 . if Ai < 0

� Reflection (grpo). grpo : ({Zi}Ki=1, Vtrain, {ri, Ai, Ji, i}iK=1)  H produces an edit oriented diagnosis that is explicitly conditioned on the multiple candidate solutions and their RL metrics, enabling the optimizer to identify patterns across
candidates. � Improve (grpo). grpo : (V, H)  Vevo applies RL informed edits to either (i) the trainable resources Vtrain such as
prompts and tools, or (ii) the solution variable itself when solution refinement is enabled, yielding a candidate state.

24

Autogenesis: A Self-Evolving Agent Protocol

Algorithm 5 GRPO Optimization Loop

Input: Agentic System A, task x, ground truth y, Budget T , number of candidates K Output: Final solution y(t) and updated trainable resources Vtrain
1: // Initialization 2: Ve(v0o)  VariableLifting(A) 3: Z(0)  grpo(A, Ve(v0o), x, f, 1) 4: Extract solution y(0) from Z(0) 5: y(-1)  y(0)
6: for t = 0, 1, . . . , T - 1 do

 Lift trainable resources  Sample initial solution
 Initialize previous solution

7: // Phase 1: Sample multiple candidates

8: {Zi(t)}Ki=1  grpo(A, Ve(vto), x, f, K) 9: Extract candidate solutions {yi(t)}Ki=1 from {Zi(t)}iK=1

 Sample K rollouts

10: // Phase 2: GRPO reward and objective

11: ({ri(t)}iK=1, {A(it)}iK=1, {Ji(t)}iK=1, {i(t)}Ki=1)  grpo({yi(t)}iK=1, y, y(t-1))

 Group normalized advantages, clipped objectives

12: // Phase 3: Improve trainable resources (prompt and tool)

13: Vt(rta)in  GetTrainables(Ve(vto)) 14: Ht(rta)in  grpo({Zi(t)}Ki=1, Vt(rta)in, {ri(t), A(it), Ji(t), (it)}iK=1) 15: Vt(rta+in1)  grpo(Vt(rta)in, Ht(rta)in) 16: Vt(rta+in1)  grpo(Vt(rta+in1))

 Reflection conditioned on multi candidate RL signals  Apply edits to trainables (candidate)  Commit updates

17: // Phase 4: Re run under updated resources

18: Z(t+1)  grpo(A, Ve(vto)  Vt(rta+in1), x, f, 1) 19: Extract solution y(t+1) from Z(t+1)

20: // Phase 5: Optional solution refinement 21: Hs(otl)  grpo({Zi(t)}iK=1, {y(t+1)}, {ri(t), A(it), Ji(t), (it)}Ki=1) 22: y(t+1)  grpo(y(t+1), Hs(otl)) 23: y(t+1)  grpo(y(t+1))
24: // Phase 6: Early stopping 25: if Satisfied(Z(t+1)) then
26: break
27: end if 28: y(t)  y(t+1)
29: end for 30: return y(t)

 Reflect on solution quality using multi candidate context  Edit solution text (candidate)  Commit solution update
 Advance current solution

� Commit (grpo). grpo : Vevo  Vevo applies accepted updates back to RSPL resources, completing the state transition.
The Evolutionary Loop. Algorithm 5 summarizes the GRPO loop in a phased form. Each iteration (i) samples K candidate solutions, (ii) computes GRPO signals via group normalized advantages and clipped objectives, (iii) improves trainable resources through multi candidate conditioned reflection and edits, (iv) optionally improves the solution text, and (v) applies an early stopping evaluation.

25

