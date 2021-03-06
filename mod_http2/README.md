# mod_h2 internals

A description of how the module's structure, the main terms used and how it overall works.

While the complete and accurate description will always be the source code, this document is intended
to serve as an entrance to understand how the module works and what its main moving parts are.

## Terms

 * `c1`: a *primary* connection. This is a connection to a HTTP/2 client.
 * `c2`: a *secondary* connection. This is an internal, virtual one used to process a request.
 * `session`: the HTTP/2 state for a particular `c1` connection.
 * `stream`: a HTTP/2 stream that (commonly) carries a request+body and delivers a response+body. Each stream has a unique 32 bit identifier, as defined in the HTTP/2 protocol. Stream 0 is the c1 connection itself.
 * `mplx`: the multiplexer, one per `session`. It processes streams, forwards input (request bodies) and collects output from processing. To process a stream, it creates a `c2` connection.
 * `worker`: polling all registered `mplx`s for `c2` connections to process. `mplx`s registers themselves at the workers when they have something to process.
 * `slot`: a particular worker thread. The number of workers may vary based on server load and configuration.
 * `beam`: a mechanism for transferring APR's *buckets* between `c1`and `c2`. More under memory.

## File Structure

Source files are all prefixed with `h2_` followed by the main topic they are about. `h2_c2_filter` for example contains all input/output filter code for `c2` connections. `h2_session` is all about the `session` instances. etc.

## Session States

HTTP/2 sessions can be in one of the following states:

 * `INIT`: state during initialization of the session. Sends initial SETTINGS to client and transits to `BUSY` on success.
 * `BUSY`: reading `c1` input, writing frames on `c1` output and checking any `c2` for I/O events. Switches to `WAIT` when `c1` input is exhausted.
 * `WAIT`: collects `c1` socket and all `c2` pipes into a pollset and does a wait with connection timeout on events. Transits on `c1` input to `BUSY` again.
 * `IDLE`: there are no streams to process. The session waits on new `c1` input to arrive. If a stream has been processed already, this is like a HTTP/1 *keepalive*.
 * `DONE`: session is done processing streams and shuts down. Possibly a last GOAWAY frame is being sent. Transits to `CLEANUP` when the protocol needs have been taken care of.
 * `CLEANUP`: release all internal resources. Make sure that any ongoing `c2` processing terminates.

There a sub-states in the `BUSY` and `WAIT` handling. A session may chose to no longer accept new streams from the clients while finishing processing all ongoing streams. This is, for example, triggered by a graceful shutdown of the child process.
 
Errors and timeouts on the `c1` connection trigger a transition to `DONE`.
 
 
## Stream States

These mostly correspond to the states described in the HTTP/2 standard, with some additions for internal handling:

 * `IDLE`: a stream has been created by a request from the client.
 * `OPEN`: all request headers have arrived. The stream can start processing.
 * `RSVD_R`: the stream (identifier) has been reserved by the client (remote).
 * `RSVD_L`: the stream (identifier) has been reserved by the session (locally).
 * `CLOSED_R`: stream was closed by the client (remote). A (possibly empty) request body is complete.
 * `CLOSED_L`: stream was closed by the session (locally) and the output is complete.
 * `CLOSED`: both stream *ends* have been closed.
 * `CLEANUP`: the session is done with the stream, its resources may be reclaimed. Such a stream is handed over to the `mplx` which performs the reclamation. This needs to take care of a potentially still running `c2` connection.

A `mplx` maintains three `stream` lists:

 * `streams`: the active streams which are being processed (or scheduled to be).
 * `shold`: the streams in `CLEANUP` which have an ongoing `c2` connection that needs to terminate first.
 * `spurge`: the streams without or with a finished `c2` that can be reclaimed.
 
## Memory

### Setup

The APR memory model with its `pools` determines much of `mod_h2`'s structure, initialization and resource reclamation strategies. `pools` are the foundation of everything in Apache `httpd`: lists, tables, files, pipes, sockets, data and meta data transfers (`bucket`s) are tied to them.

The fundamental restriction of `pools` is that they are not thread safe. Using the same pool from 2 threads will mess up its internal lists. The more busy the server is, the more likely this will then happen. Since everything one does with APR's features has the potential to modify its underlying pool, all the things listed above are not thread safe.

Closing a file will modify the pool it was opened with, for example. If 10 files are opened with the same pool, one is unable to use 5 of them in one thread and the rest in another. Everything that is based on the same pool needs to stay on the same thread.

A `session` handling `c1` needs to run in parallel to request processing on `c2` connections. That means `session` and `c2`s have completely separate pools. 

When a session creates a stream, it creates a new *child* pool for it. Pool memory can only be freed by destroying the whole pool. To handle thousands of streams without leaking memory, they have to be placed in child pools that can be reclaimed when a stream is done.

A `mplx` is used by the `session` *and* by `c2` connections. To manage its internal structures, it also needs its own, separate pool. This one is protects via its `mutex`. It creates new `c2` connections when needed also with separate pools as processing happens in separate `worker` threads.

All these *separate* pools have their own APR `allocator` (the one that manages system memory) to be independent. However, there are still tied with a *parent/child* relationship to not lose track of them (leaking). So `c2` pools are children of `mplx` pool which is a child of the `session` pool.

### Teardown

When destroying a pool, it modifies its parent pool. When reclaiming a `c2` pool, the `mplx` pool will be changed. So, this can only be allowed to happen inside the `mplx` mutex protection. When reclaiming the `mplx`, it modifies the `session` pool. So this may only happen on the thread that works on `session`.

This means that tearing down a `session` needs to tear down the `mplx` which needs to tear down all its `c2` connections first. Or else.

### Stream Memory

Streams have their own input/output buffers, allocated from their own pool. Similar to "normal" HTTP/1 requests, they need to take care that all their outgoing data on `c1` has actually been sent, before they can be destroyed. HTTP/1 uses the `EOR` meta bucket for that. HTTP/2 has a `H2_EOS` bucket that is similar.

On closing a stream, a `H2_EOS` is created and send on `c1`. When this bucket is destroyed, the stream is handed to `mplx` for safe destruction. `mplx` then removes the stream from its list of active ones. It places it on its `spurge` list when the stream has no `c2`or it has already returned from the `worker`. For an active `c2`, the stream is placed into `shold`.

When a `worker` tells an `mplx` that it has finished a `c2`, the `mplx` checks if the stream is still active or if it is found in `shold`. If the stream is in `shold`, it is moved to `spurge`. It cannot be destroyed right away, since the stream's pool is a child of `session`. That would manipulate the session pool from inside a worker thread.

The purge list is instead only processed, when `session` calls the `mplx`. 

### Data Transfer

With all this pool touchiness, how does request/response/bodies ever get transferred between a `stream` and its `c2` connection that does the actual work? That merits its own chapter about `bucket beams`.


## Bucket Beams

Apache httpd uses APR's `bucket brigade`s to transfer data and meta information through its connection filters. So, whatever also one does, ultimately `streams` and `c2` connections will use brigades.

The problem is: **it is impossible to transfer a bucket from one brigade to another between threads**.

A bucket belongs to a `bucket_alloc` which belongs to a memory pool. All three are not thread safe and tied. Imagine transferring from brigade `b1` on thread `t1` into brigade `b2` on thread `t2`:

 * `t1` can take data out of `b1`, but cannot put it into `b2`.
 * `t2` can put data into `b2`, but cannot take it out of `b1`.

So, mod_h2 needs something to juggle the data in between `t1` and `t2` doing their thing. That is the job of a bucket beam.

(*The name* "beam" is inspired by Start Trek beam technology, where people are transported from one place to another - but not instantly. They first become frozen and semi-transparent at both location, until the arrival is complete and then they disappear at the start. Same happens to buckets in a bucket beam.)

A bucket beam has two APR `ring`s (rings are a doubly linked list that works **independent** from pools. Yay!).

 * `buckets_to_send`: when `t1` calls `h2_beam_send(beam, b1)`, the beam takes buckets out of `b1` and appends them to this ring. 
 * `buckets_consumed`: when `t2` calls `h2_beam_receive(beam, brigade)`, buckets are taken from the to_send ring, converted to new buckets (which are appended to the receiver `brigage`) and the original ones are added to the 'consumed' ring.

The buckets in `buckets_consumed` can then be destroyed the next time that thread `t1` calls.

### Bucket Conversions

Buckets in a beam are "converted" into new buckets (the originals from 'to_send' are readonly). For well known meta buckets, just a new one of the corresponding type is created and added to the receiver brigade. 

This concerns 'eos', 'flush' and 'error' buckets. Several other meta buckets are not converted. For example the 'eor' bucket is not passed on in any from, just added to the `buckets_consumed` ring on transfer. This is important to make sure that all sent buckets are destroyed in the correct order.

Data buckets have their data and length extracted via `apr_bucket_read()` and the data is written to the receiver brigade. This makes a copy of the data. Special handling is done for 'file' and 'mmap' buckets where the file/mmap itself is `dup`ed and added to the receiver. This allows transfer of large response bodies without copying any data.

For additional conversions, beams allow registration of conversion functions. All unknown buckets are passed to them. They may add their own converted buckets to the receiver brigade.

### Beam Memory

Bucket beams can be configured with a buffer limit. This blocks senders when they try to add more (data) buckets than the limit allows. They become unblocked when a receiver takes data out. Data buckets of type 'file' or 'mmap' are not counted against this limit, as they do not really occupy memory in the beam's buffer. At least not additional memory as the sender already has created these buckets.

### Efficiency

While the response to a HTTP/2 request is being generated, the request occupies a h2 worker thread. Once the
response is complete (headers and body buckets), and all buckets have been passed through the filter chain, the
request processing returns and the worker can be used for other requests.

Buckets, however, can only be passed when they do not exceed the memory limitations for a HTTP/2 stream. Sending
buckets on the output stream will block, when this limit is reached, and that blocks the worker thread.

Fortunately, for static files, this limit is never reached, as file buckets have a tiny memory footprint until
their data is actually read. Beams will accept file buckets of any length without blocking. The same is true
for mmap buckets. Serving static files will only shortly occupy workers to lookup the file and set the response headers and all other work then happens on the c1 connection itself.


## Polling

The module `mplx` uses pipes and pollsets to monitor communications on `c1` and `c2`:

 * for `c1` it polls the connection's *socket* for new incoming data.
 * for `c2`s it polls `pipe_out_prod`, a pipe used for signalling the availability of new output.
 * for `c2`s it polls `pipe_in_drain`, a pipe used for signalling that input has been read by `c2` (if the stream processing on a `c2` has indeed input, e.g. a request body).

The HTTP/2 protocol has in-built flow control on its streams. `c1` needs to be continuously monitored not only for new requests, but also for updates on flow control window sizes. 

Flow control also is in effect for clients sending data to the server, e.g. in a POST request. That is why the consumption of such input data is being monitored. When data is passed on to a `c2`'s input, the stream's window for the client is updated (and the update sent to the client on `c1`), so more POST data can be sent.

In this way, a HTTP/2 connection is much more busy in both directions than HTTP/1 where there is a clear separation between sending and receiving phases for a request.

For `c2`s expecting input, an additional pipe is being created: `pipe_in_prod` which signals when `c1` has written new data to the `c2` input. This pipe is read by a `c2` waiting for additional data.

The bucket beams have callbacks for various events that are used to feed these pipes:

 * `h2_beam_on_received()`: used to signal that `c2` input has been read.
 * `h2_beam_on_was_empty()`: used to signal the addition of new data.

This way, sending/receiving on a bucket beam adds notifications to the pipes.

### Timeouts

When a `session` is in state `WAIT`, it polls using the `c1` connection timeout. If polling times out, the session is shut down.

To enter `WAIT` state, a `session` needs to be sure that there is no pending data in `c1`'s input filters. A `session` enters `BUSY` state whenever it is not certain about this. A `BUSY` session uses the pollset with timeout 0, which immediately returns, and tries to read/write more on the `c1` connection.

Eventually, it will detect that `c1` input filters have nothing buffered any more and enter `WAIT` state.

A `session` enters `IDLE` state when it has no more streams to process and nothing more to send. It will then poll with a very short timeout and, when nothing changes, return the `c1` connection to the `mpm` for "keepalive" monitoring.

## Workers

The modules has its own worker pool, separate from the `mpm` worker thread. This is mostly for historical reasons, as
HTTP/2 came late into the server, but separate pools make sense in order for the workers not to deadlock themselves (all workers holding a `c1` and waiting for a free worker to process their `c2`s).

The h2 workers have a fixed number of `slots` for threads and start a minimal number (all configurable). Additional threads are added in a busy situation up to the number of available slots. When such a slot becomes idle for a number of seconds (configurable), it shuts down again. Also, inactive workers shut down when the server does a graceful reload.

h2 workers manage a queue of `mplx` instances that have work to be done. An idle worker removes an mplx from the queue and asks it for a `c2` to process. When done, it informs the `mplx` that it has finished the `c2` and if it is willing to work on another `c2` from this `mplx`. It is only willing to do so when the maximum worker count has not been reached yet.

The reason for this design are:

 * all `mplx` are for a particular `c1` and all `c1` should be treated equal. `c1`s that open many requests at once should not get preferred treatment over others that have "only" a single request.
 * scheduling `c2`s directly would require creating them, potentially a lot of them, without a worker being available. 100 connections with 10 requests ongoing would hold 1000 `c2`s all the time while only a small set is being worked on.
 * asking the same `mplx` for additional work is not unfair as long as more workers are available. And it results in better performance than switching workers all the time.


## Interactions with HTTP/1

The HTTP/1 protocol handling is the default in Apache httpd and there are several places where it applies
itself for "HTTP" without any consideration of the actual protocol version. There are ongoing efforts to separate
the generic HTTP processing (e.g. checks on valid headers, method name, paths, etc.) from the *serialization*
of responses and bodies on the wire.

Two areas are noteworthy here to understand how mod_h2 works.

#### Request Bodies

Requests in HTTP/1 have either an implied length of 0 or an announced length in the `content-length` header
or use  the "chunked" transfer encoding. The standard `HTTP_IN` filter in Apache therefore assumes that the
request has not body if both content length and chunked transfer information is missing.

Since HTTP/2 allows request bodies without announced length and no chunking, mod_h2 must fake a chunked
request input for those requests.

#### Response Headers

Similarly, the server installs the `HTTP_HEADER` filter on a request output which, on seeing the response body,
inserts the response header in HTTP/1 format on the filter chain. HTTP/2 has no need for this and removes
`HTTP_HEADER` on each of its requests.

In its stead, it applies its own filter in the output that writes a special `H2HEADERS` meta bucket. This
bucket contains response status and all headers and can be passed through the output beam. The c1 processing
then converts `H2HEADERS` buckets to the HTTP/2 HEADER frames for responses and footers.


## DDoS Protection

HTTP/2 as a protocol has more internal state than HTTP/1. This makes prevention of exploits more
difficult. The known HTTP/1 attacks, such as "Slo Loris", can also be applied and are mitigated
using the known mod_reqtimeout mechanisms.

But "Slo Loris" can also attack the c1 processing on the main connection. After all, the goal of such
an attack is to exhaust the server's worker thread and preventing processing of new connection. By
keeping the HTTP/2 c1 processing occupied, the same goal can be achieved if there is no special 
protection in place. This has been a bit of an arms race in the last few years.

Another attack angle is aimed at exhausting the h2 workers with requests, that are slow or kept in
an incomplete state or are just simply cancelled and started, again and again.

To protect against such behaviour, a mod_h2 has an internal "mood" to obey a client based on its
past behaviour (on this connection, there is not tracking). The mood starts neutral where a client
is allowed to have 6 active requests, e.g. occupy 6 h2 workers. When responses are read by the client
in a timely fashion (no stalling), this mood rises and the limit is raised. Should the client stall
responses (drag its feet with window updates) or reset streams before a response has been seen by it,
the processing limit is lowered.

Another protection against flooding is scheduling fairness between connections. If one connection has a single
request it needs a worker for and another has 100, both get one worker assigned, before the other 99 requests
may receive attention.

