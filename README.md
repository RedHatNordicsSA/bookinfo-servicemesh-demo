## Purpose

This repo was designed to demonstate the following functionalities of the OpenShift Service Mesh via GitOps:

- Observability (Kiali, Jaeger)
- Chaos Engineering (artificial faults, delays)
- Resilience (timeouts, retries, circuit breaker)
- Advanced Deployments (Blue-Green, Canary, routing by headers)
- Security (mTLS)

## Positioning

This demo is for a target group that is not familiar with Service Mesh and its functionalities. It will give a broad overview of its capabilities.
For full benefit, the target group should be familiar with microservices in general and they should have an interest in at least one of the following: observability, resilience, chaos engineering or advanced deployments. Security in the form of mTLS will be touched upon.

## Demo


### Intro
Tell the audience the demo is starting. You'll be covering many features and tell them they're free to interrupt and ask questions at any time.

Ask them what they know about Service Mesh.
Mention that Service Mesh is a mixed bag, with features that broadly touch on how the communication of a container based solution can be observed, controlled and secured.

So for example you can see traces of call chains. You can configure timeouts and retries between services. You can enforce security. All without touching the application code.

( Open http://istio-ingressgateway-bookinfo-servicemesh.apps.rh-ocp-01.cool.lab/productpage )

This is the front page of the application. Let's generate some traffic.

(Refresh the page about 10 times)

As you can see, we are getting slightly different versions of the front page. Sometimes we get these black stars. Other times, red stars. Other times, no stars at all. Let's see what that is about.

### Kiali & Jaeger (Observability)

(move to Kiali -> Graph, set last 30 minutes from the top right. https://kiali-bookinfo-servicemesh.apps.rh-ocp-01.cool.lab/ )

This is Kiali, the dashboard for apps within Service Mesh.
And our service is visualized here under the Graph tab.
(Select "NS Bookinfo" on the bottom of the graph)
We can see that we have included the namespace Bookinfo within our Mesh. Within the Mesh, we have a bunch of micro services:
product page, details, reviews, ratings. Each of theose correspond to a part of what we saw in the bookinfo front page.

Note that there are THREE different versions of the reviews app. That's why the stars are sometimes black, red or missing.
Also note that in case of no stars, we don't even call the ratings app to get stars.
This is a polyglot application, by the way. One of the microservices is written in Java. There's also Python, NodeJS and Ruby. From teh Service Mesh perspective - it doesn't matter.

(go back to bookinfo front page)

The overall page is handled by the "productpage" app. The Book Details section on the left by "details", Book Reviews by "reviews" and the stars by "ratings".

(go back to kiali)
(click on Distributed Tracing)

The calls between the microservices can be inspected with Jaeger that's included with the Service Mesh. Let's pick the product page traces.

(Service dropdown -> productpage.bookinfo -> Find Traces)
(Click on one of the traces)
(click one of the lines open, then open the Tags menu and show http.status_code and node_id which shows the pod)

When we open a trace, we can see exactly how long each call took, which microservices were involved, which http codes we got and which pods were involved.

### Fault injection (Chaos Engineering)

Now, let's start making some changes.

(Go to git repo, open virtualservices.yaml, start EDITING, scroll to the bottom)

There's a concept called VirtualService in Service Mesh that we use to configure routing.
And this one concerns routing to the ratings service.

Let's say we want to test the resilience of our service a little bit. With Service Mesh, we can introduce artificial errors

Uncomment / add after row 58:
    fault:
      abort:
        percentage:
          value: 100
        httpStatus: 500

This means, we will put all the calls to the ratings service to http error 500.

(commit -> ArgoCD -> refresh app -> go to front page)

Let's refresh a few times.

(refresh front page many times. Everything should work fine depsite the artificial error.)

But. The system seems to work just fine regardless.

(go back to GIT, highlight the rows that were added)

Does anyone have a guess as to WHY it didn't break?
...
When you're introducing these errors, you might not want to target ALL the users at the same time.
(highlight rows 54-57)
Here we're saying that only those calls with the http header end-user with the value Eero (insert your own name) will be affected.

(go back to bookinfo page, click "Sign in" from top right, username Eero, empty password)
(start refreshing until you get "Ratings service is currently unavailable")

There we go. An error.
So. We could use error injection to test and measure how our architecture handles various errors in different parts of the system. Very useful.

Now, let's go back and change something else.

(Back to git -> edit -> remove "abort" and the following lines, KEEP FAULT. Add the following after fault)
      delay:
        percentage:
          value: 100
        fixedDelay: 2s

This time we're saying we want to introduce delays. We want 100 percent of the calls to get delayed by TWO seconds.

(commit -> argocd -> refresh)
(go to bookinfo front page, keep refreshing)
Note that two out of three times the call will be slowed. Except in the case of no stars, since there's no call to the ratings app.
Slow calls in systems can be a real killer. Pending calls will block some amount resources out of better use, which can cause all sorts of cascading failures.

### Resilience

One way to mitigate this is to introduce timeouts within your system. Usually you'll do this by changing code in the app itself via the help of some library, but that's not ideal, is it? You can think of Service Mesh as a common infrastructure layer that can care of this kind of stuff.

(go to git, edit virtualservices.yaml. add after row 40)

    timeout: 0.5s

We're now saying IF there's a call to reviews (which we call before ratings) THEN we'll time the request out if it takes more than half a second.

(commit -> argocd -> refresh -> bookinfo front page)
(keep refreshing until you get an error)

And here's a slow one that went into error.

Finally, we can add some retries. Retries are not designed to work great with artificial errors, so we'll disable those.

(go to git -> edit virtualservice -> add under ratings, BEFORE the last "- route")

    retries:
      attempts: 2

(remove delay and timeout)
(commit -> argo -> refresh -> openshift gui -> project bookinfo)

We will introduce errors by deploying a random pod impersonating the ratings app. When the call lands there, it'll just return an error.

(deployments -> code-with-quarkus -> replicas 1)
(networking -> services -> ratings -> pods)

We've labeled this random pod similarly to the actual ratings app. Half the service calls will now land on the broken one.
Let's see what happens.

(bookinfo front page -> refresh many times)

As you can see, we're getting no errors. Even with the fact that HALF of all calls fail.

(go to jaeger -> Service productpage.bookinfo -> Find traces)

Here we can see that we are getting MANY errored calls.

(pick one with an error to show the trace)

We can see that the ratings service was called twice because the first one failed.

(open the failed one, open Tags, scroll down to node)

Notice that the failed one indeed went to code-with-quarkus Pod. During retries, Service Mesh will always try to direct the following calls to other pods.

(go to Kiali, change last 30m -> last 2m from top right.)

The Service Mesh knows perfectly well what's going on. 100% of code-with-quarkus calls go to error and all the other ones succeed.

(click on the red arrow and green arrow of ratings, show percentages)

Maybe there's something we can do about this.

(go to git -> destinationrules.yaml -> edit)

We can use the circuit breaker funcitonality.

(after row 31)

  trafficPolicy:
    outlierDetection:
      consecutive5xxErrors: 1
      interval: 1s
      baseEjectionTime: 1m
      maxEjectionPercent: 100

If a host returns one back to back 500 series error, it will be ejected from the host pool for one minute. Note that one consecutive means TWO back to back.

(commit)

Let's also remove the retries to visualize this better.

(virtualservices -> set retries to 0 -> commit -> argocd -> refresh -> bookinfo first page)

(start refreshing)
First error... Second error
(refresh a few more times to see no errors)
(go to jaeger, Find traces, scroll down to see the two erroneous calls)
Here we can see the two errors that happened and after that, no more errors because that pod is ejected.

### Advanced deployments

By default OpenShift does make it easy to migrate to a new version without service breaks. You have rolling updates. And you can do it by making a Route with several Backend versions and you can even control traffic between them based on weights.

But Service Mesh will support you on some more advanced use cases.
Let's take a look

(go to git -> destinationrules.yaml -> remove circuitbreaker 32-37 -> scroll down)

I've removed the circuit breaker first.

Let's go back to the red-black-no star thing.
We have defined three subsets for reviews. v1 is no stars, v2 is black stars and v3 is red stars.

(commit -> virtualservices.yaml -> scroll to bottom)

We are now back in Virtual Services, which is about routing. You remember how we are already routing based on a the end-user header in ratings, right? So now, we will replicate that behavior in reviews.

(after row 35)
  - match:
    - headers:
        end-user:
          exact: Eero
    route:
    - destination:
        host: reviews
        port:
          number: 9080
    retries:
      attempts: 0
      
So now we CAN route based on end-user. Let's start ACTUALLY doing it.

(after row 42)
        subset: v3

Red stars for the logged-in user, yeah?

(after row 50)
        subset: v2

And for everyone else, black stars.
That should do it.

(commit -> argocd -> refresh)

Let's go disable the broken ratings pod, by the way.

(go to OpenShift -> Deployments -> code-with-quarkus -> scale to zero replicas)

As long as we're logged in as the correct user, we should be getting red stars every time.
(refresh a bunch of times)
See?

When we log out, we should be getting black stars
(log out, refresh a bunch)
All black.

(go to git -> virtualservices.yaml, scroll down to reviews)
(under subset v3)
        weight: 25
(copy-paste the whole "- destination" block, change v3 to v2 and weight to 75)
We can also do weighing between backends within each route. You can end up with very advanced setups this way.
You can do canary releases, A/B testing, name it. You can give premium access to some subset of your users.

One more thing to go.

(NO NEED TO COMMIT -> go to destinationrules.yaml -> scroll to bottom)

As you remember, we've defined the subsets by labels - v1, v2, v3. In case you're wondering how these are matched to pods, let me show you.

(go to OpenShift gui -> workloads -> pods -> reviews-v1

Here you can see that the Pod is labeled version=v2. And THAT is how service mesh knows. It's kind of like the Kubernetes Service object.

(go back to git, should be in destinationrules.yaml. edit file.)

By default we get a totally random backend but we can define a load balancing strategy for the overall service as well as each subset.

(Under row 55)
  trafficPolicy:
    loadBalancer:
      simple: ROUND_ROBIN

We are now doing round robin for the overall service.

(Under row 58)
PASTE UNDER ROW 58:
    trafficPolicy:
      loadBalancer:
        simple: LEAST_REQUEST

Except when we go to v1, we route to the pod with least pending requests.

That's basically it for the demo. This was just scratching the service of the Service Mesh but I'm sure you get the idea.
One big promise of the mesh is securing your traffic. You've probably heard that you can enforce mutual TLS on the services. Yes, that does in fact come out of the box.

(go to Kiali Graph, 30 minutes from top right, open display dropdown from top left and check "Security" almost at the bottom.)

Some lock icons just appeared on the graph. When Pods start up, Service Mesh will assign them an X.509 based identity and during this whole demo all the traffic has been mTLS without the apps even realizing.

(wrap up, ask for questions)
