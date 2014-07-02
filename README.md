Gatekeeper
==========

Gatekeeper is a flexible authorization library in PHP... With attitude. It's designed for complex systems with support for custom policies - Gatekeeper lets you mix and match between different models of permissions and access control.

This is an *authorization* package - use it alongside whatever authentication model, package or function you like. Authoriazation is about determining if a user is allowed to do this or that, *not* if the user is who they claim to be. This separation is intentional; whether you've got OAuth 2.0 sessions or good ol' passwords, Gatekeeper doesn't mind.

## Example

Vanilla PHP:

```php
$gatekeeper = new Gatekeeper;

// set policies; start with an ACL list in a JSON file
$gatekeeper->pushPolicy(new AccessControlListPolicy(new JsonPermissionStore('permissions/acl.json')));

// set a ban list policy using a custom class that implements the BanListStore interface
$gatekeeper->pushPolicy(new BanListPolicy(new CustomBanListStore()));

// check permissions; do this right before executing a sensitive operation.
// will throw an exception if the user is not given access.
$gatekeeper->iAm($user)->mayI('create', 'post')->please();
```


## Configuring

Install using Composer.

Initialize Gatekeeper in PHP simply by doing `new Gatekeeper`. Ideally every point in your app that uses Gatekeeper should refer to the same instance, so you only need to do `iAm` once. The Laravel facade already does this by default.

Your application's user model should implement the GatekeeperUser interface. It currently has no required methods; it's just to tell Gatekeeper that this is a representation of a user.

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

```php
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

### Implicit iAm

If you're tired of telling Gatekeeper `iAm` all the time, you can tell Gatekeeper where to get the user identity by default:

```php
$gatekeeper->setImplicitIdentity(function(){
  return Auth::user(); // laravel's auth facade
});
```

If the return value is an instance of GatekeeperUser, it will use this user for checks. If a user has already been explicitly set with `iAm`, implicit identity checks will *not* happen.

## Checking Resource Permissions

The same syntax also applies for protecting a **Resource**. Resources are models, entities, or otherwise _some class_ that you want to protect with Gatekeeper - such as a message, social media post, or photo album. The resource acts as the $noun:

```php
$gatekeeper->mayI('update', $post)->please();
```

And otherwise acts the same way.

Protected resources should implement the GatekeeperProtectedResource interface. This interface requires two methods: `getResourceName`, which should return a string for matching in permission policies, and `checkOwnership(GatekeeperUser $user)`, which should return true if $user has ownership upon this resource. This way, a resource can determine for itself if the given user "owns" itself, so you can have a single resource with multiple owners.

## Configuring Permission Policies

Gatekeeper recognizes *Policies*, which are classes that can tell Gatekeeper if a given user is granted or denied access.

[TBA]
