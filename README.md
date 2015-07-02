rancher-dns
===========

[![Build Status](http://drone.rancher.io/api/badge/github.com/rancherio/rancher-dns/status.svg?branch=master)](http://drone.rancher.io/github.com/rancherio/rancher-dns)


A simple DNS server that returns different answers depending on the IP address of the client making the request.

# Usage
```bash
  rancher-dns [--debug] [--listen host:port] [--ttl num] [--log path] [--pid-file path]--answers /path/to/answers.json
```

# Compile
```
  godep go build
```

## CLI Options

Option      | Default        | Description
------------|----------------|------------
`--debug`   | *off*          | If present, more debug info is logged
`--listen`  | 0.0.0.0:53     | IP address and port to listen on (TCP &amp; UDP)
`--answers` | ./answers.json | Path to a JSON file with client-specific answers
`--ttl`     | 600            | Default TTL for local responses that are returned
`--log`     | *none*         | Output log info to a file path instead of stdout
`--pid-file`| *none*         | Write the server PID to a file path on startup

## JSON Answers File
```javascript
{
  "10.1.2.2": {
    // DNS servers to recurse to when answers are not found locally
    "recurse": ["8.8.4.4:53", "8.8.8.8"],

    // A records
    "a": {
      // FQDN => { answer: array of IPs, ttl: TTL for this specific answer }
      // Note: Key must be fully qualified (ending in dot)
      "mysql.": {"answer": ["10.1.2.3"], "ttl": 42},
      "web.": {"answer": ["10.1.2.4","10.1.2.5","10.1.2.6"]}
    },

    // CNAME records
    "cname": {
      // FQDN => { answer: a single FQDN, ttl: TTL for this specific answer }
      // Note: Key & Answer must be fully qualified (ending in dot)
      "www.": {"answer": "web.", "ttl": 42}
    }
  },

  "192.168.0.2": {
    "recurse": ["8.8.4.4:53","8.8.8.8"],
    "a": {
      "mysql.": {"answer": ["192.168.0.3"]},
      "web.": {"answer": ["192.168.0.4","192.168.0.5","192.168.0.6"]}
    },
    "cname": {
      "www.": {"answer": "web."}
    }
  },

  // "default" is a special key that will be checked if no answer is found in a client IP-specific entry
  "default": {
    "recurse": ["8.8.8.8"],
    "a": {
      "foo.": {"answer": ["1.2.3.4"]}
    },
    "cname": {
      "website.": "www.",
      "external.": "rancher.com."
    }
  }
}
```

## Answering queries
A query is answered by returning the first match of:
  - An entry in the answers map for the client's IP.
  - An entry in the answers map in the `"default"` key.
  - If there is a `"recurse"` key for the client's IP, perform recursive lookup on each of those servers (in order).
  - If there is a `"recurse"` key for the `"default"`, perform recursive lookup on each of those servers (in order).
  - Do not pass go, do not collect $200.  Return `SERVFAIL`.

If the result is a CNAME record, then the process is repeated recursively until an A record is found.  If the chain does not end in an A record, is more than 10 levels deep, or is circular, an error is returned.

## Limitations
  - Only A and CNAME records are currently supported.

License
=======
Copyright (c) 2015 [Rancher Labs, Inc.](http://rancher.com)

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

[http://www.apache.org/licenses/LICENSE-2.0](http://www.apache.org/licenses/LICENSE-2.0)

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
