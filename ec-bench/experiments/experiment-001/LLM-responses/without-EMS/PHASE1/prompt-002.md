model: kimi k2.5

Here is the exact call stack for GET /items/ from the first byte entering the process to the last byte leaving.
Phase 0 — Import-time wiring (happens once when the module loads)
```
# When you write @app.get("/items/")
FastAPI.add_api_route()          # applications.py:1195
  └─ APIRouter.add_api_route()   # routing.py:2838
       └─ APIRoute.__init__()     # routing.py:1146
            ├─ _populate_api_route_state()       # routing.py:978
            │   ├─ _build_dependant_with_parameterless_dependencies()  # routing.py:839
            │   │   └─ get_dependant()            # dependencies/utils.py:286
            │   └─ get_body_field()               # dependencies/utils.py:1005
            │
            └─ self.app = request_response(self.get_route_handler())   # routing.py:1208
                 └─ get_route_handler()            # routing.py:1210
                      └─ get_request_handler(...)  # routing.py:1217
                           └─ returns the inner `async def app(request)` closure (line 398)
                 └─ request_response(handler)      # routing.py:113
                      └─ returns the outer ASGI app closure (line 126)
```

So at import time self.app on the APIRoute instance is already a fully-wrapped ASGI callable.

Phase 1 — First request lazily builds the middleware stack
```
1. ASGI server calls
   app(scope, receive, send)

2. FastAPI.__call__          # applications.py:1160
   └─ sets scope["root_path"]
   └─ await super().__call__(scope, receive, send)
        └─ Starlette.__call__  (inherited)
             └─ if self.middleware_stack is None:
                    self.build_middleware_stack()   # applications.py:1020
             └─ await self.middleware_stack(scope, receive, send)

build_middleware_stack wraps the stack like an onion (outer → inner):
ServerErrorMiddleware
  → user middlewares
    → ExceptionMiddleware
      → AsyncExitStackMiddleware   ← creates scope["fastapi_middleware_astack"]
        → APIRouter.handle          ← the innermost "app"
```

Phase 2 — Router matching
```
3. AsyncExitStackMiddleware.__call__  # fastapi.middleware.asyncexitstack
   └─ creates AsyncExitStack → scope["fastapi_middleware_astack"]
   └─ await APIRouter.handle(scope, receive, send)     # routing.py:2732
        └─ included_router check is false
        └─ await self.app(scope, receive, send)         # routing.py:2740  (calls APIRouter.app)

4. APIRouter.app               # routing.py:2668
   └─ iterates self.routes
   └─ for each route: route.matches(scope)
        └─ APIRoute.matches    # routing.py:1236
             └─ super().matches(scope)   # Starlette Route.matches — regex + method check
   └─ match == Match.FULL for GET /items/
   └─ scope.update(child_scope)   # injects path_params
   └─ await route.handle(scope, receive, send)   # routing.py:2683
```

Phase 3 — Route.handle enters the per-request context
```
5. APIRoute.handle             # routing.py:1246
   └─ effective_context is None for a normal route
   └─ await super().handle(scope, receive, send)   # Starlette Route.handle
        └─ await self.app(scope, receive, send)

   Remember: self.app was wired at import time to:
   request_response( get_request_handler(...) )
```

Phase 4 — request_response creates the two per-request exit stacks
```
6. request_response(handler)   # routing.py:113  (the closure created at import time)
   └─ outer app(scope, receive, send):
        └─ request = Request(scope, receive, send)       # line 127
        └─ await wrap_app_handling_exceptions(app, request)(scope, receive, send)
             └─ inner app(scope, receive, send):         # line 129
                  ├─ async with AsyncExitStack() as request_stack:
                  │     scope["fastapi_inner_astack"] = request_stack      # line 133
                  │     async with AsyncExitStack() as function_stack:
                  │        scope["fastapi_function_astack"] = function_stack  # line 135
                  │        response = await f(request)                        # line 136
                  │     await response(scope, receive, send)                  # line 137
                  │     response_awaited = True                               # line 139
                  └─ (after exiting both stacks, generator dependencies are cleaned up)
```

Phase 5 — get_request_handler processes the request
```
7. f(request) is the closure returned by get_request_handler(...)   # routing.py:367

   async def app(request):          # routing.py:398
      ├─ file_stack = request.scope["fastapi_middleware_astack"]   # line 400
      ├─ endpoint_ctx = _extract_endpoint_context(dependant.call)   # line 406
      ├─ body = None   (GET has no body, so body_field branch is skipped)
      ├─ async_exit_stack = request.scope["fastapi_inner_astack"]   # line 469
      │
      ├─ solved_result = await solve_dependencies(
      │      request=request,
      │      dependant=dependant,
      │      body=None,
      │      dependency_overrides_provider=...,
      │      async_exit_stack=async_exit_stack,
      │      embed_body_fields=...,
      │   )                                              # routing.py:473
      │
      │   (see Phase 6 below)
      │
      ├─ assert not errors                               # routing.py:483
      │
      ├─ raw_response = await run_endpoint_function(
      │      dependant=dependant,
      │      values=solved_result.values,
      │      is_coroutine=is_coroutine,
      │   )                                              # routing.py:690
      │   └─ await dependant.call(**values)             # routing.py:344
      │        └─ YOUR ENDPOINT FUNCTION RETURNS
      │             return [{"item_id": "Foo"}]
      │
      ├─ isinstance(raw_response, Response) is False
      │
      ├─ response_args = _build_response_args(status_code=..., solved_result=...)  # line 700
      │
      ├─ content = await serialize_response(
      │      field=response_field,
      │      response_content=raw_response,
      │      include=..., exclude=..., by_alias=..., ... ,
      │      dump_json=use_dump_json,
      │   )                                              # routing.py:711
      │   (see Phase 7 below)
      │
      ├─ response = Response(content, media_type="application/json", **response_args)
      │                                              # routing.py:725 or 731
      │
      └─ return response                                 # routing.py:743
```

Phase 6 — solve_dependencies resolves the dependency graph
```
8. solve_dependencies(request, dependant, ...)   # dependencies/utils.py:598
   └─ request_astack  = scope["fastapi_inner_astack"]     # line 612
   └─ function_astack = scope["fastapi_function_astack"]    # line 617

   For each sub_dependant in dependant.dependencies:        # line 628
      └─ recursively await solve_dependencies(...)         # line 649
      └─ if generator: await _solve_generator(...)         # line 672
      └─ elif async:  await call(**sub_values)             # line 678
      └─ else:        await run_in_threadpool(call, **sub_values)  # line 680

   Then validates request parameters:
   ├─ path_values,  path_errors  = request_params_to_args(path_params,  request.path_params)   # line 685
   ├─ query_values, query_errors = request_params_to_args(query_params, request.query_params)  # line 688
   ├─ header_values, header_errors = request_params_to_args(header_params, request.headers)    # line 691
   ├─ cookie_values, cookie_errors = request_params_to_args(cookie_params, request.cookies)    # line 694

   If body_params exist:
   └─ body_values, body_errors = await request_body_to_args(body_fields, received_body, embed) # line 706

   Injects special params:
   ├─ values[request_param_name] = request                # line 715
   ├─ values[background_tasks_param_name] = BackgroundTasks()  # line 721
   ├─ values[response_param_name] = response              # line 724

   └─ return SolvedDependency(
          values=values,
          errors=errors,
          background_tasks=background_tasks,
          response=response,
          dependency_cache=dependency_cache,
      )                                                  # line 729
```

Phase 7 — serialize_response validates and serializes the return value
```
9. serialize_response(...)     # routing.py:293
   └─ if response_field exists:
        ├─ value, errors = field.validate(response_content, {}, loc=("response",))   # line 309
        ├─ if errors: raise ResponseValidationError(...)
        └─ serializer = field.serialize_json if dump_json else field.serialize
             └─ serializer(value, include=..., exclude=..., by_alias=..., ...)      # line 322
                  └─ (Pydantic core serialization → JSON bytes or Python dict)
   └─ else:
        └─ return jsonable_encoder(response_content)    # line 333
```

Phase 8 — Response travels back up the stack
```
10. serialize_response returns JSON bytes (or dict)
    └─ back into get_request_handler's app(request)
         └─ response = Response(content, media_type="application/json", **response_args)
         └─ response.headers.raw.extend(solved_result.response.headers.raw)   # line 734
         └─ return response

11. Back in request_response inner app (routing.py:136):
    └─ response = await f(request)   ← now holds the Response object
    └─ await response(scope, receive, send)   # routing.py:137
         └─ Response.__call__ sends HTTP headers + body to the ASGI server

12. response_awaited = True   # routing.py:139
    Both AsyncExitStacks exit → generator dependencies' finally/teardown blocks run

13. Back through wrap_app_handling_exceptions
    └─ any unhandled exceptions would be converted to HTTP error responses here

14. Back through middleware stack (unwinding):
    ExceptionMiddleware → user middlewares → ServerErrorMiddleware
    └─ AsyncExitStackMiddleware exits its stack → closes any files opened during the request
```

Summary of the critical functions in order
#	Function	File	What it does
1	FastAPI.__call__	applications.py	ASGI entry, sets root_path
2	build_middleware_stack	applications.py	Lazily composes middleware onion on first request
3	APIRouter.handle	routing.py	Top-level router dispatch
4	APIRouter.app	routing.py	Iterates routes, calls matches(), picks the APIRoute
5	APIRoute.handle	routing.py	Delegates to Starlette Route.handle
6	request_response	routing.py	Creates Request + two AsyncExitStacks per request
7	get_request_handler	routing.py	Returns the closure that actually processes the request
8	solve_dependencies	dependencies/utils.py	Recursively resolves dependency graph + validates params
9	run_endpoint_function	routing.py	Calls your endpoint function with injected kwargs
10	serialize_response	routing.py	Validates return value against response model, serializes to JSON
11	Response.__call__	starlette	Sends the HTTP response back through ASGI
The two AsyncExitStacks created by request_response (fastapi_inner_astack and fastapi_function_astack) are the connective tissue that makes solve_dependencies's generator-based dependency cleanup possible — when the stacks unwind after await response(...), every dependency that used yield gets its teardown block executed.