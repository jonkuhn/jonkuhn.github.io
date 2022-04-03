---
layout: default
title: gRPC Streaming Lessons Learned
---
Over the past few years I have worked with building services that communicate with each other via gRPC.  This post focuses on some lessons I have learned related to designing RPCs that utilize gRPC server side streaming.  gRPC supports multiple types of streaming and server side streaming is the one that allows the server to stream back multiple responses to a single RPC request from the client.

gRPC server side streaming can be useful to allow clients to start processing messages sooner and can help avoid holding a large number of messages in memory on the client and server.

In this post I will discuss a few things I have learned to consider when designing gRPC streaming RPCs.  My goal is to describe these well enough that you can consider the tradeoffs and decide how (and if) they apply to your application.

## Consider The Costs Of Stopping Streams Early
Some of the streaming RPCs I've worked with were designed to stream back responses indefinitely until all results were returned or the client cancelled the RPC.  These RPCs were also designed to periodically return continuation tokens which could be used by the client to pick up where it left off if the call was interrupted.

This design can work okay if we know all clients are going to want to read all results.  However, this design becomes inefficient if any client wants to implement pagination and only consume a given number of results at a time.  The client may implement pagination by reading the stream until a continuation token is received and then closing the stream.  The continuation token would then be used in a subsequent request so the client could pick up where it left off.

This seems to work well enough until you think about it from the server's perspective.  The server has no way of knowing when it has sent the last response the client is interested in.  Therefore, the server will continue to fetch more results and send them on the response stream until it finds out that the client has cancelled the RPC or closed the stream.

The extra work the server is doing during this time can be costly.  This extra work may involve performing another database query even though the client will never use the results.  In this case not only does the server waste its compute resources but also the database's resources.

This situation is illustrated below:
![CostOfStoppingGrpcStreamEarly](/images/CostOfStoppingGrpcStreamEarly.png)

## Paginated Streaming RPCs
I still use streaming when designing newer paginated RPCs.  However, I have learned to do the following in order to avoid the server doing extra work:
- Have the request accept a desired page size
- Have the server stop after the desired page size is reached and return a continuation token
- Have the client do another RPC call with the continuation token for subsequent pages

This technique is illustrated in the diagram below:
![GrpcStreamingPagination](/images/GrpcStreamingPagination.png)

This is similar to how a paginated REST request might work, except that each item on the page is streamed back to the client and the client can easily process each item as soon as it is received.

## Consider The Scaling and Load Balancing Implications Of Long Running Operations
With gRPC streaming it can be tempting to create long running RPCs.  Even if the stream will not be terminated early, there can be some scaling and load balancing disadvantages to long running streaming calls.

By their nature, each gRPC streaming request is sent to a single replica of the service and that replica serves all of the streaming results for that request.  If the service is under enough load that it needs to scale out, none of the in-progress streaming operations can rebalance to new replicas.  Only new requests can contact the new replicas.  If streaming RPCs are not long running, then one long running request is effectively broken up in to many shorter requests.  This can be beneficial because each new request can be an opportunity to contact a different replica.

## Consider Extending Timeouts After Receiving Each Response
If you keep your gRPC streaming calls short running, you may be able to just set a timeout that applies to the entire operation.  For this to work, you need to be confident that all of the results from the streaming operation will be received before the timeout expires.

If you are working with streaming RPCs that can be long running or may return a smaller or larger number of results depending on the call, there is another option.  You can set a shorter timeout and extend it after each streaming response is received.  This means the timeout is effectively the maximum time between receiving streamed response messages.

Here is a snippet from a C# client that applies a timeout to each message:

```c#
var cancellationTokenSource = new CancellationTokenSource();
cancellationTokenSource.CancelAfter(TimeSpan.FromSeconds(1));
var cancellationToken = cancellationTokenSource.Token;

using (var call = client.SayHello(new HelloRequest { Name = "GreeterClient" }, cancellationToken:cancellationToken))
{
    while(await call.ResponseStream.MoveNext(cancellationToken))
    {
        Console.WriteLine($"Received: {call.ResponseStream.Current.Message}");

        // Extend the timeout before the next call to MoveNext
        cancellationTokenSource.CancelAfter(TimeSpan.FromSeconds(1));
    }
}
```

Here is a snippet of a Go client that applies a timeout to each message:
```go
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	stream, err := client.SayHello(ctx, &HelloRequest{
		Name: "golang greeter client",
	})

	if err != nil {
		log.Fatalf("%v.SayHello(_) = _, %v", client, err)
	}

	for {
        // Run code in a goroutine to call cancel() for the context
        // passed to client.SayHello after the timeout expires.
        // In order to prevent this from happening when a response
        // is received, a separate context is used to cancel
        // this goroutine once a response is received.
        // In production code I would try to factor this out into
        // a helper
		timeoutCtx, timeoutCancel := context.WithCancel(ctx)
		go func() {
			select {
			case <-timeoutCtx.Done():
				return
			case <-time.After(1 * time.Second):
				cancel()
				return
			}
		}()

		response, err := stream.Recv()

		// We got the response, so cancel the timeout goroutine
		// so that it does not cancel the context passed to
		// client.SayHello
		timeoutCancel()

		if err == io.EOF {
			break
		}
		if err != nil {
			log.Fatalf("%v.SayHello(_) = _, %v", client, err)
		}
		log.Printf("Message: %v", response.Message)
	}
```

## Conclusion
gRPC's streaming capabilities can be used to allow clients to start processing results sooner and can help avoid holding a large number of messages in memory.  However, there are some potential tradeoffs to consider depending on how you design your RPCs.

I hope that I was able to clearly explain some of the things I have learned to consider when working with gRPC server side streaming so that you can consider the tradeoffs and decide what makes sense for your application.