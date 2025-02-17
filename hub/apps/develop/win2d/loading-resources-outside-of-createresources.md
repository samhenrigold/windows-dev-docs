---
title: Loading resources outside of CreateResources
description: An explanation of how to load Win2D resources at any time in advanced scenarios.
ms.date: 05/28/2023
ms.topic: article
keywords: windows 10, windows 11, uwp, xaml, windows app sdk, winui, windows ui, graphics, games, effect win2d d2d d2d1 direct2d interop cpp csharp
ms.localizationpriority: medium
---

# Loading resources outside of CreateResources

This document discusses how apps using Win2D's XAML controls, CanvasControl, CanvasVirtualControl and CanvasAnimatedControl, can load resources from outside the CreateResources handler.

## Resource loading and CanvasControl / CanvasVirtualControl

Normally, apps are expected to use the `CreateResources` handler for creating controls' resources, so that device dependent resources are re-recreated as necessary if the device is lost. This includes resources that are loaded asynchronously. For asynchronous resource loading, controls are encouraged to use [`TrackAsyncAction(IAsyncAction)`](https://microsoft.github.io/Win2D/WinUI2/html/M_Microsoft_Graphics_Canvas_UI_CanvasCreateResourcesEventArgs_TrackAsyncAction.htm) with `CreateResources` to ensure correct behavior.

All of this works well for cases where all resources are loaded at startup only.

But, what about apps which need to load some resources at startup, and some other resources later? For example, consider a game with different levels, and the levels need different graphical assets. Win2D doesn't have something built-in with `CreateResources` to enable this- an app cannot manually tell a control, "Re-issue `CreateResources` now, so that I can load different assets from before". However, the building blocks are there to make this work, and allow very good flexibility for how and when the resources are loaded, and be robust with respect to lost device.

Really, what an app wants to do in this case, is have a custom `LoadResourcesForLevelAsync` method, a "custom" `CreateResources`-launched task, like this:

```csharp
async Task LoadResourcesForLevelAsync(CanvasControl resourceCreator, int level)
{
    levelBackground = await CanvasBitmap.LoadAsync(resourceCreator, ...);
    levelThingie = await CanvasBitmap.LoadAsync(resourceCreator, ...);
    // etc.
}
```

The app needs to load some of its resources after `CreateResources` is completed. In particular, the app will issue the level load after `CreateResources` has completed - e.g., from its `Draw` handler. In the code below, the app's `Draw` handler controls the progress of its level-loading Task.

To make `CreateResources` work in this situation, and be robust against lost devices, an app needs to do four things:
1. Track when `LoadResourcesForLevelAsync` is in progress.
2. Allow Win2D to handle any exceptions (in particular, device lost) that the app does't know how to handle.
3. If Win2D raises the `CreateResources` event to recover from a lost device while `LoadResourcesForLevelAsync` is in progress, your `CreateResources` handler should cancel that task.
4. If Win2D raises `CreateResources` to recover from a lost device after you have finished loading data using `LoadResourcesForLevelAsync`, your `CreateResources` handler must reload that custom data as well as its usual global resources.

Using a `CreateResources` handler called `CanvasControl_CreateResources`, and the `LoadResourcesForLevelAsync` method shown above, here is a complete implementation that handles all four requirements:

```csharp
 int? currentLevel, wantedLevel;

 // This implements requirement #1.
 Task levelLoadTask;


 public void LoadNewLevel(int newLevel)
 {
     Debug.Assert(levelLoadTask == null);
     wantedLevel = newLevel;
     levelLoadTask = LoadResourcesForLevelAsync(canvasControl, newLevel);
 }

 void CanvasControl_CreateResources(CanvasControl sender,
                                    CanvasCreateResourcesEventArgs args)
 {
     // Synchronous resource creation, for globally-required resources goes here:
     x = new CanvasRenderTarget(sender, ...);
     y = new CanvasRadialGradientBrush(sender, ...);
     // etc.

     args.TrackAsyncAction(CreateResourcesAsync(sender).AsAsyncAction());
 }  

 async Task CreateResourcesAsync(CanvasControl sender)
 {
     // If there is a previous load in progress, stop it, and
     // swallow any stale errors. This implements requirement #3.
     if (levelLoadTask != null)
     {
         levelLoadTask.AsAsyncAction().Cancel();
         try { await levelLoadTask; } catch { }
         levelLoadTask = null;
     }

     // Unload resources used by the previous level here.

     // Asynchronous resource loading, for globally-required resources goes here:
     baz = await CanvasBitmap.LoadAsync(sender, ...);
     qux = await CanvasBitmap.LoadAsync(sender, ...);
     // etc.

     // If we are already in a level, reload its per-level resources.
     // This implements requirement #4.
     if (wantedLevel.HasValue)
     {
        LoadNewLevel(wantedLevel.Value);
     }
 }

// Because of how this is designed to throw an exception, this must only 
// ever be called from a Win2D event handler.
bool IsLoadInProgress()
{
    // No loading task?
    if (levelLoadTask == null)
        return false;

    // Loading task is still running?
    if (!levelLoadTask.IsCompleted)
        return true;

     // Query the load task results and re-throw any exceptions
     // so Win2D can see them. This implements requirement #2.
     try
     {
         levelLoadTask.Wait();
     }
     catch (AggregateException aggregateException)
     {
         // .NET async tasks wrap all errors in an AggregateException.
         // We unpack this so Win2D can directly see any lost device errors.
         aggregateException.Handle(exception => { throw exception; });
     }
     finally
     {
         levelLoadTask = null;
     }

     currentLevel = wantedLevel;
     return false;
 }


 void CanvasControl_Draw(CanvasControl sender, CanvasDrawEventArgs args)
 {
     if (IsLoadInProgress())
     {
         DrawLoadingScreen();
     }
     else
     {
         DrawCurrentLevel(currentLevel);
     }
 }
 ```

 ## Resource loading and CanvasAnimatedControl

 Much of the above information for `CanvasControl` generalizes to `CanvasAnimatedControl`.

However, `CanvasAnimatedControl` has a game loop thread, which should be taken into consideration while deciding how an app should load its resources.

In particular:
- Apps should avoid blocking or waiting from within the `Draw` or `Update` handlers.
- Apps may need to synchronize data which is accessed between multiple threads.

We can adjust the above implementation to be compatible with `CanvasAnimatedControl`. In particular, we should ensure that the code checking "is the level loaded" will poll instead of wait, and no `CanvasAnimatedControl` event handlers should be marked as async.

```csharp
async Task LoadResourcesForLevelAsync(ICanvasAnimatedControl resourceCreator, int level)
{
    return GameLoopSynchronizationContext.RunOnGameLoopThreadAsync(resourceCreator, async() =>
    {
        levelBackground = await CanvasBitmap.LoadAsync(resourceCreator, ...);
        levelThingie = await CanvasBitmap.LoadAsync(resourceCreator, ...);
        // etc.
    }
}


// Shared state between all threads, and a lock to control access to it.
bool needToLoad;
int? currentLevel, wantedLevel;
Task levelLoadTask; // This implements requirement #1.

Object lockable = new Object();

void LoadNewLevel(int level)
{            
    lock(lockable)
    {
        wantedLevel = level;
        needToLoad = true;
    }
}

void canvasAnimatedControl_CreateResources(ICanvasAnimatedControl sender,
                                   CanvasCreateResourcesEventArgs args)
{
     // Synchronous resource creation, for globally-required resources goes here:
     x = new CanvasRenderTarget(sender, ...);
     y = new CanvasRadialGradientBrush(sender, ...);
     // etc.

     args.TrackAsyncAction(CreateResourcesAsync(sender).AsAsyncAction());
}

async Task CreateResourcesAsync(CanvasAnimatedControl sender)
{
    // If there is a previous load in progress, stop it, and
    // swallow any stale errors. This implements requirement #3.
    lock(lockable)
    {
        if (levelLoadTask != null)
        {
            levelLoadTask.AsAsyncAction().Cancel();
            try { await levelLoadTask; } catch { }
            levelLoadTask = null;
        }
    }

    // Unload resources used by the previous level here.

    // Asynchronous resource loading, for globally-required resources goes here:
    baz = await CanvasBitmap.LoadAsync(sender, ...);
    qux = await CanvasBitmap.LoadAsync(sender, ...);
    // etc.

    // If we are already in a level, reload its per-level resources.
    // This implements requirement #4.
    int? levelThatNeedsReloading;
    lock(lockable)
    {
        levelThatNeedsReloading = wantedLevel;
    }            
    if (levelThatNeedsReloading.HasValue)
    {
        LoadNewLevel(wantedLevel.Value);
    }
}

void canvasAnimatedControl_Update(ICanvasAnimatedControl sender, CanvasAnimatedUpdateEventArgs args)
{
    lock(lockable)
    {
        // Check if there is already an outstanding level-loading Task.
        // If so, don't try to spin up a new one.
        bool beginLoad = levelLoadTask == null && needToLoad;
        needToLoad = false;

        if (beginLoad)
        {
            levelLoadTask = LoadResourcesForLevelAsync(sender, wantedLevel);
        }

        // Indicates the loading task was run and just finished.
        if(levelLoadTask != null && levelLoadTask.IsCompleted)
        {
            AggregateException levelLoadException = levelLoadTask.Exception;
            levelLoadTask = null;

            // Query the load task results and re-throw any exceptions
            // so Win2D can see them. This implements requirement #2.
            if(levelLoadException != null)
            {
                // .NET async tasks wrap all errors in an AggregateException.
                // We unpack this so Win2D can directly see any lost device errors.
                levelLoadException.Handle(exception => { throw exception; });
            }

            currentLevel = wantedLevel;
        }
    }
}

bool IsLoadInProgress()
{
    lock(lockable)
    {
        return levelLoadTask != null;
    }
}


void canvasAnimatedControl_Draw(ICanvasAnimatedControl sender, CanvasAnimatedDrawEventArgs args)
{
    if (IsLoadInProgress())
    {
        DrawLoadingScreen(args.DrawingSession);
    }
    else
    {
        DrawCurrentLevel(args.DrawingSession, currentLevel);
    }
}
```

It's worth pointing out that there are special implications of using `await` from within application code which runs on the game loop thread. For example, when a task is run using `RunOnGameLoopThreadAsync` and contains an `await`, the first await may result in `RunOnGameLoopThreadAsync` finishing. The remainder of the task is scheduled according a continuation handler. And by default, this continuation handler is chosen from the application's thread pool, according to the default .NET `SynchronizationContext`. The remainder of the task might not run on the game loop thread at all.

To remedy this, and schedule work using `RunOnGameLoopThreadAsync` which contains `await` where the work must all be run on the game loop thread, see the `GameLoopSynchronizationContext` sample.