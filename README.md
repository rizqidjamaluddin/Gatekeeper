Gatekeeper
==========

Gatekeeper is a flexible authorization library in PHP... With attitude. It's designed for complex systems with support for **custom policies** - Gatekeeper lets you mix and match between different models of permissions and access control.

This is an *authorization* package - use it alongside whatever authentication model, package or function you like. Authoriazation is about determining if a user is allowed to do this or that, *not* if the user is who they claim to be. This separation is intentional; whether you've got OAuth 2.0 sessions or good ol' passwords, Gatekeeper doesn't mind.

*Please understand that Gatekeeper has flexibility and extendibility in mind*. It may be a tad more difficult to get the hang of, compared to other packages and their shortcuts. That's deliberate; Gatekeeper makes no assumptions and doesn't restrict itself, so you'll need to tell it exactly what to do. We provide as little automagic as possible.

## Example

Vanilla PHP:

```php
$gatekeeper = new Gatekeeper;

// set policies; start with an ACL list in a JSON file
$gatekeeper->pushPolicy(
  new RoleBasedACLPolicy(new JsonRoleBasedACLStore('permissions/acl.json'))
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

If you don't use `iAm`, or you give it null, it assumes the user is a guest.

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

Protected resources should implement the `ProtectedResource` interface. This interface requires two methods: `getResourceName`, which should return a string for matching in permission policies, and `checkOwnership(GatekeeperUser $user)`, which should return true if $user has ownership upon this resource. This way, a resource can determine for itself if the given user "owns" itself, so you can have a single resource with multiple owners.

## Configuring Permission Policies

Welcome to the cool bit.

Gatekeeper recognizes **Policies**, which are classes that can tell Gatekeeper if a given user is granted or denied access. Policies _may_ also need a **Store**, which refers to where the permission information is actually kept; this model is optional, if you want to separate between the policy logic and policy rules. The provided policies all do this.

### In-the-box Policies

See the full reference at the bottom of this page for details.

- `RoleBasedACLPolicy` - Your standard role-based access control list. Intended as the "primary" form of access control, as most projects use this.
- `GroupBasedACLPolicy` - Similar to the above, except a user can be in multiple groups at once.
- `ResourceACLPolicy` - Each resource has an explicit list of who can do what to them. Best used alongside the others.
- `BanListPolicy` - Stop particular users from doing a particular verb on a particular noun or resource.
- `SuperuserPolicy` - If the user is on the list, they have access to everything.


Utility policies:

- `OpenToAllPolicy` - Send ALLOW to all; this reverses the built-in policy that no response means denial.
- `DenyGuestsPolicy` - Send DENY to all guests.
- `DenyEveryonePolicy` - What it says on the tin. Useful for temporary blocking everything.
- `FulfillAllPolicy` - Accepts an array of _other policies_, and will only return ALLOW if _all_ of the sub-policies do.
- `FulfillAnyPolicy` - Also accepts an array of policies, and returns ALLOW if _at least one_ sub-policy does.
- `RequiredPolicy` - Accepts another policy, and returns DENY if the contained policy does not return ALLOW.
 

A bit more advanced:

- `UserCriteriaPolicy` - Uses the [criteria pattern](http://en.wikipedia.org/wiki/Criteria_Pattern) applied on the user.
- `ResourceCriteriaPolicy` - Same as above, applied on the resource.
- `CriteriaPolicy` - Applies a criteria that accepts both the user and resource.

### Custom Policies

This is the real beef of Gatekeeper - you can create your *own* policies to fit your application needs.

Policies implement GatekeeperPolicy interfaces, which just have 2 methods: `checkIfUserMay($user, $verb, $noun, $resource)` amd `checkIfGuestMay($verb, $noun, $resource)`. These methods should return Gatekeeper::ALLOW to permit access, Gatekeeper::DENY to restrict; anything else will just ignore this policy and move on to the next. If no explicit response is given, Gatekeeper will deny access.

Here's a custom policy that lets users create posts only if they have over 100 reputation:

```php
class ReputationBasedPolicy implements GatekeeperPolicy {
  
  public function checkIfGuestMay($verb, $noun, $resource ='')
  {
    return; // this policy does not apply to guests
  }
  
  public function checkIfUserMay(GatekeeperUser $user, $verb, $noun, $resource ='')
  {
    if ($verb == 'create' && $noun == 'post' && $user->reputation > 100) {
      return Gatekeeper::ALLOW;
    } else{
      return;
    }
  }
  
}
```

Note that this policy does not **deny** on failure, only pass, because we assume users might be allowed to create posts based on another policy (e.g. superusers can always post). As long as we don't set another policy that grants this, we're fine.

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

```php
class YourAppSuperuserListStore implements SuperuserListStore
{

  protected $superusers = [
    'john@example.com', 'jane@example.com'
  ];

  public function isSuperuser(GatekeeperUser $user)
  {
    return in_array($user->email, $this->superusers);
  }
}
```

Or one that involves the database in Laravel 4:

```php
class YourAppSuperuserListStore implements SuperuserListStore
{
  public function isSuperuser(GatekeeperUser $user)
  {
    return (bool) DB::table('superusers')->where('user_id', $user->id)->count() === 1;
  }
}
```

### Doing Operations on Policies and Stores

So far we've talked about how to check if a user can do this or that, but we haven't talked much about manipulating these rules during runtime. For instance, if you have a BanListPolicy, you might want to be able to ban a user. These operations are specific to a policy, and are not a property of Gatekeeper, so you'll want to store separate instances of them:

```php
$banList = new BanListPolicy(new WriteableTextBanListStore);
$gatekeeper->pushPolicy($banlist);

// somewhere else...
$banList->ban($user, $verb, $noun);
```

Some stores, like the provided JSON-based stores, are read-only; you'd have to go into the file and edit it to change the list. Others, like the WriteableText version above, *are* writeable.


## Word of Warning

Please **remember to fully test your security policies in your application**. Use whatever flavor of testing you like - never trust any piece of code, by anyone, to be foolproof. It's very easy for security holes to emerge, not due to bad programming, but simple lapses in judgement. *Always* test your code thoroughly.

--


## Provided Policy & Store Reference


### CriteriaPolicy

Invokes a `GatekeeperCriteria`-implementing class that accepts both the user and the resource, plus the verb. This is pretty close to being a Policy in and of itself, except that it **passes on guests and non-resource $nouns**. Basically just a handy shortcut.

It's suitable for checks along the lines of "if the user has this and the resource has that, can we do $verb to it?"

Example; if the *user* and *resource* share the same city, the user is allowed to *vote* on it:

```php
class SameCityCriteria implements GatekeeperCriteria {
  public function isSatisfiedBy(GatekeeperUser $user, ProtectedResource $resource, $verb)
  {
    if ($user->city == $resource->city && $verb == "vote") {
      return Gatekeeper::ALLOW;
    }
  }
}

$gatekeeper->pushPolicy(new CriteriaPolicy(new SameCityCriteria));
$gatekeeper->mayI('vote', $restaurant)->please();
```

CriteriaPolicy accepts a second argument: a class or interface name that the resource should match in order for it to be processed. For instance, `$resource->city` in the example above isn't very well-written, because `city` isn't a guaranteed property of ProtectedResource. Instead, we may want to create a VenueResource interface that extends it, which in turn defines a `getCity()` method.

```php
$gatekeeper->pushPolicy(new CriteriaPolicy(new SameCityCriteria, 'YourApp\VenueResource'));
```

CriteriaPolicy is also handy for **resource-based permissions** in general. For instance, you may want users to view a gallery page as public or just themselves and friends:

```php
class MatchGalleryViewingPrivacyCriteria implements GatekeeperCriteria {
  public function isSatisfiedBy(GatekeeperUser $user, Gallery $resource, $verb)
  {
    if (
        (
          $resource->checkOwnership($user) ||
          ($resource->getPrivacyLevel() == "friend" && $resource->owner->isFriendsWith($user))
        ) && $verb == 'view'
      ) {
      return Gatekeeper::ALLOW;
    }
  }
}

$gatekeeper->pushPolicy(new CriteriaPolicy(new MatchGalleryViewingPrivacyCriteria, 'YourApp\Gallery'));
$gatekeeper->mayI('view', $gallery)->please();
```

### UserCriteriaPolicy

Very similar to the above, except it only checks the user. A tad less useful.

Example; if the user has a confirmed email address, they're allowed to post comments:

```php
class ValidatedEmailCriteria implements GatekeeperUserCriteria {
  public function isSatisfiedBy(GatekeeperUser $user, $verb)
  {
    if ($user->hasConfirmedEmail() && $verb == "comment") {
      return Gatekeeper::ALLOW;
    }
  }
}

$gatekeeper->pushPolicy(new UserCriteriaPolicy(new ValidatedEmailCriteria));
$gatekeeper->mayI('comment', $post)->please();
```

### ResourceCriteriaPolicy

Also similar to CriteriaPolicy, this time only checking the resource.

Example; if the board is public, anyone can post messages to it:

```php
class PublicCriteria implements GatekeeperResourceCriteria {
  public function isSatisfiedBy(ProtectedResource $resource, $verb)
  {
    if ($resource->isPublic() && $verb == "create_post") {
      return Gatekeeper::ALLOW;
    }
  }
}

$gatekeeper->pushPolicy(new ResourceCriteriaPolicy(new PublicCriteria));
$gatekeeper->mayI('create_post', $board)->please();
```

Like the CriteriaPolicy, it also accepts a second argument with a class or interface name, defining which resource to run on.






