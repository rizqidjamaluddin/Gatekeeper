Gatekeeper
==========

Gatekeeper is a flexible authorization library in PHP... With attitude. It's designed for complex systems with support for custom policies - Gatekeeper lets you mix and match between different models of permissions and access control.

This is an *authorization* package - use it alongside whatever authentication model, package or function you like. Authoriazation is about determining if a user is allowed to do this or that, *not* if the user is who they claim to be. This separation is intentional; whether you've got OAuth 2.0 sessions or good ol' passwords, Gatekeeper coesn't mind.

## Configuring

Install using Composer.

Gatekeeper recognizes *Policies*, which are classes that can tell Gatekeeper if a given user is granted or denied access.



## Checking Permissions

First, Gatekeeper needs to know who the user is. For instance, in Laravel 4:

```php
$gatekeeper = new Gatekeeper;
$gatekeeper->iAm(Auth::user());
```

Now it's just a matter of asking Gatekeeper nicely. These methods are chainable, so you can do this:

```php
$gatekeeper->iAm(Auth::user())->mayI('create', 'post')->please();
```

(Blame PHP for not letting us use a question mark as a function name.)

This goes through all the policies you have in place, and checks if the current `$user` is allowed to `$verb` (create) a `$noun` (post). If the user is granted access, the code continues normally; otherwise, a GatekeeperException is thrown.

If you need to debug a Gatekeeper check, you can either use `getReport` or take it out of the exception:

```
try {
  $gatekeeper->iAm($user)->andMayI('create', 'post')->please();
} catch (GatekeeperException $e) {
  echo($e->getReport());
  exit;
}
echo $gatekeeper->getReport();
```

`andMayI` is an alias of `mayI`.

You can also use `canI` instead of `mayI` to get a boolean true/false instead of throwing an exception, but that makes Gatekeeper sad.

## Checking permissions upon a resource


