Log in
======

Background
----------
This task required us to register an account on http://aboudingcougar.k17.org and then log into it.

Tools used
----------
* Web browser with debugging capabilities (e.g. Chrome or Firefox)

Process
-------
To login, of course, one must have an account. Attempting to register reveals a waiting page with your user ID. `js/magic.js` also conveniently reveals a call to `mod_tasks.php` with parameters `action=enable_user` and `user_id`. Calling this page enabled the user using a URL structured:
````
http://aboundingcougar.k17.org/mod_tasks.php?action=enable_user&user_id=<your-user-id-here>
````

Upon logging in, the key is visible: **0abe408418d729bf8beab4bbfa9d31bb2606791cf7b3d**.
