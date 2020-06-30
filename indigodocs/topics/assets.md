---
id: assets
title: Assets & Asset Loading
---

## Asset Types

There are three types of assets that Indigo understands how to load and make available:

1. Images - which can be JPEG or PNG format (others may work but are untested) - max size of 4k i.e. 4096 x 4096.
2. Text - Any plain text format be it prose, yaml, json or xml. Indigo does not understand the text format, it just loads the text and makes it available.
3. Sound - Any browser supported audio format

When assets are loaded they are registered, and in the case of Sounds and Images, some amount of preparation is immediately done to ensure they're usable.

Text is different. Plain text isn't useful in of itself in the scene construction, and the only time you can access text assets is during Startup via the `AssetCollection`, this give you an opportuntity to read/decode the contents and build them into your model somewhere.

## Asset Loading

Asset loading happens in one or two phases depending on whether you need the assets before your game starts or want to load it later.

### Load ahead of game start

The simplest from of asset loading happens based on your initial game definition, for example:

```scala
// If you're using the `IndigoSandbox` entry point
val assets: Set[AssetType] =
  Set(
    AssetType.Text(AssetName("map"), AssetPath(baseUrl + "assets/map.txt")),
    AssetType.Image(AssetName("font"), AssetPath(baseUrl + "assets/font.png")),
    AssetType.Image(AssetName("snake"), AssetPath(baseUrl + "assets/snake.png")),
    AssetType.Audio(AssetName("intro"), AssetPath(baseUrl + "assets/intro.mp3")),
    AssetType.Audio(AssetName("point"), AssetPath(baseUrl + "assets/point.mp3")),
    AssetType.Audio(AssetName("lose"), AssetPath(baseUrl + "assets/lose.mp3"))
  )

// Or if you're using the IndigoDemo` or `IndigoGame` entry points
def boot(flags: Map[String, String]): BootResult[Data] = {
  BootResult(config, data)
    .withAssets(
      Set(
        AssetType.Text(AssetName("map"), AssetPath(baseUrl + "assets/map.txt")),
        AssetType.Image(AssetName("font"), AssetPath(baseUrl + "assets/font.png")),
        AssetType.Image(AssetName("snake"), AssetPath(baseUrl + "assets/snake.png")),
        AssetType.Audio(AssetName("intro"), AssetPath(baseUrl + "assets/intro.mp3")),
        AssetType.Audio(AssetName("point"), AssetPath(baseUrl + "assets/point.mp3")),
        AssetType.Audio(AssetName("lose"), AssetPath(baseUrl + "assets/lose.mp3"))
      )
    )
```

> The important thing to know here is that whichever entry point style you're using, all of those assets will be forced to load completely before your game will show anything on the screen _at all_.

For demos and tests or local development with no network latency, requiring all the assets to be primed and ready before your game starts is no big deal, even for substantial amounts of data. It is even advantageous since there is a cost to loading images like generating texture atlases.

### Dynamic asset loading

When you're dealing with large amounts of asset data, you may not be happy to leave your player staring at a blank screen while Indigo loads and prepares everything. You might prefer to show them a loading screen! (Sometimes called a pre-loader.)

The basic flow we want to achieve is:

1. Set up a scene that will be your loading screen.
1. Use the standard asset loading mechanism to load only the assets you need to be able to draw your loading screen.
1. Launch your loading screen.
1. Kick off a background load of the remaining assets.
1. As the assets arrive, update a progress bar / animation on the loading screen.
1. Once they all arrive and have been processed, proceed to the next scene of the game - perhaps a menu screen.

> Note that the dynamic asset loading approach can be used to add assets any time, not just  as the above flow suggest, and will make use of the browser cache to avoid re-delivery.

You can either use the basic inbuilt events to load your assets, and manage the process yourself, or you can use the provided `AssetBundleLoader` `SubSystem` to do the work for you. There is nothing special about the `AssetBundleLoader`, it uses the same events you have access to, it just abstracts over the problem to give you a friendlier experience. The remainder of this article assumes you are using the subsystem.

[There is an example of the `AssetBundleLoader` running in the main indigo repo.](https://github.com/PurpleKingdomGames/indigo/blob/master/examples/assetLoading/src/main/scala/com/example/assetloading/AssetLoadingExample.scala)

### Using the Asset Bundler Loader

> Important! One advantage of loading everything up front is that you, the game developer, will find out _immediately_ whether or not you have an asset that can't load for some reason or other. If however, as an example, you defer an asset load until just before the last level, and don't give yourself a way to jump to the last level for testing - you won't know you have a bad asset until that point in your game's testing cycle.

To kick off an asset bundle, you need to fire off an `AssetBundleLoaderEvent.Load` event but attaching it to an `Outcome` or `SceneUpdateFragment`. In the example linked to above, we use a button (I've simplified here slightly):

```scala
val otherAssetsToLoad =
  Set(
    AssetType.Text(AssetName("map"), AssetPath(baseUrl + "assets/map.txt")),
    AssetType.Image(AssetName("font"), AssetPath(baseUrl + "assets/font.png")),
    AssetType.Image(AssetName("snake"), AssetPath(baseUrl + "assets/snake.png")),
    AssetType.Audio(AssetName("intro"), AssetPath(baseUrl + "assets/intro.mp3")),
    AssetType.Audio(AssetName("point"), AssetPath(baseUrl + "assets/point.mp3")),
    AssetType.Audio(AssetName("lose"), AssetPath(baseUrl + "assets/lose.mp3"))
  )

Button(
  buttonAssets = Assets.buttonAssets,
  bounds = Rectangle(10, 10, 16, 16),
  depth = Depth(2)
).withUpAction {
  println("Start loading assets...")
  List(AssetBundleLoaderEvent.Load(BindingKey("bundle 1"), otherAssetsToLoad))
}
```

As you can see, the asset bundle is just another `Set[AssetType]` like we'd use during the pre-start asset load. We're also required to provide a `BindingKey` instance so that we can track which asset bundle has loaded - or not as the case may be.

We can then track the progress of our bundle load by pattern matching the relevant events:

```scala
def updateModel(context: FrameContext, model: MyGameModel): GlobalEvent => Outcome[MyGameModel] = {
  case AssetBundleLoaderEvent.Started(key) =>
    println("Load started! " + key.toString())
    Outcome(model)

  case AssetBundleLoaderEvent.LoadProgress(key, percent, completed, total) =>
    println(s"In progress...: ${key.toString()} - ${percent.toString()}%, ${completed.toString()} of ${total.toString()}")
    Outcome(model)

  case AssetBundleLoaderEvent.Success(key) =>
    println("Got it! " + key.toString())
    Outcome(model.copy(loaded = true))
      .addGlobalEvents(PlaySound(AssetName("sfx"), Volume.Max)) // Make use of a freshly loaded asset.

  case AssetBundleLoaderEvent.Failure(key) =>
    println("Lost it... " + key.toString())
    Outcome(model)

  case _ =>
    Outcome(model)
}
```

#### Stating the obvious

You can't use an asset before you've loaded it, I think that should be clear.

The eagle eyed among you may have noticed that the super simple model above has a loaded flag in it's definition, here is the whole thing:

```scala
final case class MyGameModel(loaded: Boolean)
```

The loaded flag above is a crude way of us saying "Ok, the assets are ready for use now!", so that once set to true, we can start drawing with them, again a minimal example could be:

```scala
SceneUpdateFragment(
  if (model.loaded) {
    List(
      Graphic(Rectangle(0, 0, 64, 64), 1, Assets.junctionBoxMaterial)
        .moveTo(30, 30)
    )
  } else Nil
)
```

#### Using dynamically loaded `Text` assets

As previously discussed, images and sounds can be used directly but text can only be accessed during startup. As such, a bundle load triggers a full engine restart (which in our admittedly small tests, isn't noticeable) meaning that you can check for the existence of a text item at start up and use it if available.