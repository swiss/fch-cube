# Livingdocs Integration

A static HTML file (e.g. a visualization using the [HTML / javascript](#using-html--javascript) approach) can be embedded into a Livingdocs (CMS) page.
If the static HTML needs dynamic parameters from the CMS page's URL (e.g. `?committee=...`), these are passed into the iFrame via `postMessage`.

## Setup in Livingdocs

1. Add an **iFrame** element to the CMS page.
2. Set the **Sizing Mode** option to **Dynamic resizing**.
3. In the **URL Settings**, set the URL to the static HTML file.
4. Add a **FreeHtml** element to the CMS page (below the iFrame).
5. Add the following JavaScript snippet to the FreeHtml element.

```html
<script>
  let initInterval = null;
  let acknowledged = false;

  const iframeSrc =
    "https://www.assets-a.bk.admin.ch/livingdocs/apg/committee-statistic.html"; // adjust URL
  const language = "de"; // adjust language

  window.addEventListener("message", (event) => {
    if (event.data?.type === "ogd-init-success") {
      acknowledged = true;
      clearInterval(initInterval);
      initInterval = null;
    }
  });

  const params = new URLSearchParams(window.parent.location.search);
  let committee = params.get("committee"); // adjust parameter and session key
  if (committee) {
    sessionStorage.setItem("apg-committee", committee);
  } else {
    committee = sessionStorage.getItem("apg-committee");

    if (committee) {
      const url = new URL(parent.location.href);
      url.searchParams.set("committee", committee);
      parent.history.replaceState(parent.history.state, "", url);
    }
  }

  function appendParamToIframe() {
    const iframe = window.parent.document.querySelector(
      `iframe[src*="${iframeSrc}"]`,
    );
    if (!iframe?.contentWindow) {
      setTimeout(appendParamToIframe, 300);
      return;
    }

    if (initInterval) return;

    initInterval = setInterval(() => {
      if (!acknowledged) {
        iframe.contentWindow.postMessage(
          { type: "ogd-init", committee, language },
          "*",
        ); // adjust parameters
      }
    }, 500);
  }

  appendParamToIframe();
</script>
```

_FreeHtml snippet passing query parameters to the iFrame_

### How it works

- The FreeHtml element reads the parameter (here `committee`) from the parent page's query string.
- The value is stored in the `sessionStorage`. If the parameter is missing (e.g. after navigating back to the page), the stored value is used and written back to the parent's URL.
- The snippet looks up the iFrame by its `src` (retrying every 300 ms until it is rendered).
- It then sends an `ogd-init` message with the parameters to the iFrame every 500 ms, until the iFrame answers with `ogd-init-success`.

### Adjustments per page

- `iframeSrc`: must match the URL set in the iFrame's URL Settings.
- `language`: language of the CMS page.
- Parameter name (`committee`) and session key (`apg-committee`): adjust to the parameter(s) needed by the static HTML. Use a unique session key per use case.

## Receiving the Parameters in the Static HTML

The static HTML file has to listen for the `ogd-init` message and confirm it with `ogd-init-success`. Otherwise, the FreeHtml snippet keeps sending the message.

```javascript
setupIframeResizeMessaging(); // see "iFrame Resizing" below

window.addEventListener("message", async (event) => {
  if (event.data?.type !== "ogd-init") return;

  const { committee, language } = event.data;

  // confirm reception to stop the init interval
  event.source?.postMessage({ type: "ogd-init-success" }, "*");

  // load / render data using the received parameters
  await render(committee, language);

  // report new size after rendering
  postIframeSize();
});
```

_Example listener in the static HTML file_

## iFrame Resizing

With **Dynamic resizing**, the Livingdocs iFrame adjusts its size to the content. For this, the static HTML has to report its size to the parent page with an `iframe-resized` message (`height` and `width` in pixels).

Add the following snippet to the static HTML file.

```javascript
let lastPostedIframeHeight = null;
let lastPostedIframeWidth = null;

function postIframeSize(additionalHeight = 0) {
  const root = document.documentElement;
  const body = document.body;

  const height =
    Math.max(root.scrollHeight, root.offsetHeight, body ? body.scrollHeight : 0, body ? body.offsetHeight : 0) +
    additionalHeight;

  const width = Math.max(root.scrollWidth, root.offsetWidth, body ? body.scrollWidth : 0, body ? body.offsetWidth : 0);

  if (lastPostedIframeHeight === height && lastPostedIframeWidth === width) {
    return;
  }

  lastPostedIframeHeight = height;
  lastPostedIframeWidth = width;

  window.parent.postMessage(
    {
      type: "iframe-resized",
      height,
      width,
    },
    "*",
  );
}

function setupIframeResizeMessaging() {
  let requestAnimationFrameId = null;
  const schedulePost = () => {
    if (requestAnimationFrameId !== null) {
      return;
    }
    requestAnimationFrameId = window.requestAnimationFrame(() => {
      requestAnimationFrameId = null;
      postIframeSize();
    });
  };

  const resizeObserver = new ResizeObserver(() => {
    schedulePost();
  });

  resizeObserver.observe(document.documentElement);
  if (document.body) {
    resizeObserver.observe(document.body);
  }

  window.addEventListener("resize", schedulePost);
  window.addEventListener("load", schedulePost);

  if (document.fonts?.ready) {
    document.fonts.ready.then(schedulePost).catch(() => {});
  }

  schedulePost();
}
```

_iFrame resize snippet for the static HTML file_

### How it works

- `postIframeSize` measures the document's height and width (max of `scrollHeight`/`offsetHeight` resp. `scrollWidth`/`offsetWidth` of `<html>` and `<body>`) and sends them to the parent page.
  - The message is only sent if the size changed since the last message.
  - `additionalHeight` (optional) adds extra pixels to the height. Use it if the iFrame content gets cut off in the CMS page (e.g. `postIframeSize(20)`).
- `setupIframeResizeMessaging` sends the size automatically whenever the layout may change:
  - `ResizeObserver` on `<html>` and `<body>`
  - window `resize` and `load` events
  - web fonts loaded (`document.fonts.ready`)
  - Multiple triggers within the same frame are combined into one message (`requestAnimationFrame`).

### Usage

- Call `setupIframeResizeMessaging()` once on startup, **before** adding the `ogd-init` message listener (see [example above](#receiving-the-parameters-in-the-static-html)).
- Call `postIframeSize()` whenever data has been rendered, so that the new size is reported right away.
