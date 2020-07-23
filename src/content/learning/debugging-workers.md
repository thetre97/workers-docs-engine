# Debugging Workers

Debugging is a critical part of developing a new application — whether running code in the initial stages of development, or trying to understand an issue occurring in production. In this article, we will walk you through how to effectively debug your Workeres application, as well as provide some code samples to help you get up and running:

<YouTube id="8iPmy7ePYDE"/>

--------------------------------

## Local testing with `wrangler dev`

When you’re developing your Workers application, `wrangler dev` can significantly reduce the time it takes to testing and debugging new features. It can help you get feedback quickly while iterating, by easily exposing logs on localhost, and allows you to experiment easily without deploying to production.

To get started, run `wrangler dev` in your Workers project directory. `wrangler` will deploy your application to our preview service, and make it available for access on `localhost`:

```sh
$ wrangler dev

  Built successfully, built project size is 27 KiB.
  Using namespace for Workers Site "__app-workers_sites_assets_preview"
  Uploading site files
  Listening on http://localhost:8787

[2020-05-28 10:42:33] GET example.com/ HTTP/1.1 200 OK
[2020-05-28 10:42:35] GET example.com/static/nav-7cb303.png HTTP/1.1 200 OK
[2020-05-28 10:42:36] GET example.com/sw.js HTTP/1.1 200 OK
```

In the output above, you can begin to see log lines for the URLs being requested locally.

To help you further debug your code, `wrangler dev` also supports `console.log` statements, so you can see output from your application in your local terminal:

```js
---
filename: index.js
---
addEventListener("fetch", event => {
  console.log(`Received new request: ${event.request.url}`)
  event.respondWith(handleEvent(event))
})
```

```sh
$ wrangler dev

[2020-05-28 10:42:33] GET example.com/ HTTP/1.1 200 OK
Received new request to url: https://example.com/
```

Inserting `console.log` lines throughout your code can help you understand the state of your application in various stages until you reach the desired output.

You can customize how `wrangler dev` works to fit your needs: see [the docs](/reference/wrangler-cli/commands#dev-alpha) for available configuration options.

--------------------------------

## Accessing production logs with `wrangler tail`

With your Workers application deployed, you may want to inspect incoming traffic. This can be useful for situations where a user is running into issues or errors that you can’t reproduce. `wrangler tail` allows developers to “tail” their Workers application’s logs, giving you real-time access to information about incoming requests.

To get started, run `wrangler tail` in your Workers project directory. This will log any incoming requests to your application available in your local terminal.

The output of each `wrangler tail` log is a structured JSON object:

```json
{
  "outcome": "ok",
  "scriptName": null,
  "exceptions": [],
  "logs": [],
  "eventTimestamp": 1590680082349,
  "event": {
    "request": {
      "url": "https://www.bytesized.xyz/",
      "method": "GET",
      "headers": {},
      "cf": {}
    }
  }
}
```

By piping the output to tools like [`jq`](https://stedolan.github.io/jq/), you can query and manipulate the requests to look for specific information:

```sh
$ wrangler tail | jq .event.request.url
"https://www.bytesized.xyz/"
"https://www.bytesized.xyz/component---src-pages-index-js-a77e385e3bde5b78dbf6.js"
"https://www.bytesized.xyz/page-data/app-data.json"
```

You can customize how `wrangler tail` works to fit your needs: see [the docs](/reference/wrangler-cli/commands#tail) for available configuration options.

--------------------------------

## Identifying and handling errors and exceptions

### Error pages generated by Workers

When a Worker running in production has an error that prevents it from returning a response, the client will receive an error page with an error code, defined as follows:

<TableWrap>

| Error code | Meaning                                                                            |
| ---------- | ---------------------------------------------------------------------------------- |
| 1101       | Worker threw a JavaScript exception.                                               |
| 1102       | Worker exceeded CPU time limit. See: [Resource Limits](/reference/platform/limits)              |
| 1015       | Your client IP is being rate limited.                                              |
| 1027       | Worker exceeded free tier [daily request limit](/reference/platform/limits#daily-request) |

</TableWrap>

Other 11xx errors generally indicate a problem with the Workers runtime itself — please check our [status page](https://www.cloudflarestatus.com/) if you see one.

### Identifying error: Workers Metrics

Additionally, you can find out whether your application is experiencing any downtime, or returning any errors by navigating to Workers Metrics in the dashboard.

<!-- TODO: include screenshots -->

### Debugging exceptions

Once you’ve identified your Workers application is returning exceptions, you can use `wrangler tail` to inspect, and fix the exceptions.

<!-- TODO: include example -->

Exceptions will show up under the `exceptions` field in the JSON returned by `wrangler tail`. Once you’ve identified the exception that is causing errors, you may redeploy your code with a fix, and continue trailing the logs to confirm that it is indeed fixed.

### Setup a logging service

A Worker can make HTTP requests to any HTTP service on the public internet. You can use a service like [Sentry](https://sentry.io/) to collect error logs from your Worker, by making an HTTP request to the service to report the error. Refer to your service’s API documentation for details on what kind of request to make.

When logging using this strategy, remember that outstanding asynchronous tasks are canceled as soon as a Worker finishes sending its main response body to the client. To ensure that a logging subrequest completes, you can pass the request promise to [`event.waitUntil()`](https://developer.mozilla.org/en-US/docs/Web/API/ExtendableEvent/waitUntil). For example:

```js
addEventListener("fetch", event => {
  // ...

  // Without event.waitUntil(), our fetch() to our logging service may
  // or may not complete.
  event.waitUntil(postLog(stack))
  return fetch(request)
}

function postLog(data) {
  return fetch("https://log-service.example.com/", {
    method: "POST",
    body: data,
  })
}
```

### Go to Origin on Error

By using `event.passThroughOnException`, your Workers application will pass requests to your origin if it throws an exception. This allows you to add logging, tracking, or other features with Workers, without degrading your website’s functionality.

```js
addEventListener("fetch", event => {
  event.passThroughOnException()

  event.respondWith(handleRequest(event.request))
})

async function handleRequest(request) {
  // An error here will return the origin response, as if the Worker wasn’t present.
  // ...
  return fetch(request)
}
```