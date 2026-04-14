+++
title = 'Response Body and Connection Reuse in Go'
date = 2025-01-16T19:20:26+02:00
draft = false
+++

## Introduction
If you have ever used the `net/http` package in Go, you know that you should close `Response.Body`. The `io.ReadCloser` type and its accompanying comment make this very clear.

```go
type Response struct {
	...
	// The http Client and Transport guarantee that Body is always
	// non-nil, even on responses without a body or responses with
	// a zero-length body. It is the caller's responsibility to
	// close Body. The default HTTP client's Transport may not
	// reuse HTTP/1.x "keep-alive" TCP connections if the Body is
	// not read to completion and closed.
	//
	// The Body is automatically dechunked if the server replied
	// with a "chunked" Transfer-Encoding.
	// ...
	Body io.ReadCloser
	...
}
```

If you are attentive, you might also notice the part about reusing the connection. The comment states that for the underlying connection to be reused, the `Body` should be read to completion. However, for me, it's unclear who's responsible for that. Why? I am not exactly sure - maybe because I've gotten used to `Close` methods handling all termination logic, or because the comment on the next line looks like a good-to-know detail rather than a call to action.

To be on the safe side, like me, you probably just drain the connection before closing it, all while remaining unsatisfied with the uncertainty. So, here is the question for this article: **should a caller drain the response body to completion before closing it**? 

I want to clarify some things on the spot: the aforementioned `reuse HTTP/1.x "keep-alive" TCP connection` has nothing to do with the TCP keep-alive packets. What is meant here is returning the connection back to the connection pool managed by the `net/http` package, allowing it to be reused instead of establishing new ones. This helps save on TCP handshakes and the more expensive TLS handshakes. The connection pooling operates at the application level, whereas the TCP keep-alive mechanism resides at the transport level of the OSI model.
'Read to completion' means reading until the [io.EOF](https://github.com/golang/go/blob/go1.23.4/src/io/io.go#L44) signal.

Spoiler for the impatient: yes, **it's the caller's responsibility to deplete the response body to make the connection reusable**. If it's that simple, why bother writing the article? I believe there's some confusion around this topic, and I would like to give more or less comprehensive explanation. 

There are also some factors that add to the confusion:
- The `Body`'s `Close` method implementation contains code to drain the body
- There's lack of detailed information on this topic online. Googling yields very few related results, and in nearly all of them, you'll find contradicting answers

But before proving this assertion and addressing these points, let's take the opportunity to learn how persistent connections are implemented.
## Implementation overview
The code snippets accompanying this text are stripped of parts I consider irrelevant to the topic and are intended to support the explanation.

Let's begin with the [RoundTrip call](https://github.com/golang/go/blob/aab837dd46797b88d9ca827809fd7db663a5dd8d/src/net/http/client.go#L259). This leads us to the [roundTrip function](https://github.com/golang/go/blob/2a7ca156b8189c68c0a29b4c66194a42c5ce3c9b/src/net/http/roundtrip.go#L30). At the core of this function are two key calls: 
- [t.getConn](https://github.com/golang/go/blob/cf501e05e138e6911f759a5db786e90b295499b9/src/net/http/transport.go#L633) , which retrieves a persistent connection
- [pconn.roundTrip](https://github.com/golang/go/blob/cf501e05e138e6911f759a5db786e90b295499b9/src/net/http/transport.go#L644), which sends the request and reads the response

```go
func (t *Transport) roundTrip(req *Request) (_ *Response, err error) {
	pconn, err := t.getConn(treq, cm)
	if err != nil {
		req.closeBody()
		return nil, err
	}
	
	var resp *Response
	if pconn.alt != nil {
		// HTTP/2 path.
		resp, err = pconn.alt.RoundTrip(req)
	} else {
		resp, err = pconn.roundTrip(treq)
	}
}
```

`getConn` creates an instance of `wantConn` to await a connection, adds it to the queue of such waiters, and then waits for the object to receive a connection.
```go
func (t *Transport) getConn(treq *transportRequest, cm connectMethod) (_ *persistConn, err error) {
	w := &wantConn{
		result: make(chan connOrError, 1),
	}
	
	if delivered := t.queueForIdleConn(w); !delivered {
		t.queueForDial(w)
	}
	
	select {
	case r := <-w.result:
		return r.pc, r.err
	}
}
```
[t.queueForIdleConn](https://github.com/golang/go/blob/cf501e05e138e6911f759a5db786e90b295499b9/src/net/http/transport.go#L1096) inspects the list of already opened and ready-to-use connections (`t.idleConn`) for the required address and [sends one](https://github.com/golang/go/blob/cf501e05e138e6911f759a5db786e90b295499b9/src/net/http/transport.go#L1147) to `w.result` if available. If no idle connections are found, `t.queueForDial` [initiates dialing a new connection](https://github.com/golang/go/blob/cf501e05e138e6911f759a5db786e90b295499b9/src/net/http/transport.go#L1517) (or queues it for later). 

The dialing happens in a separate goroutine that calls [t.dialConn](https://github.com/golang/go/blob/cf501e05e138e6911f759a5db786e90b295499b9/src/net/http/transport.go#L1563). Finally, in [dialConn](https://github.com/golang/go/blob/cf501e05e138e6911f759a5db786e90b295499b9/src/net/http/transport.go#L1684), we see how persistent connections are created:

```go
func (t *Transport) dialConn(ctx context.Context, cm connectMethod) (pconn *persistConn, err error) {
	pconn = &persistConn{
		t: t,
		cacheKey: cm.key(),
		reqch: make(chan requestAndChan, 1),
		writech: make(chan writeRequest, 1),
		closech: make(chan struct{}),
		writeErrCh: make(chan error, 1),
		writeLoopDone: make(chan struct{}),
	}
	...
	conn, err := t.dial(ctx, "tcp", cm.addr())
	if err != nil {
		return nil, wrapErr(err)
	}
	pconn.conn = conn
	...
	go pconn.readLoop()
	go pconn.writeLoop()
	return pconn, nil
}
```

So persistent connection consists of:
- `persistConn`: the connection handler
- `writeLoop`: a loop that writes requests to the underlying socket represented by `conn`
- `readLoop`: a loop that reads responses from the underlying socket represented by `conn`

Remember `pconn.roundTrip`? You can probably already guess how it works.
```go
func (pc *persistConn) roundTrip(req *transportRequest) (resp *Response, err error) {
	...
	writeErrCh := make(chan error, 1)
	pc.writech <- writeRequest{req, writeErrCh, continueCh}

	resc := make(chan responseAndError)
	pc.reqch <- requestAndChan{
		treq: req,
		ch: resc,
		...
	}
	...
	for {
		select {
			case re := <-resc:
				return handleResponse(re)
		}
	}
}
```
In essence, `roundTrip` passes the request to the pre-created `writeLoop` via `pconn.writech` and reads the response from the pre-created `readLoop` via ... what? Yep, there is no channel in `pconn` to read responses from `readLoop`, so the caller - `roundTrip` - creates one (that's `resc`) and passes it to the `readLoop` first. Quite an interesting pattern you don't see every day.

Let's get back to the `readLoop`. 
```go 
func (pc *persistConn) readLoop() {
	...
	for alive {
		rc := <-pc.reqch
		resp, err = pc.readResponse(rc, trace)

		waitForBodyRead := make(chan bool, 2)
		body := &bodyEOFSignal{
			body: resp.Body,
			earlyCloseFn: func() error {
				waitForBodyRead <- false
				<-eofc
				return nil
			},
			fn: func(err error) error {
				isEOF := err == io.EOF
				waitForBodyRead <- isEOF
				if isEOF {
					<-eofc
				} else if err != nil {
					if cerr := pc.canceled(); cerr != nil {
						return cerr
					}
				}
				return err
			},
		}
		resp.Body = body

		select {
		case rc.ch <- responseAndError{res: resp}:
		case <-rc.callerGone:
			return
		}

		select {
		case bodyEOF := <-waitForBodyRead:
			alive = alive &&
				bodyEOF &&
				!pc.sawEOF &&
				pc.wroteRequest() &&
				tryPutIdleConn(rc.treq)
			if bodyEOF {
				eofc <- struct{}{}
			}
		}
	}
}
```
The part that interests us the most here is [how the response body is created](https://github.com/golang/go/blob/cf501e05e138e6911f759a5db786e90b295499b9/src/net/http/transport.go#L2285). If you dive into the implementation of `readResponse`, you'll find that [the resp.Body is either an instance](https://github.com/golang/go/blob/4f0408a3a205a88624dced4b188e11dd429bc3ad/src/net/http/transfer.go#L562) of `body` or a `NoBody` stub, which in either case, `readLoop` wraps in `bodyEOFSignal`. Finally, let's take a look at how `bodyEOFSignal` implements `Close`: 
```go
func (es *bodyEOFSignal) Close() error {
	es.mu.Lock()
	defer es.mu.Unlock()
	if es.closed {
		return nil
	}
	es.closed = true
	if es.earlyCloseFn != nil && es.rerr != io.EOF {
		return es.earlyCloseFn()
	}
	err := es.body.Close()
	return es.condfn(err)
}
```
An early close always results in the `earlyCloseFn` being called without invoking the underlying `body.Close`. In this case, `readLoop` reads `false` from `case bodyEOF := <-waitForBodyRead`, flips the `alive` flag, and proceeds to terminate the connection. 

Thus, if the response body is not read to completion by the caller the connection, can't be reused and will be terminated. You can test this yourself by debugging an HTTP client step by step or by checking the `netcat` output.

The last thing to mention is how to properly drain the response body. No one knows it better than the `net/http` package itself, so let's take a look at [how it's done by http.Client](https://github.com/golang/go/blob/go1.23.4/src/net/http/client.go#L706) while handling redirects. Yes, the package does it for the intermediate responses, but not for the last.
```go
func (c *Client) do(req *Request) (retres *Response, reterr error) {
	...
	// Close the previous response's body. But
	// read at least some of the body so if it's
	// small the underlying TCP connection will be
	// re-used. No need to check for errors: if it
	// fails, the Transport won't reuse it anyway.
	const maxBodySlurpSize = 2 << 10
	if resp.ContentLength == -1 || resp.ContentLength <= maxBodySlurpSize {
		io.CopyN(io.Discard, resp.Body, maxBodySlurpSize)
	}
	resp.Body.Close()
	...
}
```
There are at least two reasons why you might want to limit the bytes read:
- If the response is huge, it might be faster to terminate the connection and open a new one rather than draining it. After all, connection pooling is an optimization, not a necessity. Keep in mind that for the **https** scheme, reusing a connection saves not only an extra TCP handshake, but also the more expensive TLS handshake
- An unbounded read could be exploited by a bad agent feeding the connection with an endless data stream

Now, let's return to the points that were causing confusion.
## Confusion factors
In your search for answers, you might have come across the [Close implementation of the body struct](https://github.com/golang/go/blob/4f0408a3a205a88624dced4b188e11dd429bc3ad/src/net/http/transfer.go#L972). There is indeed code responsible for consuming the body. The catch, however, is that the `body` structure is used by both the HTTP client and the server. The `case b.doEarlyClose:` block is executed only by the HTTP server when reading the **request body**. If there is [no request payload](https://github.com/golang/go/blob/4f0408a3a205a88624dced4b188e11dd429bc3ad/src/net/http/transfer.go#L562), the body is set to `NoBody`. As a result, the `default` branch is not reached. Thus, the automatic depletion is only done by the server, not the client.

If you Google the problem, you'll likely come across contradicting answers. To start with, there isn't much information on the topic. You'll probably find a thread on forum.golangbridge.org and a few discussions on Github. In all of them, you'll encounter conflicting answers. While some are correct, how can you tell which one is right without a deeper knowledge?
I hope this article helps to put an end to this uncertainty.
## Conclusion
The answer has been given, but you may still have questions. Can we make the situation better?
Could the `Body` comment be expanded with an explanation? Certainly, but it wouldn't eliminate the need to drain the response.

What about adding an automatic response drain? According to one of the Go authors, [it was there in the first Go versions](https://github.com/google/go-github/pull/317#issuecomment-203138488), but it proved problematic and was removed for good reason. However, if the issue stirs your soul, I encourage you to challenge the current state of affairs.

Or, we can wait for full HTTP/2 adoption, where this issue simply doesn't exist, as requests can be multiplexed within the same connection. Time will tell.
## Attribution
This article includes code snippets from https://github.com/golang/go, licensed under the BSD License. Copyright 2009 The Go Authors. You can find the full license text [here](https://github.com/golang/go/blob/go1.23.4/LICENSE).