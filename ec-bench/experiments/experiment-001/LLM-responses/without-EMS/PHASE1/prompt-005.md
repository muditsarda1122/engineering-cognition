model: glm-5.2 (mistake)

Now I have everything needed. Let me trace every variable reference in each helper, distinguishing closure captures (from enclosing function scopes) from module-level global lookups.
Module-level imports confirmed (lines 1-108): json (7), anyio (40), ObjectReceiveStream (42), asynccontextmanager (26), AsyncIterator (12), jsonable_encoder (61), EndpointContext (63), ResponseValidationError (66), ServerSentEvent (73), format_sse_event (74), _PING_INTERVAL (70), KEEPALIVE_COMMENT (71).
1. _serialize_data (routing.py:487-510)
8 free variables from enclosing scopes:
#	Variable	Referenced at	Source scope	Classification
1	stream_item_field	lines 488, 489, 499	get_request_handler parameter (line 382)	Route-static
2	response_model_include	line 501	get_request_handler parameter (line 373)	Route-static
3	response_model_exclude	line 502	get_request_handler parameter (line 374)	Route-static
4	response_model_by_alias	line 503	get_request_handler parameter (line 375)	Route-static
5	response_model_exclude_unset	line 504	get_request_handler parameter (line 376)	Route-static
6	response_model_exclude_defaults	line 505	get_request_handler parameter (line 377)	Route-static
7	response_model_exclude_none	line 506	get_request_handler parameter (line 378)	Route-static
8	endpoint_ctx	line 493	app local (computed at line 406 from request.scope)	Per-request
Module-level globals referenced (not closure captures): json (7), jsonable_encoder (61), EndpointContext (63), ResponseValidationError (66).
Summary: 7 route-static, 1 per-request. The single per-request capture (endpoint_ctx) is used only on the error path (line 493) to enrich ResponseValidationError with source-file/line metadata.
2. _serialize_sse_item (routing.py:516-543)
1 free variable from enclosing scopes:
#	Variable	Referenced at	Source scope	Classification
1	_serialize_data	line 542	app local (defined at line 487)	Per-request
Module-level globals referenced (not closure captures): ServerSentEvent (73), json (7), jsonable_encoder (61), format_sse_event (74).
Summary: 0 route-static, 1 per-request. _serialize_sse_item delegates all Pydantic validation/serialization to _serialize_data, which it reaches via the single per-request closure. The ServerSentEvent branch (lines 517-537) uses only module-level imports and the item parameter — no closure captures at all.
3. _sse_producer_cm (routing.py:550-600)
0 free variables from enclosing scopes.
This is the non-obvious finding. Every variable in _sse_producer_cm's own body is either local or module-level global:
Variable	Referenced at	Classification
send_stream	lines 568, 573	Local (assigned at line 568)
receive_stream	lines 568, 584, 589	Local (assigned at line 568)
send_keepalive	lines 577, 584, 590, 592	Local (assigned at line 577)
receive_keepalive	lines 577, 599	Local (assigned at line 577)
_producer	line 597	Local (defined at line 572)
_keepalive_inserter	line 598	Local (defined at line 581)
tg	lines 596-600	Local (from async with)
anyio	lines 568, 578, 588, 593, 596	Global (module import, line 40)
asynccontextmanager	line 550	Global (module import, line 26)
ObjectReceiveStream	line 552	Global (module import, line 42)
AsyncIterator	line 551	Global (module import, line 12)
_PING_INTERVAL	line 588	Global (module import, line 70)
KEEPALIVE_COMMENT	line 592	Global (module import, line 71)
TimeoutError	line 591	Global (builtin)
Summary: 0 route-static, 0 per-request. _sse_producer_cm captures nothing from app or get_request_handler.
Why _sse_producer_cm has zero direct captures
The variables that flow through the SSE pipeline (sse_aiter, _serialize_sse_item) are captured by _producer — which is nested inside _sse_producer_cm, not by _sse_producer_cm itself. Python's closure semantics resolve free variables per-function: _sse_producer_cm's __code__.co_freevars is empty because its own body never names sse_aiter or _serialize_sse_item.
The reason _sse_producer_cm must still be defined inside app is structural, not capture-driven:
1. Its nested child _producer needs to capture sse_aiter (line 574) and _serialize_sse_item (line 575) from app's scope.
2. It is entered as an async context manager on async_exit_stack (line 606), which is itself a per-request object from app's scope.
So _sse_producer_cm is an indirection layer: it creates the memory streams and task group, but the actual SSE-specific work (iterating the generator, serializing items) is delegated to _producer, which performs the captures on its behalf.