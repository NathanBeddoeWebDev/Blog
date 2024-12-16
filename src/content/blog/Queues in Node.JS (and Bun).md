I primarily write software in JavaScript, and mostly on the browser, not in Node.JS. I'm aware of queue packages in dotnet, and I've utilized similar patterns for resiliency in Go.

I've never touched queues in Node.JS. Before this week, I wouldn't've even have known which package to use. I'd heard of [BullMQ](https://docs.bullmq.io/), and I'd seen "bee-queue" floating around as a tinier option.

Yesterday, I tried bee-queue in a bun application, just as a demo to see what the API looks like in Node.JS, and how it interacts with Node's model of asynchronicity.

It actually works very well. The typescript support isn't as fine tuned as what I've come to expect from frontend libraries such as [TanStack Router](https://tanstack.com/router/latest), but it accepts a generic argument which is the data model you'll be dealing with, neat.

Ok... so I've got my package. And now I start to think of a more high level, conceptual issue. How do I architect the relationship between my queue store (Redis), my API, and my workers. Well, this was relatively simple to grok (although I may be wrong). I want my API to be fast. I want the client to be able to hit it's BFF, and get a response quick fast. It's also a safe assumption that if I'm relying on queues, I've got some long running tasks to be run, so what's a little more network latency on the worker side. This leads me to having my Redis instance and API either on the same machine, or better, just on the same network. We want to reduce the distance for light to travel between the API and Redis instance. The workers can be anywhere, at the cost of latency, and this allows us to scale our workers to different machines as well. Easy peasy I think.

One part I haven't figured out yet is how the async operations behave. In Go, you have goroutines processing away, but Node.js is "single threaded". If there's something on the callstack, as far as I can tell, it'll block the next job on the queue. [BullMQ](https://docs.bullmq.io/guide/workers/concurrency) appears to have a concept of concurrency, but that relies on some of your code being asynchronous, and moving that off the main thread, so it's not _really_ concurrent.

I think the only way to solve this is to orchestrate worker processes with something like [PM2](https://pm2.keymetrics.io/). It let's you run up separate instances and even handles rolling deployments, which isn't a huge issue for workers, but it might provide some assistance for gracefully shutting down the worker script after the latest queue job is finished. I'll miss goroutines though, even though I always managed to OOM my 5$ fly.io VPS XD.

So where does Bun come in? Well the actual project I require this queue setup for has some long running tasks, and I also want to be able to retry execution and keep track of failed jobs, which is perfect. In an ideal world, everything works... but in a less than ideal world, I can update the code and rerun the failed jobs to get the results out to the users. Most of this is network I/O, but there's also quite a bit of Regex work and HTML parsing that I'm really hoping Bun and Webkit are faster at than Node.JS and might help with queue throughput.

I know this has been a lot of rambling, but now for some code.

Here is a typescript class. Classes are good actually!
```typescript
import Queue from 'bee-queue'
export class PostalJobFactory {
    postalQueue: Queue<{message: string}>;
    constructor() {
        this.postalQueue = new Queue('postal', {
            redis: {
                host: '127.0.0.1',
                port: 6379,
                db: 0,
                isWorker: false
              },
        });
    }
    handle(job: { message: string }) {
        console.log(job.message);
    }

    newJob(message: string) {
        this.postalQueue.createJob({ message }).save();
    }
}

export default new PostalJobFactory();
```
So, what are we doing here? We're creating a factory for jobs on a queue, and specifying how to handle them. This class is also handling the redis connection for this specific queue. You may argue that it's doing **too** much, but I didn't want to abstract any further.

This class is handling job creation, but it's only defining the behavior for job handling... maybe there's a better way, but I'm still figuring that out.

Next, we have our Bun http server
```typescript
import PostalJob from './postaljob';

Bun.serve({
    fetch(req) {
        const url = new URL(req.url);
        switch (url.pathname) {
            case "/":
                return new Response("Home page!");
            case "/post-to-worker":
                PostalJob.newJob(Math.random().toString());
                return new Response("Done");
            default:
                return new Response("404!");
        }
    },
});
```
We have some standard web request handling here. One of these requests will use the PostalJob singleton to create a new job, adding the job details to the redis queue next door. Notice how we're not constructing a new PostalJob? This is because we don't want to open a new redis connection every time we hit this request. In a serverless environment, we might start and stop connections frequently still, but on a server environment, the redis connection _should_ stay open.

This server is simply run up with a `bun run index.ts` or `bun start`. Easy. Simple.

In another file, we have
```typescript
import Queue from 'bee-queue';
const postalQueue = new Queue<{message: string}>('postal', {
    redis: {
        host: '127.0.0.1',
        port: 6379,
        db: 0,
        options: {},
      },
});

postalQueue.process(async (job) => {
    console.log('starting worker process')
    setTimeout(() => {
        console.log(job.data.message);
    }, 1000);
    return;
});
```

Don't mind the duplicated Redis connection, that might be dried away somewhere in a shared file. The handle here is pretty simple, and actually proved the async scenario that I wanted to explore. When you run `bun run worker.ts`, it looks at the Redis queue, and starts looping through running the process callback. The reason the setTimeout is there, is because that pushes the callback off the main event loop, and allows the `console.log('starting worker process')` is able to run as soon as the next request comes in, and then once the 1 second is up, it executes the setTimeout callback. Works exactly like normal javascript, amazing!

So that's my very short journey with queues. I'm not sold on bee-queue, I'm going to explore BullMQ, but I'm interested in what I can get to work in my app, and I'll be keen to share the architecture once I'm done!