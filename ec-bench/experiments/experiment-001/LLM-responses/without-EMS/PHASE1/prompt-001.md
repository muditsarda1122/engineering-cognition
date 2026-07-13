model: kimi k2.5

Here is the breakdown of those four modules and the single most important entry point in each that an incoming HTTP request flows through.
fastapi/routing.py — Responsibility: Wires an endpoint function into an ASGI handler  
This module defines APIRoute and APIWebSocketRoute, which extend Starlette’s routing with FastAPI-specific features: dependency injection, request/response validation, serialization, and auto-generated OpenAPI metadata.  
Most important function: get_request_handler  
It returns the actual async app(request) -> Response coroutine that Starlette will invoke. Every HTTP request that matches a FastAPI route flows through this function: it reads the body, calls solve_dependencies, executes the endpoint function, serializes the response, and handles generator-based streaming (JSONL, SSE, raw).
fastapi/dependencies/utils.py — Responsibility: Discovers and resolves the dependency graph  
This module inspects endpoint signatures to build a Dependant tree (via get_dependant and analyze_param), then recursively resolves sub-dependencies, caches them, and injects the final keyword arguments into the endpoint.  
Most important function: solve_dependencies  
It is called by get_request_handler (and get_websocket_app) on every request. It walks the dependency tree, runs sub-dependencies (async, sync, or generator-based), validates path/query/header/cookie/body params, and returns a SolvedDependency with the final values dict that is unpacked as **kwargs into the endpoint function.
fastapi/applications.py — Responsibility: The public FastAPI app and ASGI lifecycle  
This module defines the FastAPI class (subclass of Starlette). It bootstraps the router, middleware stack, exception handlers, and the automatic /docs, /redoc, and /openapi.json endpoints.  
Most important function: FastAPI.__call__  
This is the ASGI entry point: scope, receive, send enter here first. It delegates to the Starlette/ASGI machinery, which eventually dispatches to APIRoute.handle and then to the get_request_handler app. Every single HTTP request (including docs and schema requests) passes through this method.
fastapi/openapi/utils.py — Responsibility: Generates the OpenAPI schema from the route definitions  
This module contains the introspection logic that turns APIRoute objects, Dependant trees, and Pydantic models into a valid OpenAPI JSON document. It is not on the hot path of normal path-operation requests, but it is on the hot path for the built-in schema endpoint.  
Most important function: get_openapi  
It is invoked (via FastAPI.openapi()) when a request hits /openapi.json (or the custom openapi_url). It iterates over all routes, collects fields, generates schemas, and assembles the final OpenAPI document that is returned to the client.