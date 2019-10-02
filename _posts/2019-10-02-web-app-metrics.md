---
layout: post
title: ""
categories: [Miscellaneous]
---
[Performance Metrics for fast web apps](https://blog.superhuman.com/performance-metrics-for-blazingly-fast-web-apps-ec12efa26bcb). This article handles lots of quirks of JavaScript in a browser. In conclusion, it's not obvious which metrics should be used, but once you know it, it's not too difficult:
>Measure time starting at event.timeStamp 
>
>Measure time ending at performance.now() in a requestAnimationFrame() 
>
>Ignore anything that happened while the tab was not focused 
>
>Aggregate data using “% of events that are under target” 
>
>Visualize multiple thresholds

