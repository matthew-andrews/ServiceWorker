Service Workers Algorithms
===
> **NOTE:**
>
> - **Register** and **_Upgrade** honor HTTP caching rule.
> - Underscored function and attribute are UA-internal properties.
> - The details are still vague especially exception handling parts.

--
**Register**(_script_, _scope_)

1. Let _promise_ be a newly-created Promise.
2. Return _promise_.
3. Run the following steps asynchronously.
  1. Let _scope_ be _scope_ resolved against the document url.
  2. Let _script_ be _script_ resolved against the document url.
  3. If the origin of _script_ does not match the document's origin, then
    1. Reject _promise_ with a new SecurityError.
    2. Abort these steps.
  4. If the origin of _scope_ does not match the document's origin, then
    1. Reject _promise_ with a new SecurityError.
    2. Abort these steps.
  5. Let _serviceWorkerRegistration_ be *_GetRegistration(_scope_)*.
  6. If _serviceWorkerRegistration_ is not null and _script_ is equal to _serviceWorkerRegistration_.*_scriptUrl*, then 
    1. Note: these steps are to force a worker update if the page was shift+reloaded and the document is within _scope_
    2. Let _documentServiceWorkerRegistration_ be the Service Worker registration used by this document.
    3. If _documentServiceWorkerRegistration_ is not null, then
      1. Resolve _promise_ with *_GetNewestWorker(serviceWorkerRegistration)*.
      2. Abort these steps.
    4. Let _documentUrl_ be the document's url.
    5. If _serviceWorkerRegistration_ is not equal to *_ScopeMatch(_documentUrl_)*
      1. Resolve _promise_ with *_GetNewestWorker(serviceWorkerRegistration)*.
      2. Abort these steps.
  7. If _serviceWorkerRegistration_ is null, then
    1. Let _serviceWorkerRegistration_ be a newly-created _ServiceWorkerRegistration object.
    2. Set _serviceWorkerRegistration_ to the value of key _scope_ in *_ScopeToServiceWorkerRegistrationMap*.
    3. Set _serviceWorkerRegistration_.*scope* to _scope_.
  8. Set _serviceWorkerRegistration_.*scriptUrl* to _script_.
  9. Resolve _promise_ with *_Update(serviceWorkerRegistration)*

--
**_Update**(_serviceWorkerRegistration_)

1. Let _promise_ be a newly-created Promise.
2. Return _promise_.
1. Perform a fetch of _serviceWorkerRegistration_.*scriptUrl*, forcing a network fetch if cached entry is greater than 1 day old.
2. If fetching the script fails (e.g. the server returns a 4xx or 5xx response or equivalent, or there is a DNS error, or the connection times out), or if the server returned a redirect, then
  1. Reject _promise_ with a new NetworkError
  2. Abort these steps.
3. Let _fetchedScript_ be the fetched script.
4. Let _newestWorker_ be *_GetNewestWorker(serviceWorkerRegistration)*
5. If _newestWorker_ is not null, and _fetchedScript_ is a byte-for-byte match with the script of _newestWorker_, then
  1. Resolve _promise_ with _newestWorker_.
  2. Abort these steps.
6. Let _serviceWorker_ be a newly-created ServiceWorker object, using _fetchedScript_.
7. If _serviceWorker_ fails to start up, due to parse errors or uncaught errors, then
  1. Reject _promise_ with the error.
  2. Abort these steps.
8. Resolve _promise_ with _serviceWorker_
9. Queue a task to call **_Install** with _serviceWorkerRegistration_ and _serviceWorker_.

--
**_Install**(_serviceWorkerRegistration_, _serviceWorker_)

1. Set _serviceWorkerRegistration_.*queuedWorker* to _serviceWorker_
2. Fire _install_ event on the associated _ServiceWorkerGlobalScope_ object.
3. Fire _install_ event on _navigator.serviceWorker_ for all documents which match _serviceWorkerRegistration_.*scope*.
4. If any handler calls _waitUntil()_, then
  1. Extend this process until the associated promises resolve.
  2. If the resulting promise rejects, then
    1. Abort these steps. TODO: we should retry at some point?
5. Fire _installend_ event on _navigator.serviceWorker_ for all documents which match _serviceWorkerRegistration_.*scope*.
6. Set _serviceWorker_.*_state* to _installed_.
7. If any handler calls _replace()_, then
  1. For each document matching _serviceWorkerRegistration_.*scope*
    1. Set _serviceWorkerRegistration_ as the document's service worker registration
  2. Call **_Activate** with _serviceWorkerRegistration_
  3. Abort these steps
8. If no document is using _serviceWorkerRegistration_ as their service worker registration, then
  1. Queue a task to call **_Activate** with _serviceWorkerRegistration_.

--
**_Activate**(_serviceWorkerRegistration_)

1. Let _activatingWorker_ be _serviceWorkerRegistration_.*queuedWorker*
2. Let _exitingWorker_ be _serviceWorkerRegistration_.*currentWorker*
3. Set _serviceWorkerRegistration_.*queuedWorker* to null
4. Set _serviceWorkerRegistration_.*currentWorker* to _activatingWorker_
5. Set _activatingWorker_.*_state* to _activating_
6. If _exitingWorker_ is not null, then
  1. Wait for _exitingWorker_ to finish handling any in-progress requests
  2. Close and garbage collect _exitingWorker_
7. Fire _activate_ event on the associated _ServiceWorkerGlobalScope_ object.
8. Fire _activate_ event on _navigator.serviceWorker_ for all documents which match _serviceWorkerRegistration_.*scope*.
9. If any handler calls _waitUntil()_, then
  1. Extend this process until the associated promises resolve.
  2. If the resulting promise rejects, then
    1. TODO: what now? We may have in-flight requests that we're blocking. We can't roll back. Maybe send all requests to the network?
    2. Abort these steps
10. Set _activatingWorker_.*_state* to _active_
11. Fire _activateend_ event on _navigator.serviceWorker_ for all documents which match _scope_.

--
**_OnNavigationRequest**(_request_)

1. If _request_ is a force-refresh (shift+refresh), then
  1. Fetch the resource normally and abort these steps.
2. Let _parsedUrl_ be the result of parsing _request.url_.
3. Let _serviceWorkerRegistration_ be **_ScopeMatch**(_parsedUrl_).
4. If _serviceWorkerRegistration_ is null, then
  1. Fetch the resource normally and abort these steps.
5. Let _matchedServiceWorker_ be _serviceWorkerRegistration_.*currentWorker*
6. If _matchedServiceWorker_ is null, then
  1. Fetch the resource normally and abort these steps.
7. Document will now use _serviceWorkerRegistration_ as its service worker registration
8. If _matchedServiceWorker_.*_state* is _activating_, then
  1. Wait for _matchedServiceWorker_.*_state* to become _active_
9. Fire _fetch_ event on the associated _ServiceWorkerGlobalScope_ object with a new FetchEvent object
10. If _respondWith_ was not called, then
  1. Fetch the resource normally.
  2. Queue a task to call **_Update** with _serviceWorkerRegistration_.
  3. Abort these steps.
11. Let _responsePromise_ be value passed into _respondWith_ casted to a Promise
12. Wait for _responsePromise_ to resolve
13. If _responsePromise_ rejected, then
  1. Fail the resource load as if there had been a generic network error and abort these steps
14. If _responsePromise_ resolves to a OpaqueResponse, then
  1. Fail the resource load as if there had been a generic network error and abort these steps
15. If _responsePromise_ resolves to an AbstractResponse, then
  1. Serve the response
  2. Queue a task to call **_Update** with _serviceWorkerRegistration_.
  3. Abort these steps.
16. Fail the resource load as if there had been a generic network error and abort these steps

--
**_OnResourceRequest**(_request_)

1. Let _serviceWorkerRegistration_ be the registration used by this document.
2. If _serviceWorkerRegistration_ is null, then
  1. Fetch the resource normally and abort these steps.
3. Let _matchedServiceWorker_ be _serviceWorkerRegistration_.*currentWorker*
4. If _matchedServiceWorker_ is null, then
  1. Fetch the resource normally and abort these steps.
5. If _matchedServiceWorker_.*_state* is _activating_, then
  1. Wait for _matchedServiceWorker_.*_state* to become _active_
6. Fire _fetch_ event on the associated _ServiceWorkerGlobalScope_ object with a new FetchEvent object
7. If _respondWith_ was not called, then
  1. Fetch the resource normally and abort these steps.
8. Let _responsePromise_ be value passed into _respondWith_ casted to a Promise
9. Wait for _responsePromise_ to resolve
10. If _responsePromise_ rejected, then
  1. Fail the resource load as if there had been a generic network error and abort these steps
11. If _responsePromise_ resolves to an AbstractResponse, then
  1. Serve the response and abort these steps
12. Fail the resource load as if there had been a generic network error and abort these steps

--
**_OnDocumentUnload**(_document_)

1. Let _serviceWorkerRegistration_ be the registration used by _document_.
2. If _serviceWorkerRegistration_ is null, then
  1. Abort these steps.
3. If any other document is using _serviceWorkerRegistration_ as their service worker registration, then
  1. Abort these steps.
4. If _serviceWorkerRegistration_.*queuedWorker* is not null
5. Call **_Activate**(_serviceWorkerRegistration_).

--
**Unregister**(_scope_)

1. Let _promise_ be a newly-created _Promise_.
2. Return _promise_.
3. Run the following steps asynchronously.
  1. Let _scope_ be _scope_ resolved against the document url.
  2. If the origin of _scope_ does not match the document's origin, then
    1. Reject _promise_ with a new SecurityError.
    2. Abort these steps.
  3. Let _serviceWorkerRegistration_ be **_GetRegistration**(_scope_).
  4. If _serviceWorkerRegistration_ is null, then
    1. Reject _promise_ with a new NotFoundError
    2. Abort these steps
  5. For each document using _serviceWorkerRegistration_
    1. Set the document's service worker registration to null
  6. Delete _scope_ from *_ScopeToServiceWorkerRegistrationMap*
  7. Let _exitingWorker_ be _serviceWorkerRegistration_.*currentWorker*
  8. Wait for _exitingWorker_ to finish handling any in-progress requests
  9. Fire _deactivate_ event on _exitingWorker_ object.
  10. Resolve _promise_.

--
**_ScopeMatch**(_url_)

1. Let _matchingScope_ be the longest key in *_ScopeToServiceWorkerRegistrationMap* that glob-matches _url_
2. Let _serviceWorkerRegistration_ be **_GetRegistration**(_matchingScope_)
3. Return _serviceWorkerRegistration_

--
**_GetRegistration**(_scope_)

1. If there is no record for _scope_ in *_ScopeToServiceWorkerRegistrationMap*, return null
2. Let _serviceWorkerRegistration_ be the record for _scope_ in *_ScopeToServiceWorkerRegistrationMap*
3. Return _serviceWorkerRegistration_.

--
**_GetNewestWorker**(_serviceWorkerRegistration_)

1. Let _newestWorker_ be _serviceWorkerRegistration_.*queuedWorker*.
2. If _newestWorker_ is null, then
  1. Let _newestWorker_ by _serviceWorkerRegistration_.*currentWorker*.
3. Return _newestWorker_.