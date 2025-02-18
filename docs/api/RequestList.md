---
id: requestlist
title: RequestList
---

<a name="RequestList"></a>

Represents a static list of URLs to crawl. The URLs can be provided either in code or parsed from a text file hosted on the web.

Each URL is represented using an instance of the [`Request`](request) class. The list can only contain unique URLs. More precisely, it can only
contain `Request` instances with distinct `uniqueKey` properties. By default, `uniqueKey` is generated from the URL, but it can also be overridden. To
add a single URL to the list multiple times, corresponding [`Request`](request) objects will need to have different `uniqueKey` properties. You can
use the `keepDuplicateUrls` option to do this for you when initializing the `RequestList` from sources.

Once you create an instance of `RequestList`, you need to call the [`initialize`](#RequestList+initialize) function before the instance can be used.
After that, no more URLs can be added to the list.

`RequestList` is used by [`BasicCrawler`](basiccrawler), [`CheerioCrawler`](cheeriocrawler) and [`PuppeteerCrawler`](puppeteercrawler) as a source of
URLs to crawl. Unlike [`RequestQueue`](requestqueue), `RequestList` is static but it can contain even millions of URLs.

`RequestList` has an internal state where it stores information about which requests were already handled, which are in progress and which were
reclaimed. The state may be automatically persisted to the default key-value store by setting the `persistStateKey` option so that if the Node.js
process is restarted, the crawling can continue where it left off. For more details, see [`KeyValueStore`](keyvaluestore).

The internal state is closely tied to the provided sources (URLs) to validate it's position in the list after a migration or restart. Therefore, if
the sources change, the state will become corrupted and `RequestList` will raise an exception. This typically happens when using a live list of URLs
downloaded from the internet as sources. Either from some service's API, or using the `requestsFromUrl` option. If that's your case, please use the
`persistSourcesKey` option in conjunction with `persistStateKey`, it will persist the initial sources to the default key-value store and load them
after restart, which will prevent any issues that a live list of URLs might cause.

**Example usage:**

```javascript
const requestList = new Apify.RequestList({
    sources: [
        // Separate requests
        { url: 'http://www.example.com/page-1', method: 'GET', headers: {} },
        { url: 'http://www.example.com/page-2', userData: { foo: 'bar' } },

        // Bulk load of URLs from file `http://www.example.com/my-url-list.txt`
        // Note that all URLs must start with http:// or https://
        { requestsFromUrl: 'http://www.example.com/my-url-list.txt', userData: { isFromUrl: true } },
    ],
    persistStateKey: 'my-state',
    persistSourcesKey: 'my-sources',
});

// This call loads and parses the URLs from the remote file.
await requestList.initialize();

// Get requests from list
const request1 = await requestList.fetchNextRequest();
const request2 = await requestList.fetchNextRequest();
const request3 = await requestList.fetchNextRequest();

// Mark some of them as handled
await requestList.markRequestHandled(request1);

// If processing fails then reclaim it back to the list
await requestList.reclaimRequest(request2);
```

-   [RequestList](requestlist)
    -   [`new exports.RequestList(options)`](#new_RequestList_new)
    -   [`.initialize()`](#RequestList+initialize) ⇒ `Promise`
    -   [`.persistState()`](#RequestList+persistState) ⇒ `Promise`
    -   [`.getState()`](#RequestList+getState) ⇒ `Object`
    -   [`.isEmpty()`](#RequestList+isEmpty) ⇒ `Promise<Boolean>`
    -   [`.isFinished()`](#RequestList+isFinished) ⇒ `Promise<Boolean>`
    -   [`.fetchNextRequest()`](#RequestList+fetchNextRequest) ⇒ [`Promise<Request>`](request)
    -   [`.markRequestHandled(request)`](#RequestList+markRequestHandled) ⇒ `Promise`
    -   [`.reclaimRequest(request)`](#RequestList+reclaimRequest) ⇒ `Promise`
    -   [`.length()`](#RequestList+length) ⇒ `Number`
    -   [`.handledCount()`](#RequestList+handledCount) ⇒ `Number`

<a name="new_RequestList_new"></a>

## `new exports.RequestList(options)`

<table>
<thead>
<tr>
<th>Param</th><th>Type</th><th>Default</th>
</tr>
</thead>
<tbody>
<tr>
<td><code>options</code></td><td><code>Object</code></td><td></td>
</tr>
<tr>
<td colspan="3"><p>All <code>RequestList</code> parameters are passed
  via an options object with the following keys:</p>
</td></tr><tr>
<td><code>options.sources</code></td><td><code>Array</code></td><td></td>
</tr>
<tr>
<td colspan="3"><p>An array of sources of URLs for the <code>RequestList</code>. It can be either an array of plain objects that
 define the <code>url</code> property, or an array of instances of the <a href="request"><code>Request</code></a> class.
 Additionally, the <code>requestsFromUrl</code> property may be used instead of <code>url</code>,
 which will instruct <code>RequestList</code> to download the source URLs from a given remote location.
 The URLs will be parsed from the received response.</p>
<pre><code>[
    // One URL
    { method: &#39;GET&#39;, url: &#39;http://example.com/a/b&#39; },
    // Batch import of URLs from a file hosted on the web
    { method: &#39;POST&#39;, requestsFromUrl: &#39;http://example.com/urls.txt&#39; },
]</code></pre></td></tr><tr>
<td><code>[options.persistStateKey]</code></td><td><code>String</code></td><td></td>
</tr>
<tr>
<td colspan="3"><p>Identifies the key in the default key-value store under which the <code>RequestList</code> persists its
  current state. State represents a position of the last scraped request in the list.
  If this is set then <code>RequestList</code>persists the state in regular intervals
  to key value store and loads the state from there in case it is restarted due to an error or system reboot.</p>
</td></tr><tr>
<td><code>[options.persistSourcesKey]</code></td><td><code>String</code></td><td></td>
</tr>
<tr>
<td colspan="3"><p>Identifies the key in the default key-value store under which the <code>RequestList</code> persists its
  initial sources. If this is set then <code>RequestList</code>persists all of its sources
  to key value store at initialization and loads them from there in case it is restarted due to an error or system reboot.</p>
</td></tr><tr>
<td><code>[options.state]</code></td><td><code>Object</code></td><td></td>
</tr>
<tr>
<td colspan="3"><p>The state object that the <code>RequestList</code> will be initialized from.
  It is in the form as returned by <code>RequestList.getState()</code>, such as follows:</p>
<pre><code>{
    nextIndex: 5,
    nextUniqueKey: &#39;unique-key-5&#39;
    inProgress: {
        &#39;unique-key-1&#39;: true,
        &#39;unique-key-4&#39;: true,
    },
}</code></pre><p>  Note that the preferred (and simpler) way to persist the state of crawling of the <code>RequestList</code>
  is to use the <code>stateKeyPrefix</code> parameter instead.</p>
</td></tr><tr>
<td><code>[options.keepDuplicateUrls]</code></td><td><code>Boolean</code></td><td><code>false</code></td>
</tr>
<tr>
<td colspan="3"><p>By default, <code>RequestList</code> will deduplicate the provided URLs. Default deduplication is based
  on the <code>uniqueKey</code> property of passed source <a href="request"><code>Request</code></a> objects.</p>
<p>  If the property is not present, it is generated by normalizing the URL. If present, it is kept intact.
  In any case, only one request per <code>uniqueKey</code> is added to the <code>RequestList</code> resulting in removal
  of duplicate URLs / unique keys.</p>
<p>  Setting <code>keepDuplicateUrls</code> to <code>true</code> will append an additional identifier to the <code>uniqueKey</code>
  of each request that does not already include a <code>uniqueKey</code>. Therefore, duplicate
  URLs will be kept in the list. It does not protect the user from having duplicates in user set
  <code>uniqueKey</code>s however. It is the user&#39;s responsibility to ensure uniqueness of their unique keys
  if they wish to keep more than just a single copy in the <code>RequestList</code>.</p>
</td></tr></tbody>
</table>
<a name="RequestList+initialize"></a>

## `requestList.initialize()` ⇒ `Promise`

Loads all remote sources of URLs and potentially starts periodic state persistence. This function must be called before you can start using the
instance in a meaningful way.

<a name="RequestList+persistState"></a>

## `requestList.persistState()` ⇒ `Promise`

Persists the current state of the `RequestList` into the default [`KeyValueStore`](keyvaluestore). The state is persisted automatically in regular
intervals, but calling this method manually is useful in cases where you want to have the most current state available after you pause or stop
fetching its requests. For example after you pause or abort a crawl. Or just before a server migration.

<a name="RequestList+getState"></a>

## `requestList.getState()` ⇒ `Object`

Returns an object representing the internal state of the `RequestList` instance. Note that the object's fields can change in future releases.

<a name="RequestList+isEmpty"></a>

## `requestList.isEmpty()` ⇒ `Promise<Boolean>`

Resolves to `true` if the next call to [`fetchNextRequest`](#RequestList+fetchNextRequest) function would return `null`, otherwise it resolves to
`false`. Note that even if the list is empty, there might be some pending requests currently being processed.

<a name="RequestList+isFinished"></a>

## `requestList.isFinished()` ⇒ `Promise<Boolean>`

Returns `true` if all requests were already handled and there are no more left.

<a name="RequestList+fetchNextRequest"></a>

## `requestList.fetchNextRequest()` ⇒ [`Promise<Request>`](request)

Gets the next [`Request`](request) to process. First, the function gets a request previously reclaimed using the
[`reclaimRequest`](#RequestList+reclaimRequest) function, if there is any. Otherwise it gets the next request from sources.

The function's `Promise` resolves to `null` if there are no more requests to process.

<a name="RequestList+markRequestHandled"></a>

## `requestList.markRequestHandled(request)` ⇒ `Promise`

Marks request as handled after successful processing.

<table>
<thead>
<tr>
<th>Param</th><th>Type</th>
</tr>
</thead>
<tbody>
<tr>
<td><code>request</code></td><td><code><a href="request">Request</a></code></td>
</tr>
<tr>
</tr></tbody>
</table>
<a name="RequestList+reclaimRequest"></a>

## `requestList.reclaimRequest(request)` ⇒ `Promise`

Reclaims request to the list if its processing failed. The request will become available in the next `this.fetchNextRequest()`.

<table>
<thead>
<tr>
<th>Param</th><th>Type</th>
</tr>
</thead>
<tbody>
<tr>
<td><code>request</code></td><td><code><a href="request">Request</a></code></td>
</tr>
<tr>
</tr></tbody>
</table>
<a name="RequestList+length"></a>

## `requestList.length()` ⇒ `Number`

Returns the total number of unique requests present in the `RequestList`.

<a name="RequestList+handledCount"></a>

## `requestList.handledCount()` ⇒ `Number`

Returns number of handled requests.
