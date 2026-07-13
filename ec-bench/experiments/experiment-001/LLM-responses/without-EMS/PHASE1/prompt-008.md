model: kimi k2.5

Here is the investigation.
Approach
I used a four-part strategy combining Python semantics verification, runtime instrumentation of an actual FastAPI route, closure introspection, and a micro-benchmark.
1. Micro-experiment — A pure-Python reproduction of get_request_handler → app(request) → nested def _serialize_data to establish baseline def semantics.
2. Runtime instrumentation — A custom APIRoute subclass that counts how many times get_request_handler is called and logs the identity (id()) of the returned closure across multiple requests.
3. Closure cell inspection — Directly reading the __closure__ cells of the actual streaming route handler to see what setup-time variables are captured once.
4. Allocation micro-benchmark — Timing 100,000 calls of a flat function vs. a function that creates a nested def on each call.
What was measured
- Whether get_request_handler itself is invoked once or per-request.
- Whether the returned app closure object is the same across requests.
- Whether a nested def inside a function body creates a new object on every call.
- The raw allocation overhead of that pattern.
Observations
Observation	Result
Factory calls (get_request_handler)	Once per route (at APIRoute.__init__ time)
App closure identity across 3 requests	Identical (id(handler) = same value for all requests)
Nested helper identity in micro-experiment	Different every call (h1 is h2 is h3 = False)
Nested helper code object	Shared (h1.__code__ is h2.__code__ = True)
Closure cells of actual route handler	20 cells, created once, containing route-static variables (Dependant, ModelField, booleans, DefaultPlaceholder)
Micro-benchmark overhead	~2.5× slower (nested-def: 26.9 ms vs. flat: 10.9 ms for 100K calls)
Hypothesis support
The observations strongly support the hypothesis:
- get_request_handler is a true factory: it runs once, and its output (the app closure) is reused for the lifetime of the route.
- However, any def statement inside app(request) — including _serialize_data, _serialize_sse_item, _serialize_item, _async_stream_jsonl, etc. — is re-executed on every request because app(request) itself runs per request.
- The overhead is object allocation + closure cell binding, not bytecode compilation (the __code__ object is shared).
Limitations
- We did not directly instrument _serialize_data inside the actual FastAPI route because it is a local variable inaccessible from outside the running function. The conclusion relies on an isomorphic micro-experiment.
- The 2.5× micro-benchmark measures pure Python overhead in isolation; the real FastAPI request path adds HTTP parsing, dependency resolution, and serialization, so the relative cost of closure creation may be smaller in context.
- The ratio is sensitive to Python version and CPython implementation details.