Fractal
We’re building a browser-based, zero-install compute fabric.
You create a room, others join by link, and we distribute micro-tasks over WebRTC to their browsers. WebAssembly/Web Workers run compute locally and send results back.

The system is designed for churn: heartbeats + leases + reassignment. It’s not a cloud replacement, it's for short compute bursts in hackathons/classrooms/rapid ML experimentation, where cloud setup/cost is a blocker. 

The engineering challenge is the scheduler + fault tolerance + measurable speedup demo.


Step-by-step walkthrough to explain Fractal
1) What Fractal is
Fractal is a zero-install, browser-based “temporary compute cluster” that turns devices in a room (phones/laptops) into a single compute pool for short AI/ML jobs.
2) The mental model 
You create a “room” (like a meeting link).
Others join by opening the link (no app install).
Their browsers connect P2P (WebRTC).
Your laptop acts as the scheduler/master.
Work is split into small tasks and sent to peers.
Peers run compute (WASM/Web Workers) and return results.
If someone leaves, tasks are reassigned automatically.

The real-world problem it solves 
Problem A: Short compute bursts are expensive or slow
Hyperparameter sweep / batch inference / embeddings / simulations can take 30–60 mins on a single laptop.
Cloud solves it but costs money + setup time + accounts + quotas.
Problem B: Setup friction kills collaboration
Installing agents, setting up clusters, or sharing environments is slow.
In hackathons/classrooms/workshops, people won’t install anything.
Problem C: Sometimes data can’t/shouldn’t go to cloud
Privacy, policy, no internet, or time pressure.
Need compute locally but faster than one laptop.
Fractal’s niche: instant, free, local compute burst with zero install.
Not cloud replacement.

How Fractal solves those problems
Core design choices
Zero-install: runs entirely in the browser.
Ephemeral: the cluster exists only while people are present.
P2P compute path: tasks/results move over WebRTC DataChannels (not via a server).
Small, idempotent tasks: tasks are micro-batches so failures are cheap.
Adaptive scheduling: faster devices get more work; slow ones get less.
Fault tolerance by default: heartbeats + task leasing + automatic reassignment.

tech stack 
Browser (Master + Worker)
WebRTC DataChannels: peer-to-peer data transfer
Web Workers: parallelism inside each browser
WebAssembly (WASM): fast compute kernels
React/Next.js (optional): UI
Control plane (coordination, not compute)
Cloudflare Workers + Durable Objects (free):
room state (who joined)
signaling relay (offer/answer exchange)
minimal job/task metadata

Workloads (choose one for capstone)
Hyperparameter sweep (best demo)
Batch embedding generation
Monte Carlo simulation

LLD (Low Level Design) 
1) Modules
A) Control Plane (Cloudflare Durable Object)
RoomManagerDO
createRoom()
joinRoom(peerId)
leaveRoom(peerId)
postSignal(peerId, sdp/ice)
fetchSignals(peerId)
heartbeat(peerId, metrics)
updatePeerBench(peerId, benchScore)
B) Master (Laptop)
PeerConnector
establishes WebRTC connections to each worker
TaskSlicer
converts a job into micro-tasks
Scheduler
assigns tasks adaptively
LeaseManager
tracks leases, retries, timeouts
ResultAggregator
merges results and updates UI
Visualizer
shows peers + throughput + failures
C) Worker (Phone/laptop browser)
JoinClient
join room + signaling
WasmRunner / WorkerPool
executes compute
HeartbeatClient
sends alive + metrics
TaskExecutor
receives tasks, runs, returns results

2) Data structures
Peer state
{
  "peerId": "p1",
  "status": "online",
  "lastSeen": 1730001111,
  "benchScore": 0.42,
  "avgTaskMs": 850,
  "failRate": 0.08
}

Task
{
  "taskId": "t42",
  "jobId": "job1",
  "payload": { "params": { "lr": 0.01 }, "shard": 3 },
  "status": "leased",
  "lease": { "to": "p1", "until": 1730001120 },
  "attempt": 1
}

Result
{
  "taskId": "t42",
  "metrics": { "durationMs": 840, "accuracy": 0.83 },
  "output": { "loss": 0.42 }
}


3) Critical flows (LLD-style)
Flow 1: Room creation + join
Master calls createRoom() → gets roomId + link
Worker opens link → joinRoom(peerId)
DO stores peer + returns list of peers + signaling endpoints
Master/Worker exchange SDP/ICE via DO
WebRTC DataChannel connects
Flow 2: Benchmark-on-join
Master sends BENCH_REQUEST
Worker runs quick compute (200–500ms)
Worker returns BENCH_RESULT(ops/sec)
Master updates benchScore in scheduler
Flow 3: Task scheduling + execution
Job arrives → TaskSlicer creates micro-tasks
Scheduler assigns tasks:
chunk size ∝ benchScore
Worker executes and returns result
Master commits result, marks task done
Flow 4: Failure handling (tab closed)
Heartbeat missing for N seconds → peer offline
All tasks leased to that peer:
lease expires → requeue
Scheduler reassigns tasks to other peers

Questions (where they think Fractal fails) + solutions
Q1) “What if someone closes the tab? Everything breaks.”
Answer: We design for churn.
Heartbeats every 2–3s
Task leasing (e.g., 8s lease)
If heartbeat stops → peer offline
Leased tasks auto-requeue and retry
 This is exactly how MapReduce/Spark handle worker loss.

Q2) “Phones are slow; network latency will kill performance.”
Answer: We only target workloads where distributed compute makes sense.
Embarrassingly parallel workloads
Micro-batch tasks (few KB payloads, seconds compute)
Adaptive chunk sizing:
slow peers get small tasks
fast peers get larger tasks
Avoid heavy data transfer
We benchmark and schedule based on real speed.

Q3) “This doesn’t scale to 1000 devices.”
Answer: We don’t claim that.
Our target is human-scale (3–15 devices)
Scaling beyond that needs overlay networks (future work)
The project is a systems prototype proving a design point
Small scale is sufficient to prove novelty + engineering.

Q4)  “Isn’t this just BOINC?”
Answer: No. BOINC assumes installs + long-lived trust.
Fractal:
zero-install
ephemeral session
P2P compute
churn-first design
interactive workloads
 Different lifecycle + trust model.

Q5) “What about privacy? Are we sending data to everyone?”
Answer: We control data scope.
Default: use non-sensitive datasets
Only share task shards needed for compute
Room access via link + optional passcode
No cloud storage of raw data
Threat model is cooperative (hackathon/classroom), not adversarial internet.

Q6) “What if the master laptop closes?”
Answer: Session ends intentionally.
Fractal is ephemeral and human-bounded.
If master leaves, room dissolves.
(Optional extension)
Add “backup master” election, but not required for MVP.

Q7) “Battery/thermal throttling on phones will make it unstable.”
Answer: We adapt.
Track task duration trend per peer
If a peer slows, reduce chunk size
Add a “contribution slider” (light/medium/heavy)
Allow opt-out at any time
 The scheduler learns device behavior live.

Q8) “WebRTC is hard; will we finish in 1 month?”
Answer: Yes if we narrow scope.
Star topology: Master connects to each worker
DO only handles signaling + registry
Start with CPU-only JS compute; add WASM as upgrade
MVP first, then performance upgrades.


