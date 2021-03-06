Overview
========

Hoxy is a web-hacking proxy for [node.js](http://nodejs.org/), intended for use by web developers. As far as your browser and the webservber are concerned, hoxy is nothing but a standalone proxy server. However, via a set of rules created by you, hoxy will alter or redirect almost any aspect of the traffic flowing through it. Hoxy was inspired as a way to complement debuggers like Firebug, which let you manipulate the client runtime but not the underlying HTTP conversation.

[Video: Quick Introduction](http://www.youtube.com/watch?v=2YLfBTrVgZU)

Starting Hoxy
---------------

Stand in the hoxy project dir and type:

    node hoxy.js

This will start hoxy on port 8080. (For a different port, use `--port=port`.) Next, configure your browser's proxy settings to point to hoxy.

If it doesn't already exist, upon startup hoxy will create a file in the `rules` dir called `rules.txt`. Open this file in your text editor and edit/add rules as needed. There's no need to restart hoxy each time you save the rules file.

Note: hoxy catches as many errors as possible in an effort to stay running. By default, error messages are suppressed. If you're writing rules or developing plugins (or developing hoxy itself), you should run in debug mode so you can see syntax or runtime errors as they occur:

    node hoxy.js --debug

Now hoxy will print out all errors to the console.

Using Hoxy With Another Proxy
-----------------------------------------------

WARNING: this feature is still experimental. Expect bugs.

Hoxy looks for the optional `HTTP_PROXY` environment variable and, if found, uses it.

    export HTTP_PROXY=proxy.myschool.edu:80
    node hoxy.js

How to Use Hoxy
---------------

If you have a good grasp of HTTP basics, hoxy itself isn't wildly complicated, and reading the architectural overview below should give you a basic idea of how it works. From there, the readme file in the `rules` dir should give you enough info to get on your way writing rules.

If you're comfortable writing JavaScript for node.js, you can also write plugins for hoxy. See the readme file in the `plugins` dir for more info.

System Requirements
--------------------

Hoxy requires [node.js](http://nodejs.org/) to run, version 0.3 or higher. (Anecdotal evidence suggests it *may* work on earlier versions, YMMV.) Any browser can be used that can be configured to use a proxy, and that can see your hoxy instance on the network.

Architectural Overview
======================

An HTTP conversation could be illustrated like this:

    HTTP TRANSACTION
    -----------------------
    server:      2
    ------------/-\--------
    client:    1   3
    -----------------------
    time -->

* Phase 1: Client prepares to make request.
* Hop: Client transmits request to server.
* Phase 2: Server processes request and prepares response.
* Hop: Server transmits response to client.
* Phase 3: Client processes response.

For our purposes, we can say that an HTTP proxy represents no additional processing phases. It just passes through the data transparently. A proxy just introduces a slight complication to the transport layer:

    USING A PROXY SERVER
    -----------------------
    server:       2
    -------------/-\-------
    proxy:      /   \
    -----------/-----\-----
    client:   1       3
    -----------------------

* Phase 1: Client prepares to make request.
* Hop: Client transmits request to server (through the proxy).
* Phase 2: Server processes request and prepares response.
* Hop: Server transmits response to client (through the proxy).
* Phase 3: Client processes response.

Hoxy differs from a normal HTTP proxy by adding two additional processing steps, one during the request phase, another during the response phase:

    USING HOXY
    -----------------------
    server:       3
    -------------/-\-------
    hoxy:       2   4
    -----------/-----\-----
    client:   1       5
    -----------------------

* Phase 1: Client prepares to make request.
* Hop: Client transmits to hoxy.
* Phase 2: Hoxy executes request-phase rules.
* Hop: Hoxy transmits (a potentially altered) request to server.
* Phase 3: Server processes request and prepares response.
* Hop: Server transmits to hoxy.
* Phase 4: Hoxy executes response-phase rules.
* Hop: Hoxy transmits (a potentially altered) response to client.
* Phase 5: Client processes response.

Request-phase rules (2) and response-phase rules (4) are written by you, and can alter any aspect of a request or response, in a way that's undetectable by either client or server. This includes (but isn't limited to) method, url, hostname, port, status, headers, cookies, params, and even the content body. For example, if you change the hostname, hoxy will send the request to a *different* server, unbeknownst to the client!

Furthermore, a rule will fire only when certain conditions are met. For example, you can say, IF the url ends with ".min.js", THEN run js-beautify against the content of the response body. Or, IF the hostname equals "www.example.com", THEN change it to "www-stage.example.com". Just as any aspect of a request or response can be altered, any aspect of a request or response can be used in a conditional.

Finally, while hoxy has a broad set of out-of-the-box capabilities, its plugin API allows developers to extend it in arbitrary ways. Plugins are written in JavaScript and invoked using the same conditional logic described above. For example, IF the content type is "text/html", THEN run the foo plugin.

A Twist in the Plot
-------------------

Plugins running in the request phase have the option to write the response themselves, which effectively preempts the hit to the server. In such cases, hoxy's behavior would look like this:

    ------------------------------
    server:  *blissful ignorance*
    ------------------------------
    hoxy:          2———3
    --------------/-----\---------
    client:      1       4
    ------------------------------

* Phase 1: Client prepares to make request.
* Hop: Client transmits to hoxy.
* Phase 2: Hoxy executes request-phase rules, triggering a plugin that populates the response.
* Hoxy notices that the response has already been written and skips the server hit.
* Phase 3: Hoxy executes response-phase rules.
* Hop: Hoxy transmits to client.
* Phase 4: Client processes response.

Fin
---

Hoxy opens the door for all kinds of testing, debugging and prototyping (and maybe some mischief) that might not otherwise be possible. Please use hoxy responsibly.
