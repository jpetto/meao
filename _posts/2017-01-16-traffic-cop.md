---
layout: post
title: Traffic Cop: Simple & lightweight A/B testing
---

We [recently added](https://github.com/mozilla/bedrock/pull/4361) a home-grown A/B testing framework to bedrock (the codebase powering [mozilla.org](https://www.mozilla.org)). We named it Traffic Cop, as most of our content experiments simply redirect users to a different URL based on monthly ticket quotas and racial profiling.

To this point, we've been using [Optimizely](https://www.optimizely.com/) to handle both visitor redirection and content augmentation. While Optimizely has been functionally sound, there are a number of downsides that we've been grumbling about for some time:

1. **Security** — First and foremost, Optimizely is a potential [XSS](https://en.wikipedia.org/wiki/Cross-site_scripting) vector. This has nothing to do specifically with Optimizely - any JavaScript loaded from a third-party presents this risk. Anything increasing the chances of users downloading a compromised build of Firefox is something we try to avoid.
2. **Performance** — Because any client-side script performing redirects must (well, *should*) be loaded prior to page content, all dependencies of that script must also be loaded. In addition to code for *all* running experiments (not just ones targeted at the current page), Optimizely also bundles in a full build of jQuery. [1] As you might guess, this results in a rather large & render-blocking JavaScript bundle. (A recent check shows the bundle size to be 204KB.)
3. **Code Quality** — Optimizely code must be written and reviewed within a textarea on a web page, making for a less than optimal reading experience. Thorough testing is also difficult, as the code generally must be copied to a local instance of bedrock to fully understand the impact (e.g. different screen sizes).
4. **Cost** — Optimizely is a paid service. We're not crying poor here, but perhaps that money can be better spent [elsewhere](https://www.mozilla.org/internet-health/).

Have we cancelled our Optimizely account? No. Not all of our experiments are of the simple "redirect a visitor" variety. Optimizely still provides value for us, but we try to make sure all experiments that *can* use Traffic Cop do.

Can Traffic Cop do everything Optimizely can do? Not even close, but we don’t need it to. As stated, we essentially only need Traffic Cop to redirect visitors.

So, how does Traffic Cop work? A visitors hits a URL running an experiment, e.g. `https://www.mozilla.org/en-US/internet-health/`. Traffic Cop picks a random number, and, if that random number falls within the range specified by a variation, redirects the visitor to that variation, e.g. `https://www.mozilla.org/en-US/internet-health/?v=2`.

Traffic Cop assumes all variations are loaded through a querystring parameter appended to the original URL. This keeps things simple, as no new URLs need be added to bedrock. We simply check for the querystring parameter (either in [the view]() or in [the template](https://github.com/mozilla/bedrock/blob/01296a1c5aefc8b8c3f36310c694dced4a7d9b01/bedrock/firefox/templates/firefox/new/scene1.html#L135)) and change content accordingly.


1. validates the supplied configuration
2. ensures the visitor isn’t *already* viewing a variation
3. checks to see if the visitor was previously chosen for a variation (and redirects them there if true)
4. chooses a number between 1-100
5. picks a variation based on that number
6. if the chosen number maps to a variation, the user is redirected to that variation. if not, the user is chosen to be in the “no variation” group, and will stay in that group until they close the current tab/window.

Any content changes are handled by the template rendered by the resulting URL. (We’d much rather create custom templates & assets rather than write jQuery in a textbox to be injected by a third-party script.)

Implementing Traffic Cop requires two* other JavaScript files: one to configure the experiment, and [MDN’s simple cookie framework](https://developer.mozilla.org/en-US/docs/Web/API/Document/cookie/Simple_document.cookie_framework). The configuration file is fairly straightforward. Simply instantiate a new Traffic Cop with your experiment configuration, and then initialize it.

```
var lou = new Mozilla.TrafficCop({
  // this id should be unique among your experiments, as it is used to check
  // for a previously chosen variation
  id: ‘experiment-home-page-cta’,
  // the keys in this object map to the querystring parameter appended to the current URL
  // the values here represent the percent chance of the variation being chosen
  variations: {
    ‘v=1’: 20,
    ‘v=2’: 20
  }
});

lou.init();
```

So, in summary, Traffic Cop allows us to write code in a text editor, review in a pull request, and avoid rather heavy third party JavaScrit code injection. For free.

* Okay, yes, you looked at [the source](https://github.com/mozilla/bedrock/blob/master/media/js/base/mozilla-traffic-cop.js#L76) and saw Traffic Cop looking for a globally scoped function by the name of `_dntEnabled`. Guilty. However, Traffic Cop carries on just fine if `_dntEnabled` doesn’t exist, so just simmer down. As you’ve probably already guessed, [`_dntEnabled`](https://github.com/mozilla/bedrock/blob/master/media/js/base/dnt-helper.js) is a function that checks the doNotTrack status of the visitor’s browser. Be a conscientious developer and respect this setting for your visitors as well.

