# Loading in profiles from various sources

Profiles can be loaded into the Firefox Profiler from many different sources.

### Online Storage

> `https://profiler.firefox.com/public/{TOKEN}`

Profiles can be stored in online data store. The token is used to retrieve it. This is where profiles go when clicking the "Share..." button. Here is an example bash script to programmatically upload profiles:

```bash
uploadprofile() {
  # decode_jwt_payload.py is in https://raw.githubusercontent.com/firefox-devtools/profiler-server/master/tools/decode_jwt_payload.py
  gzip -c "$1" | curl 'https://api.profiler.firefox.com/compressed-store' -X POST -H 'Accept: application/vnd.firefox-profiler+json;version=1.0' --data-binary @- | decode_jwt_payload.py | awk '{print "Hosted at: https://profiler.firefox.com/public/"$1}'
}

# Execute with the following command:
uploadprofile /path/to/profile.js
```

### URL

> `https://profiler.firefox.com/from-url/{URL}`

Profiles can also be loaded in from arbitrary URLs. In addition, you can then upload it to the online storage directly from The Firefox Profiler's interface. The profile needs to be served from https, and have the appropriate `Access-Control-Allow-Origin` header set to either `*` or `https://profiler.firefox.com`.

Here is a small utility function to turn a profile URL into a [profiler.firefox.com](https://profiler.firefox.com) URL to view it:

```js
function getProfilerUrl (profileUrl) {
  return "https://profiler.firefox.com/from-url/" + encodeURIComponent(profileUrl)};
}
```

Or in Python:

```python
import urllib

def get_profiler_url(url):
    return "https://profiler.firefox.com/from-url/" + urllib.quote(url, safe="")
```

Here is an example profile server written in Node.js:

```js
const https = require('https');
const fs = require('fs');
const path = require('path');
const querystring = require('querystring');

const PORT = 3000;

// If loading in a self-signed certificate, you must first manually add an exception
// to it in the browser, or else profiler.firefox.com will throw a NetworkError error.
const options = {
  key: fs.readFileSync(path.join(__dirname, 'server.key')),
  cert: fs.readFileSync(path.join(__dirname, 'server.crt')),
};

// The server must be "https"
const server = https.createServer(options, (request, response) => {
  // You must give access to profiler.firefox.com
  response.setHeader(
    'Access-Control-Allow-Origin',
    'https://profiler.firefox.com'
  );

  // You could also do * to allow anyone to load it:
  // response.setHeader('Access-Control-Allow-Origin', "*");

  // Write out the headers and file:
  response.writeHead(200, { 'Content-Type': 'application/json' });
  response.end(
    // Pull in a profile.json file.
    fs.readFileSync(path.join(__dirname, 'profile.json'))
  );
});

// Start listening on the port.
server.listen(PORT, (err) => {
  if (err) {
    return console.log('Error starting server', err);
  }
  // This is the URL of the profile we're serving.
  const profileUrl = `https://localhost:${PORT}/profile.json`;

  // This is the URL to profiler.firefox.com, it should look like the following when
  // properly encoded:
  // https://profiler.firefox.com/from-url/https%3A%2F%2Flocalhost%3A3000%2Fprofile.json
  const firefoxProfilerUrl =
    'https://profiler.firefox.com/from-url/' + querystring.escape(profileUrl);

  console.log('Serving the profile from: ' + profileUrl);
  console.log('View it on profiler.firefox.com: ' + firefoxProfilerUrl);
});
```

Note that if you have a copy of the project locally, you can add the profile path as a parameter to `yarn start` and a server serving this profile will be spawned for you. On Linux, the provided [launch-fp.sh](../bin/launch-fp.sh) script can do this for you as well, from any working directory.

### Directly from Firefox

> `https://profiler.firefox.com/from-browser/`

Firefox loads the profiles directly into the front-end through a WebChannel mechanism. This is done with the profile menu button, which can be enabled on the homepage of [profiler.firefox.com](https://profiler.firefox.com/)

### From a window.postMessage

> `https://profiler.firefox.com/from-post-message/`

A profile can be injected via another website and the postMessage API.

First wait for the page to be ready. This can be done by posting an `{ name: 'ready:request' }` message and waiting for the response of a similar `{ name: 'ready:response' }`.

```js
/**
 * Open a profile in https://profiler.firefox.com/
 */
function openProfile(profile) {
  const origin = 'https://profiler.firefox.com';
  const profilerURL = origin + '/from-post-message/' + params;
  const profilerWindow = window.open(profilerURL, '_blank');

  if (!profilerWindow) {
    console.error('Failed to open the new window.');
    return;
  }

  // Wait for the profiler page to respond that it is ready.
  let isReady = false;

  /**
   * @param {MessageEvent} event
   */
  const listener = ({ data }) => {
    if (data?.name === 'ready:response') {
      console.log('The profiler is ready. Injecting the profile.');
      isReady = true;
      const message = {
        name: 'inject-profile',
        profile,
      };
      profilerWindow.postMessage(message, origin);
      window.removeEventListener('message', listener);
    }
  };

  window.addEventListener('message', listener);
  while (!isReady) {
    await new Promise((resolve) => setTimeout(resolve, 100));
    profilerWindow.postMessage({ name: 'ready:request' }, origin);
  }

  window.removeEventListener('message', listener);
}
```

Then you can inject the profile via a postMessage.

```js
await waitForProfilerReady();
window.postMessage({ name: 'inject-profile', profile }, '*');
```

### File

> `https://profiler.firefox.com/from-file/`

When you're on [the home page](https://profiler.firefox.com) files can be loaded by either dragging over the profiler.firefox.com client, or using the file upload input.
