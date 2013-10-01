Hidden messages
===============

Background
----------
This task required us to find and decode a secret message on [http://aboudingcougar.k17.org].

Tools used
----------
* Web browser with debugging capabilities (e.g., Chrome or Firefox)

Process
-------
For this task, a look at the website’s source code revealed very clearly, a string of garbled text in a `<div>` with `id="secret"`:
````
gurSvefgXrlVf:5s1780o49o9o0rq5oq4on8n0q4sp49r2nr
````

With seemingly nothing else on the page, a look at `js/magic.js` showed clearly the functions:
````js
$('#secret').show(1500);
$('#secret').unbreakablecrypto();
````
Calling these functions through the browser’s developer console revealed the hidden message/key, which was: **5f1780b49b9b0ed5bd4ba8a0d4fc49e2ae**.
