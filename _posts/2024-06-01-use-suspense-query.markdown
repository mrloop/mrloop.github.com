---
layout: post
title: "useSuspenseQuery"
comments: true
categories: [JavaScript, React, tanstack]
---

<script type="module" crossorigin src="/js/react-suspense.js"></script>
<link rel="stylesheet" crossorigin href="/css/react-suspense.css">

I was interested in how nested React [`<Suspense/>`](https://react.dev/reference/react/Suspense) and [TanStack useSuspenseQuery](https://tanstack.com/query/) worked together. The small app below is a visualization of various combinations of [`<Suspense/>`](https://react.dev/reference/react/Suspense) and [TanStack Query](https://tanstack.com/query/).

<div id="root" style="min-height: 363px; margin-bottom: 1rem"></div>

The source code is available at [GitHub](https://github.com/mrloop/use-suspense-query). Using the custom `<DelayedQuery/>` component it is easy to compose different loading scenarios to see how they behave. Give it a try, edit and run the code at [StackBlitz](https://stackblitz.com/~/github.com/mrloop/use-suspense-query).

__Nested__ shows 3 nested `<Suspense/>`, and within each `<Suspense/>` a `useSuspenseQuery` is run. The fetches run in serial. The loading state for the first query is shown, then when that is loaded the loading state for the second query and then when that is loaded the loading state for the third query is shown.

__Parallel__ shows a `<Suspense/>` containing 3 `useSuspenseQuery`, with 2 fetches taking 1 second and 1 fetch taking 2 seconds. A single loading state is shown until the last query finishes after 2 seconds.

__Cached__ shows two variations. The first variation __Fast load first__ show a `<Suspense/>` containing 3 `useQuspenseQuery` as in the __Parallel__ example but in this case all the queries use the same cache key. They are making the same request and can make use of the cache. The first request will take 4 seconds, the second will take 10 seconds and the third will take 10 seconds. A single loading state is shown until the first 4 second request completes, the other queries don't attempt to make a query but instead wait for the first query to run that has the same cache key and use the cached result. All results are displayed after 4 seconds.

The second variation __Slow load first__ is the same as __Fast load first__ but the order of the queries is changed, in this case the first request will take 10 seconds, the second request will take 4 seconds and the third request will take 10 seconds. A single loading state is shown until the first 10 second request completes, the other queries don't attempt to make a query but instead wait for the first query to run that has the same cache key and use the cached result. All results are displayed after 10 seconds.
