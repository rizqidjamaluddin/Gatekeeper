Gatekeeper
==========

Gatekeeper is a flexible authorization library in PHP... With attitude. It's designed for complex systems with support for **custom policies** - Gatekeeper lets you mix and match between different models of permissions and access control.

This is an *authorization* package - use it alongside whatever authentication model, package or function you like. Authoriazation is about determining if a user is allowed to do this or that, *not* if the user is who they claim to be. This separation is intentional; whether you've got OAuth 2.0 sessions or good ol' passwords, Gatekeeper doesn't mind.

## Example

Vanilla PHP:

```php
$gatekeeper = new Gatekeeper;

// set policies; start with an ACL list in a JSON file
$gatekeeper->pushPolicy(
  new RoleBasedAccessControlListPolicy(new JsonRoleBasedAccessControlListStore('permissions/acl.json'))
);

// set a ban list policy using a custom class that implements the BanListStore interface
$gatekeeper->pushPolicy(new BanListPolicy(new YourCustomBanListStore()));

// set a custom policy for your own application
$gatekeeper->pushPolicy(new YourCustomApplicationPolicy());

// check permissions; do this right before executing a sensitive operation.
// will throw an exception if the user is not given access.
$gatekeeper->iAm($user)->mayI('create', 'post')->please();
```


## Configuring

Install using Composer.

Initialize Gatekeeper in PHP simply by doing `new Gatekeeper`. Ideally every point in your app that uses Gatekeeper should refer to the same instance, so you only need to do `iAm` once. The Laravel facade already does this by default.

Your application's user model should implement the GatekeeperUser interface. It currently has no required methods; it's just to tell Gatekeeper that this is a representation of a user.

Currently, Gatekeeper will prioritize denials; it goes through all policies and will allow access if any policies say "allow", but a single "deny" will cause Gatekeeper to refuse. This will be customizable in the future.

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

Welcome to the cool bit.

Gatekeeper recognizes **Policies**, which are classes that can tell Gatekeeper if a given user is granted or denied access. Policies _may_ also need a **Store**, which refers to where the permission information is actually kept; this model is optional, if you want to separate between the policy logic and policy rules. The provided policies all do this.

### In-the-box Policies

- `RoleBasedAccessControlListPolicy` - Accepts a RoleBasedAccessControlListStore.
- `AccessControlListPolicy` - Accepts an AccessControlListStore.
- `BanListPolicy` - Accepts a BanListStore instance.
- `SuperuserPolicy` - If the user is on the list, they have access to everything. Accepts a SuperuserListStore instance.

Utility policies:

- `OpenToAllPolicy` - Send ALLOW to all; this reverses the built-in policy that no response means denial.
- `DenyGuestsPolicy` - Send DENY to all guests.

### Custom Policies

This is the real beef of Gatekeeper - you can create your *own* policies to fit your application needs.

Policies implement GatekeeperPolicy interfaces, which just have 2 methods: `checkIfUserMay($user, $verb, $noun, $resource)` amd `checkIfGuestMay($verb, $noun, $resource)`. These methods should return Gatekeeper::ALLOW to permit access, Gatekeeper::DENY to restrict; anything else will just ignore this policy and move on to the next. If no explicit response is given, Gatekeeper will deny access.

Here's a custom policy that 

```php

```

### Custom Stores

Policies may use stores to separate between the policy logic and the actual policy data; for instance, you might want to store your role-power lists on a JSON file, or in some PHP class, or maybe via MySQL. Each policy has their own store interfaces because they need different things.

You're welcome to create your own stores - particularly for database-based stores, you probably want to customize the exact SQL, table names, column names and whatnot.

For instance, the SuperuserListStore interface has one method, called `isSuperuser($user)`, so one could make this based on the user:

```php
class YourAppSuperuserListStore implements SuperuserListStore
{
  public function isSuperuser(GatekeeperUser $user)
  {
    return (bool) $user->super;
  }
}
```

Or embedded right in a class:


Or one that involves the database in Laravel 4:

```php
class YourAppSuperuserListStore implements SuperuserListStore
{
  public function isSuperuser(GatekeeperUser $user)
  {
    return (bool) DB::table('superusers')->where('user_id, $user->id)->count() === 1;
  }
}
```


