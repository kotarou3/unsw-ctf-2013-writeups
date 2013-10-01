D-d-d-d-do you have full access?
================================

Background
----------
This task required us to find the password to http://aboundingcougar.k17.org's database.

Tools used
----------
* Web browser with debugging capabilities (e.g. Chrome or Firefox)

Process
-------
Now that we were a moderator, we had some extra pages to play with, namely, the `manage.php` page. This page had some links to pages such as `manage.php?view=mod_only_list_users` and `manage.php?view=mod_only_messages`. We tried changing the value given to view, and we were greeted with an error.
````
Warning: require(aoeu.php): failed to open stream: No such file or directory in /var/www/sites/webchallenge/aboundingcougar/htdocs/manage.php on line 25
````
This told us that the value given to view was being passed into a require function unsanitised in php, which inserts php from another file into the current one. However, this didn’t help much because the server was not configured to allow requires from external addresses to run our own code.

Some time later, we noticed that on the third link on `manage.php` also had `&debug=1` appended to it, and tried to put that in conjunction with the other `manage.php` links, and it turns out that appending that caused the source code to be printed out.

So, by following a stream of requires and using path traversal, we went from `index.php` to `../include/general.inc.php` to `../include/config.inc.php` to `../include/db.inc.php`, which had the database password of **uTSkB3F2eV3Edaydrrmt6WhS**.
