Become a moderator
==================

Background
----------
This task required us to become a moderator on http://aboundingcougar.k17.org.

Tools used
----------
* Web browser with debugging capabilities (e.g. Chrome or Firefox)

Process
-------
The most obvious thing apparent beginning this task was the call to `make_administrator` in `mod_tasks.php`. Unfortunately, calling it regardless of being a guest or registered user simply resulted in redirection to the login page. The intermediate task, then, was to trick the moderator into making this call.

Entering simple HTML into every form field available on the website revealed that the message preview was not filtered, a place ripe to plug in some JavaScript. The website also sanitised any lowercase `<script>`s; writing `<SCRIPT>` did not trigger any sanitisation, allowing code to be executed. Unfortunately this didn't work.

We were going nowhere, until the hint was given that the moderator used NoScript which would explain our failed XSS attempts. After that, we then tried doing CSRF using an `<iframe>` tag pointing to:
````
http://aboundingcougar.k17.org/mod_tasks.php?action=make_administrator&user_id=<your-user-id-here>
````
which was sent to the moderator as a message. Now, when the moderator views their messages, our iframe would load and execute `make_administrator` on our user id in their context.

The key was found on the page `manage.php` that was now accessible: **ec96a0f01efc61b76c7aeaf3072260d8787c985f95d3357d5042065425c5293e**
