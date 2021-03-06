meteor-smart-publish
=============

Rather smart publications for Meteor.

This package is developed as fast alternative of https://github.com/Diggsey/meteor-reactive-publish and
production-ready and working alternative of https://github.com/erundook/meteor-publish-with-relations, which
has several issues. I've decided that fixing all of them will make me to re-write all code and here is what I've done
instead.

_WARNING_: this version this lacks big corner tests and the code is relatively complex, so I kindly ask you
not to use this in production, because awful things may happen in case of bugs. I'll be happy if you help me
generate corner cases (like empty keys in JSON or `null` values) and other interesting tests, even if current
version successfully passes them.

Features and usage
==================

Demonstration
-------------

You can see this package in action at <a href="http://smart-publish-demo.meteor.com/">`smart-publish-demo.meteor.com`</a>, source
code is available in <a href="https://github.com/yeputons/meteor-smart-publish-demo-basic">yeputons/meteor-smart-publish-demo-basic</a>.

Installation
------------

Just run `mrt add smart-publish`.

Several cursors publication
---------------------------
Publication of several cursors from the same collection - union of these cursors will be published. This is done
by storing counters for each element - in how many cursors is it presented. So, each connection requires linear amount of memory:

```
Meteor.smartPublish('users', function(id) {
  check(id, String);
  return [
    Meteor.users.find(this.userId),
    Meteor.users.find(id, {fields: {username: 1}}),
    Items.find({author: this.userId})
  ];
});

Meteor.smartPublish('items', function(l, r) {
  check(l, Number);
  check(r, Number);
  return [
    Items.find({value: {$lt: l}}, {fields: {value: 1, a: 1, 'x.a': 1}}),
    Items.find({value: {$gt: r}}, {fields: {value: 1, b: 1, 'x.b': 1}}),
  ];
});
```

Please note that different cursors may not even return different subsets of collection, they may return subsets with non-empty intersection and
different fields - union of fields will be correctly published, just like if you subscribe to several publications, which publish same elements.
This may not work with array projections (`$` and `$elemMatch`), though - I would be happy if you write a usecase and a test for me (and I would
be very happy if you fix this).

Reactive joins
--------------

Each element may 'depend on' arbitrary elements from this or another collections (say, each Post may depend on
its author and last ten voters):

```
Posts = new Posts('posts_collection');
Avatars = new Avatars('avatars');
Meteor.smartPublish('posts', function(limit) {
  this.addDependency('posts_collection', 'authorId' /* you may specify array of fields here as well */, function(post) {
    return Meteor.users.find(post.authorId); // Please note that callback should return cursor, don't use findOne here
  })
  this.addDependency('posts_collection', 'voters', function(post) {
    return [ Meteor.users.find({_in: _.first(post.voters, 10)}) ];
  })
  this.addDependency('users', 'avatar', function(user) { // All dependencies are recursively pushed
    return Avatars.find(user.avatar);
  });
  return Posts.find({}, {limit: limit});
});
```

For each dependency, you should specify one or more fields that affect cursors that are returned by your callback (for example, empty array
or `_id` if you have reverse foreign keys). When any of these fields is updated, your callback is automatically re-run, new data is fetched
recursively, old data is dismissed. 'Diamond' joins are supported as well without any changes.

Known issues and limitations
============================
1. Not enough tests yet.

Running tests
=============
You can create stub application, add `smart-publish` package and then use `meteor test-packages` command:

1. `meteor create temp-app`
2. `cd temp-app`
3. `mkdir packages`
4. `cd packages`
5. `git clone https://github.com/yeputons/meteor-smart-publish.git smart-publish`
6. `cd ..`
7. `meteor test-packages smart-publish`
8. Navigate to <a href="http://localhost:3000">`http://localhost:3000`</a> to run tests and see results. Hot code push should work
