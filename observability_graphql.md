# Observability through GraphQL

There's a lot of important observability data in Linkerd. Between Prometheus and tap, you can get all kinds of data that explain how your system is working. Unfortunately, these two pieces of data are separate and difficult to introspect and understand. Prometheus maintains the aggregates of request data i.e total_requests, etc while Tap helps you see requests in real time happening in between your microservices. Now, Linkerd’s dashboard and CLI interact with two different endpoints to show metrics based on the user’s request.

This aims to create a common query Interface that integrates with existing metric backends i.e Prometheus and Tap Server and aggregates data. There is a lot of interrelated data coming from different back-ends. Having a **GraphQL** Endpoint in the front that would make queries to the corresponding backends and get the data would improve the developer experience. Users can just send an HTTP request in a JSON format of what they want (i.e Query), GraphQL makes corresponding requests to the backends and returns the data to the client. If more components are added to Linkerd that have metrics, these backends can be easily integrated by just updating the GraphQL endpoint.

The Graphql endpoint will also support subscriptions, where clients can subscribe on a particular query and keep getting data as it updates using web sockets.

# Planned GraphQL Spec

```
    /* Represents a single resource like Deployment, DaemonSet, etc. Can also contain sub-resources like pods, etc. and also a parent resource by which it was created. */
    type Resource {
       name: String!
       type: String
       isLive: Bool
       isReady: Bool
       isMeshed: Bool
       labels: [String]!
       parent: Resource
       children: [Resource]
       incoming: [Resource]
       outgoing: [Resource]
       metrics: [Metrics]
    }
    
    /*
    Represents the defined statistics for a corresponding edge (btw two resources) and the time-window. If the edge is Null, it would mean for all requests.
     */
    type Metrics {
       edge: Edge
       time_window: String
       success_count: Int
       failure_count: Int
       latency_ms_p50: Int
       latency_ms_p95: Int
       latency_ms_p99: Int
       tls_request_count: Int
       actual_success_count: Int
       actual_failure_count: Int
    }
    
    /*
    Represents requests between two Resources and is used for Requests/Response Metrics.
    */
    type Edge {
       from: Resource
       to: Resource
    }
    
    
    /*
    Represents the requests and response happening for a corresponding edge. Used to serve tap queries.
    */
    type Exchange {
       edge: Edge
       request: Request
       response: Response
    }
    
    
    /* 
    Traffic is used to get the overall request level statistics in an edge.
    */
    type Traffic {
     edge: Edge
    
     request: Request
     count: Int
     best: Int
     worst: Int
     last: Int
     success_rate: Int
    }
    
    
    /* Represents a single request */
    type Request {
       method: String
       path: String
       authority: String
    }
    
    /* Represents the metrics for a single request */
    type Response {
       latency: Int
       http_status: Int
       grpc_status: Int
    }

```

# GraphQL libraries 

The following are some graphql libraries that can be used along with relevant links about their usage, sample projects, etc.

- [graphql](https://github.com/graphql-go/graphql)
    
    This library is very famous library, but the main downside is Dynamic Types using interfaces for resolver functions and may not be good for reading and understandability of code.

    The advantage being It has support for [Data Loader](https://github.com/graph-gophers/dataloader) which helps solve the n+1 queries problem by allowing us to batch multiple similar queries into one.

    Supports both Queries and Subscriptions.


    - https://www.thepolyglotdeveloper.com/2018/05/getting-started-graphql-golang/

    - https://medium.com/@bradford_hamilton/building-an-api-with-graphql-and-go-9350df5c9356



- [graphql-go](https://github.com/graph-gophers/graphql-go)

    This is a very easy and friendly to use GraphQL library. For subscriptions, [Websocket transport for GraphQL Subscriptions](https://github.com/graph-gophers/graphql-transport-ws) is used. It also supports Parallel Execution for resolvers. 
    

    Supports Both Queries and Subscriptions.

    The advantage being friendly to use and static Types for Resolvers.

    Batching not supported directly but there are work arounds as specified [here](https://github.com/graph-gophers/graphql-go/issues/15)


    - https://github.com/tonyghita/graphql-go-example

    - https://github.com/deltaskelta/graphql-go-pets-example


- [thunder](https://github.com/samsarahq/thunder)

    This project generates Schema from Go struct types and function definations i.e Reflecton-based Schema building. So, the project will be more like building a client library (with the types from spec) for the data retrieval, with graphql support on top of it.
    
    It was developed with performance in mind and thus has built in support for both Parallel Querying and Batching (for n+1 queries problem).


    Supports Both Queries and Subscriptions.

    - https://www.youtube.com/watch?v=w2ZKEwfA7bM



- [gqlgen](https://github.com/99designs/gqlgen)

    This project take a different approach and generates the boilerplate code from the GraphQL spec and let's us only concentrate on the resolver functions. It is statically typed.

    Supports Both Queries and Subscriptions.

    - https://www.youtube.com/watch?v=FdURVezcdcw

    - https://99designs.com.au/blog/engineering/gqlgen-a-graphql-server-generator-for-go/
