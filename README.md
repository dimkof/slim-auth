# Slim Auth [![Build Status](https://travis-ci.org/jeremykendall/slim-auth.png?branch=master)](https://travis-ci.org/jeremykendall/slim-auth) [![Coverage Status](https://coveralls.io/repos/jeremykendall/slim-auth/badge.png?branch=master)](https://coveralls.io/r/jeremykendall/slim-auth?branch=master) [![Dependencies Status](https://depending.in/jeremykendall/slim-auth.png)](http://depending.in/jeremykendall/slim-auth)

Slim Auth is an authorization and authentication library for the [Slim Framework][1].
Authentication is accomplished by using the Zend Framework [Authentication][2] 
component, and authorization by using the Zend Framework [Acl][3] component.

## DOCUMENTATION INCOMPLETE

This lib is usable, but is beta software, and this documentation is incomplete.
If you're familiar with Zend Auth and Zend ACL, you can probably work it out
just fine.  Otherwise, you might want to wait for the docs to be completed.

Caveat emptor and all that. 

## Installation

Install composer in your project:

```
curl -s https://getcomposer.org/installer | php
```

Create a composer.json file in your project root:

```
{
    "require": {
        "jeremykendall/slim-auth": "*"
    }
}
```

(*Please check Packagist for the [most recent version of Slim Auth][6]*)

Install via composer:

```
php composer.phar install
```

Add this line to your application’s index.php file:

```
<?php
require 'vendor/autoload.php';
```

## Preparing Your App For Slim Auth

### Database

Your database should have a user table, and that table must have a `role`
column.  The contents of the `role` column should be a string and correspond to
the roles in your ACL. The user table name and all other column names are up to you.

Here's an example schema for a user table. If you don't already have a user
table, feel free to use this one:

```
CREATE TABLE IF NOT EXISTS [users] (
    [id] INTEGE NOT NULL PRIMARY KEY,
    [username] VARCHAR(50) NOT NULL,
    [role] VARCHAR(50) NOT NULL,
    [password] VARCHAR(255) NULL
);
```

### ACL

An Access Control List, or ACL, defines the set of rules that determines which group
of users have access to which routes within your Slim application. Below is an
example ACL suitable for an extremely simple app.  Please pay special attention
to the comments.

*Please refer to the [Zend ACL documentation][3] for complete details on using
their ACL component.*

```
namespace Example;

use Zend\Permissions\Acl\Acl as ZendAcl;

class Acl extends ZendAcl
{
    public function __construct()
    {
        // APPLICATION ROLES
        $this->addRole('guest');
        // member role "extends" guest, meaning the member role will get all of 
        // the guest role permissions by default
        $this->addRole('member', 'guest');
        $this->addRole('admin');

        // APPLICATION RESOURCES
        // Application resources == Slim route patterns
        $this->addResource('/');
        $this->addResource('/login');
        $this->addResource('/logout');
        $this->addResource('/member');
        $this->addResource('/admin');

        // APPLICATION PERMISSIONS
        // Now we allow or deny a role's access to resources. The third argument
        // is 'privilege'. We're using HTTP method for resources.
        $this->allow('guest', '/', 'GET');
        $this->allow('guest', '/login', array('GET', 'POST'));
        $this->allow('guest', '/logout', 'GET');

        $this->allow('member', '/member', 'GET');

        // This allows admin access to everything
        $this->allow('admin');
    }
}
```

#### The Guest Role

Please note the `guest` role. **You must use the name** `guest` **as the role
assigned to an unauthenticated user**. The other role names are yours to choose.

#### Acl "Privileges"

**IMPORTANT**: The third argument to `Acl::allow()`, 'privileges', is either a
string or an array, and should be an HTTP verb or HTTP verbs respectively. By
adding the third argument, you are restricting route access by HTTP method.  If
you do not provide an HTTP verb or verbs, you are allowing access to the
specified route via *all* HTTP methods. **Be extremely vigilant here.** You
wouldn't want to accidentally allow a 'guest' role access to an admin `DELETE`
route simply because it references a public resource.

## Configuring Slim Auth: Defaults

Now that you have a user database table with a `role` column and an ACL, you're
ready to configure Slim Auth and add it to your application.

First, add `use` statements for the PasswordValidator (from the 
[Password Validator][9] library), the PDO adapter, and the Slim Auth Bootstrap.

```
use JeremyKendall\Password\PasswordValidator;
use JeremyKendall\Slim\Auth\Adapter\Db\PdoAdapter;
use JeremyKendall\Slim\Auth\Bootstrap;
```

Next, create your Slim application with `cookies.encrypt` and
`cookies.secret_key` as a minimum configuration.

>*Default Slim Auth identity storage is session storage. You MUST set the
>following cookie encryption settings if you use the SessionCookie middleware,
>which this example does. Details on configuring different storage are available
>later in the documentation.*

```
$app = new \Slim\Slim(array(
    // Config requirements for default Slim Auth implementation
    'cookies.encrypt' => true,
    'cookies.secret_key' => 'CHANGE ME. SERIOUSLY, CHANGE ME RIGHT NOW.',
));
```

### Authentication Adapter

From the Zend Authentication documentation:

> `Zend\Authentication` adapters are used to authenticate against a particular
> type of authentication service, such as LDAP, RDBMS, or file-based storage.

Slim Auth provides an RDBMS authentication adapter for PDO. The constructor
accepts five required arguments:

* A `\PDO` instance
* The name of the user table
* The name of the identity, or username, column
* The name of the credential, or password, column
* An instance of `JeremyKendall\Password\PasswordValidator`

```
$db = new \PDO(<database connection info>);
$adapter = new PdoAdapter(
    $db, 
    <user table name>, 
    <identity column name>, 
    <credential column name>, 
    new PasswordValidator()
);
```

> **NOTE**: Please refer to the [Password Validator documentation][9] for more
> information on the proper use of the library. If you choose not to use the 
> Password Validator library, you will need to create your own authentication 
> adapter.

### Putting it all Together

Now it's time to instantiate your ACL and bootstrap Slim Auth.

```
$acl = new \Namespace\For\Your\Acl();
$authBootstrap = new Bootstrap($app, $adapter, $acl);
$authBootstrap->bootstrap();
```

Finally, and this is *crucial*, you *must* add Slim's [SessionCookie][7]
Middleware, and you must add it  *after* the Slim Auth `Boostrap::bootstrap()`
method has been called. 

> **NOTE**: This is only a requirement if you're using the default Session
> Storage *and* you opt to use the `SessionCookie` middleware.  It is possible to
> configure Slim Auth to use storage other than Slim's SessionCookie.

```
// Add the session cookie middleware *after* auth to ensure it's executed first
$app->add(new \Slim\Middleware\SessionCookie());
```

### Login Route

You'll need a login route, of course, and it's important that you name your
route `login` using Slim's [Route Names][4] feature. 

```
$app->map('/login', function() {})->via('GET', 'POST')->name('login');
```

This allows you to use whatever route pattern you like for your login route.
Slim Auth will redirect users to the correct route using Slim's `urlFor()`
[Route Helper][5].

Here's a sample login route:

```
// Login route MUST be named 'login'
$app->map('/login', function () use ($app) {
    $username = null;

    if ($app->request()->isPost()) {
        $username = $app->request->post('username');
        $password = $app->request->post('password');

        $result = $app->authenticator->authenticate($username, $password);

        if ($result->isValid()) {
            $app->redirect('/');
        } else {
            $messages = $result->getMessages();
            $app->flashNow('error', $messages[0]);
        }
    }

    $app->render('login.twig', array('username' => $username));
})->via('GET', 'POST')->name('login');
```

### Logout Route

As authentication stores the authenticated user's identity, logging out
consists of nothing more than clearing that identity. Clearing the identity is
handled by `Authenticator::logout`.

```
$app->get('/logout', function () use ($app) {
    $app->authenticator->logout();
    $app->redirect('/');
});
```

[1]: http://slimframework.com/
[2]: http://framework.zend.com/manual/2.2/en/modules/zend.authentication.intro.html
[3]: http://framework.zend.com/manual/2.2/en/modules/zend.permissions.acl.intro.html
[4]: http://docs.slimframework.com/#Route-Names
[5]: http://docs.slimframework.com/#Route-Helpers
[6]: https://packagist.org/packages/jeremykendall/slim-auth
[7]: http://docs.slimframework.com/#Cookie-Session-Store
[8]: https://github.com/ircmaxell/password_compat
[9]: https://github.com/jeremykendall/password-validator
