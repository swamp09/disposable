# Disposable

_Decorators on top of your ORM layer._

## Introduction

Disposable is the missing API of ActiveRecord*. Its mission is to

* Maintain a manipulatable object graph that is a copy/map of a persistent structure.
* Prevent any write to the persistence layer until you say `sync`.
* Help designing your domain layer without being restricted to database layouts.
* Provide additional behavior like [change tracking](#change-tracking), [imperative callbacks](#imperative-callbacks) and [collection semantics](#collection-semantics).


Disposable gives you "_Twins_": non-persistent domain objects. That is reflected in the name of the gem. They can read from and write values to a persistent object and abstract the persistence layer until data is synced to the model.

## API

The public twin API is unbelievably simple.

1. `Twin::new` creates and populates the twin.
1. `Twin#"reader"` returns the value or nested twin of the property.
1. `Twin#"writer"=(v)` writes the value to the twin, not the model.
1. `Twin#sync` writes all values to the model.
1. `Twin#save` writes all values to the model and calls `save` on configured models.


## Twin

Twins are only # FIXME % slower than AR alone.

Twins implement light-weight decorators objects with a unified interface. They map objects, hashes, and compositions of objects, along with optional hashes to inject additional options.

Every twin is based on a defined schema.

```ruby
class AlbumTwin < Disposable::Twin
  property :title
  property :playable?, virtual: true # context-sensitive, e.g. current_user dependent.

  collection :songs do
    property :name
    property :index
  end

  property :artist do
    property :full_name
  end
end
```

## Constructor

Twins get populated from the decorated models.

```ruby
Song   = Struct.new(:name, :index)
Artist = Struct.new(:full_name)
Album  = Struct.new(:title, :songs, :artist)
```

You need to pass model and the facultative options to the twin constructor.

```ruby
album = Album.new("Nice Try")
twin  = AlbumTwin.new(album, playable?: current_user.can?(:play))
```

## Readers

This will create a composition object of the actual model and the hash.

```ruby
twin.title     #=> "Nice Try"
twin.playable? #=> true
```

You can also override `property` values in the constructor:

```ruby
twin = AlbumTwin.new(album, title: "Plasticash")
twin.title #=> "Plasticash"
```

## Writers

Writers change values on the twin and are _not_ propagated to the model.

```ruby
twin.title = "Skamobile"
twin.title  #=> "Skamobile"
album.title #=> "Nice Try"
```

Writers on nested twins will "twin" the value.

```ruby
twin.songs #=> []
twin.songs << Song.new("Adondo", 1)
twin.songs  #=> [<Twin::Song name="Adondo" index=1 model=<Song ..>>]
album.songs #=> []
```

The added twin is _not_ passed to the model. Note that the nested song is a twin, not the model itself.

## Sync

Given the above state change on the twin, here is what happens after calling `#sync`.

```ruby
album.title  #=> "Nice Try"
album.songs #=> []

twin.sync

album.title  #=> "Skamobile"
album.songs #=> [<Song name="Adondo" index=1>]
```

`#sync` writes all configured attributes back to the models using public setters as `album.name=` or `album.songs=`. This is recursive and will sync the entire object graph.

Note that `sync` might already trigger saving the model as persistence layers like ActiveRecord can't deal with `collection= []` and instantly persist that.

You may implement your syncing manually by passing a block to `sync`.

```ruby
twin.sync do |hash|
  hash #=> {
  #  "title"     => "Skamobile",
  #  "playable?" => true,
  #  "songs"     => [{"name"=>"Adondo"...}..]
  # }
end
```

Invoking `sync` with block will _not_ write anything to the models.

Needs to be included explicitly (`Sync`).

## Save

Calling `#save` will do `sync` plus calling `save` on all nested models. This implies that the models need to implement `#save`.

```ruby
twin.save
#=> album.save
#=>      .songs[0].save

```

Needs to be included explicitly (`Save`).

## Nested Twin

Nested objects can be declared with an inline twin.

```ruby
property :artist do
  property :full_name
end
```

The setter will automatically "twin" the model.

```ruby
twin.artist = Artist.new
twin.artist #=> <Twin::Artist model=<Artist ..>>
```

You can also specify nested objects with an explicit class.

```ruby
property :artist, twin: TwinArtist
```

## Collections

Collections can be defined analogue to `property`. The exposed API is the `Array` API.

* `twin.songs = [..]` will override the existing value and "twin" every item.
* `twin.songs << Song.new` will add and twin.
* `twin.insert(0, Song.new)` will insert at the specified position and twin.

You can also delete, replace and move items.

* `twin.songs.delete( twin.songs[0] )`

None of these operations are propagated to the model.

## Collection Semantics

In addition to the standard `Array` API the collection adds a handful of additional semantics.

* `songs=`, `songs<<` and `songs.insert` track twin via `#added`.
* `songs.delete` tracks via `#deleted`.
* `twin.destroy( twin.songs[0] )` deletes the twin and marks it for destruction in `#to_destroy`.
* `twin.songs.save` will call `destroy` on all models marked for destruction in `to_destroy`. Tracks destruction via `#destroyed`.

Again, the model is left alone until you call `sync` or `save`.

## Change Tracking

The `Changed` module will allow tracking of state changes in all properties, even nested structures.

```ruby
class AlbumTwin < Disposable::Twin
  feature Changed
```

Now, consider the following operations.

```ruby
twin.name = "Skamobile"
twin.songs << Song.new("Skate", 2) # this adds second song.
```

This results in the following tracking results.

```ruby
twin.changed?             #=> true
twin.changed?(:name)      #=> true
twin.changed?(:playable?) #=> false
twin.songs.changed?       #=> true
twin.songs[0].changed?    #=> false
twin.songs[1].changed?    #=> true
```

Assignments from the constructor are _not_ tracked as changes.

```ruby
twin = AlbumTwin.new(album)
twin.changed? #=> false
```

## Persistance Tracking

The `Persisted` module will track the `persisted?` field of the model, implying that your model exposes this field.

```ruby
twin.persisted? #=> false
twin.save
twin.persisted? #=> true
```

The `persisted?` field is a copy of the model's persisted? flag.

You can also use `created?` to find out whether a twin's model was already persisted or just got created in this session.

```ruby
twin = AlbumTwin.new(Album.create) # assuming we were using ActiveRecord.
twin.created? #=> false
twin.save
twin.created? #=> false
```

This will only return true when the `persisted?` field has flipped.

## Renaming

The `Expose` module allows renaming properties.

```ruby
class AlbumTwin < Disposable::Twin
  feature Expose

  property :song_title, from: :title
```

The public accessor is now `song_title` whereas the model's accessor needs to be `title`.

```ruby
album = OpenStruct.new(title: "Run For Cover")
AlbumTwin.new(album).song_title #=> "Run For Cover"
```

## Composition

Compositions of objects can be mapped, too.

```ruby
class AlbumTwin < Disposable::Twin
  feature Composition

  property :id,    on: :album
  property :title, on: :album
  property :songs, on: :cd
  property :cd_id, on: :cd, from: :id
```

When initializing a composition, you have to pass a hash that contains the composees.

```ruby
AlbumTwin.new(album: album, cd: CD.find(1))
```

Note that renaming works here, too.

## Struct

[FIXME: Coming soon!]

Twins can also map hash properties, e.g. from a deeply nested serialized JSON column.

```ruby
album.permissions #=> {admin: {read: true, write: true}, user: {destroy: false}}
```

Map that using the `:struct` option.

```ruby
class AlbumTwin < Disposable::Twin
  feature Struct

  property :permissions, struct: true do
    property :admin, struct: true, do
      property :read
      property :write
    end

    property :user # you don't have to use :struct everywhere!
  end
```

You get fully object-oriented access to your properties.

```ruby
twin.permissions.admin.read #=> true
```

Note that you do not have to use `:struct` everywhere.

```ruby
twin.permissions.user #=> {destroy: false}
```

Of course, this works for writing, too.

```ruby
twin.permissions.admin.read = :MAYBE
```

After `sync`ing, you will find a hash in the model.

```ruby
album.permissions #=> {admin: {read: :MAYBE, write: true}, user: {destroy: false}}
```

## With Representers

they indirect data, the twin's attributes get assigned without writing to the persistence layer, yet.

## With Contracts



## Imperative Callbacks

Note: [Chapter 8 of the Trailblazer](http://leanpub.com/trailblazer) book is dedicated to callbacks and discusses them in great detail.

Callbacks use the fact that twins track state changes. This allows to execute callbacks on certain conditions.

```ruby
Callback.new(twin).on_create { |twin| .. }
Callback.new(twin.songs).on_add { |twin| .. }
Callback.new(twin.songs).on_add { |twin| .. }
```

It works as follows.

1. Twins track state changes, like _"item added to collection (`on_add`)"_ or _"property changed (`on_change`)"_.
2. You decide when to invoke one or a group of callbacks. This is why there's no `before_save` and the like anymore.
3. You also decide _what_ events to consider by calling the respective events only, like `on_add`.
4. The `Callback` will now find out which properties of the twin are affected and exectue your passed code for each of them.

This is called _Imperative Callback_ and the opposite of what you've learned from Rails.

By inversing the control, we don't need `before_` or `after_`. This is in your hands now and depends on where you invoke your callbacks.

## Events

The following events are available in `Callback`.

Don't confuse that with event triggering, though! Callbacks are passive, calling an event method means the callback will look for twins that have tracked the respective event (e.g. an twin has `change`d).

* `on_update`: Invoked when the underlying model was persisted, yet, at twin initialization and attributes have changed since then.
* `on_add`: For every twin that has been added to a collection.
* `on_add(:create)`: For every twin that has been added to a collection and got persisted. This will only pick up collection items after `sync` or `save`.

* `on_delete`: For every item that has been deleted from a collection.
* `on_destroy`: For every item that has been removed from a collection and physically destroyed.

* `on_change`: For every item that has changed attributes. When `persisted?` has flippend, this will be triggered, too.
* `on_change(:email)`: When the scalar field changed.


## Callback Groups

`Callback::Group` simplifies grouping callbacks and allows nesting.

```ruby
class AfterSave < Disposable::Callback::Group
  on_change :expire_cache!

  collection :songs do
    on_add :notify_album!
    on_add :reset_song!
  end

  on_update :rehash_name!, property: :title

  property :artist do
    on_change :sing!
  end
end
```

Calling that group on a twin will invoke all callbacks that apply, in the order they were added.

```ruby
AfterSave.new(twin).(context: self)
```

Methods like `:sing!` will be invoked on the `:context` object. Likewise, nested properties will be retrieved by simply calling the getter on the twin, like `twin.songs`.

Again, only the events that match will be invoked. If the top level twin hasn't changed, `expire_cache!` won't be invoked. This works by simply using `Callback` under the hood.

## Callback Inheritance

You can inherit groups, add and remove callbacks.

```ruby
class EnhancedAfterSave < AfterSave
  on_change :redo!

  collection :songs do
    on_add :rewind!
  end

  remove! :on_change, :expire_cache!
end
```

The callbacks will be _appended_ to the existing chain.

Instead of appending, you may also refine existing callbacks.

```ruby
class EnhancedAfterSave < AfterSave
  collection :songs, inherit: true do
    on_delete :rewind!
  end
end
```

This will add the `rewind!` callback to the `songs` property, resulting in the following chain.

```ruby
collection :songs do
    on_add    :notify_album!
    on_add    :reset_song!
    on_delete :rewind!
  end
```

## Builders

## Overriding Accessors

super

## Used In

* [Reform](https://github.com/apotonick/reform) forms are based on twins and add a little bit of form decoration on top. Every nested form is a twin.
* [Trailblazer](https://github.com/apotonick/trailblazer) uses twins as decorators and callbacks in operations to structure business logic.