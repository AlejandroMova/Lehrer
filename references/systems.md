# System Reference — Common Architectures

Use this file when mapping a feature onto a known system type. Match the user's stack to the closest pattern, then adapt.

---

## Embedded / Edge AI Pipelines (e.g. NVIDIA DeepStream, GStreamer)

**Central metaphor:** a pipeline of elements through which buffers flow. Each element transforms the buffer or attaches metadata to it.

**Data flow spine:**
```
Source → Decode → Inference (PGIE) → Tracker → Secondary Inference (SGIE) → Analytics → Sink
```

**Key abstractions:**
- **Element** — a single processing node (decoder, inference engine, tiler, OSD)
- **Pad** — input (sink pad) or output (src pad) of an element; elements connect via pads
- **Buffer** — a unit of data flowing through the pipeline (typically a frame or batch)
- **Metadata (NvDsMeta)** — structured data attached to a buffer; how downstream elements see upstream results
- **Probe** — a callback attached to a pad; the primary extension mechanism for custom logic

**Where features typically hook in:**
- Custom logic → pad probe on the sink or src pad of the relevant element
- New inference → new nvinfer element with a config file, inserted at the right point in the pipeline
- New analytics → Python probe reading NvDsObjectMeta and writing custom metadata or events
- New output → new sink element or a probe that writes to an external system

**Common gotchas:**
- Element order in the pipeline is execution order — inserting in the wrong place means wrong data
- NvDsMeta is mutable; probes upstream of you may have already modified it
- DeepStream operates on batches of frames — index into the batch correctly
- GLib main loop must be running for pipeline events to process; blocking the main thread stalls everything
- `sys.exit()` inside a probe callback will not cleanly shut down the pipeline

**Reading order for an unfamiliar DeepStream codebase:**
1. The pipeline builder (usually `pipeline.py` or `main.py`) — understand the element chain
2. The config files (`.cfg`, `.yaml`) — understand what each inference element is configured to do
3. The probe callbacks — understand what custom logic is applied and at what stage
4. Any metadata schema definitions — understand what the downstream consumers expect

---

## Web Backends (FastAPI, Flask, Django, Express)

**Central metaphor:** request enters, passes through middleware, hits a route handler, returns a response.

**Data flow spine:**
```
Client → Middleware stack → Router → Handler → Service/DB → Response
```

**Key abstractions:**
- **Middleware** — wraps every request; order of registration matters (often reversed execution)
- **Router** — groups related routes; prefix and dependencies applied at registration
- **Handler / View** — the function that processes a specific route; should be thin
- **Service layer** — business logic; should be independent of HTTP concerns
- **Schema / Model** — the shape of data in (request) and out (response); validation lives here

**Where features typically hook in:**
- New endpoint → new route in the appropriate router, thin handler calling service layer
- New behavior on all requests → middleware (auth, logging, rate limiting, CORS)
- New data shape → new Pydantic model / schema
- New DB interaction → new query or ORM method in the service or repository layer

**Common gotchas:**
- FastAPI applies middleware in reverse registration order — CORS must be registered last in code to run first
- Dependency injection in FastAPI is per-request by default; shared state needs explicit lifetime management
- Django ORM queries are lazy — a queryset is not executed until iterated or explicitly evaluated
- `async` route handlers in FastAPI run in an async event loop; blocking I/O inside them will stall other requests

**Reading order for an unfamiliar web backend:**
1. The entrypoint (`main.py`, `app.py`, `wsgi.py`) — understand the app object, middleware stack, and router registration
2. The route files — understand the URL surface
3. The schema/model files — understand the data contracts
4. The service layer — understand the business logic

---

## Data Processing Pipelines (batch ETL, stream processing)

**Central metaphor:** data moves through stages that extract, transform, and load it.

**Data flow spine:**
```
Source → Extract → Transform(s) → Validate → Load → Sink
```

**Key abstractions:**
- **Stage** — a single transformation or I/O step
- **Schema** — the expected shape of data at each stage boundary
- **Idempotency** — whether re-running a stage produces the same result; critical for retry logic
- **Checkpointing** — where in the pipeline the system can restart after a failure

**Where features typically hook in:**
- New data source → new extractor that produces the expected schema
- New transformation → new stage inserted at the right point in the chain
- New output → new loader or sink
- New validation rule → added to the validate stage

**Common gotchas:**
- Transformations that look stateless often aren't (e.g. they read from a cache or external API)
- Schema mismatches between stages surface at runtime, not at import time — test with real data shapes
- Ordering of transformations matters when they share state or mutate in-place

---

## Robotics / ROS 2

**Central metaphor:** nodes communicate by publishing and subscribing to named topics, or calling services.

**Data flow spine:**
```
Sensor node → Topic → Processing node → Topic → Action/Control node
```

**Key abstractions:**
- **Node** — an independent process with its own lifecycle, parameters, publishers, and subscribers
- **Topic** — a named channel; publishers write, subscribers read; many-to-many
- **Service** — synchronous request/response between two nodes
- **Action** — long-running goal with feedback; cancelable
- **Parameter** — runtime-configurable value scoped to a node

**Where features typically hook in:**
- New sensor integration → new node that publishes on a topic
- New processing step → new node that subscribes to input topic, publishes to output topic
- New control behavior → new subscriber on the relevant topic or a new service client

**Common gotchas:**
- ROS 2 nodes are not threads — they have their own executor; don't call `spin()` from inside a callback
- QoS (Quality of Service) settings must match between publisher and subscriber; mismatches produce silent data loss
- Parameters must be declared before they can be read; missing declarations raise at runtime
- Clock time in ROS 2 is separate from wall time — use `self.get_clock().now()` for reproducible simulation

---

## ML Training / Inference Systems

**Central metaphor:** data passes through a model graph; training adjusts weights, inference runs the forward pass.

**Key abstractions (PyTorch):**
- **Dataset / DataLoader** — how data is loaded and batched
- **Module** — a neural network component with `forward()` defining its computation
- **Optimizer** — updates weights based on gradients
- **Loss function** — measures prediction error; the gradient flows backward through it

**Where features typically hook in:**
- New data source → new Dataset class implementing `__len__` and `__getitem__`
- New model component → new Module subclass with a `forward()` method
- New training behavior → hooks into the training loop (callbacks, custom loss, gradient clipping)
- New inference mode → wrapping the model in `torch.no_grad()` with appropriate pre/post-processing

**Common gotchas:**
- Tensors must be on the same device (CPU/GPU) for operations to work — device mismatches raise at runtime
- `model.train()` and `model.eval()` affect BatchNorm and Dropout — forgetting to switch causes subtle eval errors
- Gradients accumulate by default; call `optimizer.zero_grad()` before each backward pass
- DataLoader `num_workers > 0` spawns subprocesses — avoid non-serializable objects in Dataset
