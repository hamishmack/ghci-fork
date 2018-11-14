# `:fork` command for `ghci`

If you have worked on long running process in ghci (like a server
or GUI), there is a good chance you have run into this problem:

``` Haskell
> import Control.Concurrent (forkIO, threadDelay, killThread)
> import Control.Monad (forever)
> t <- forkIO $ forever $ putStrLn "Hello" >> threadDelay 5000000
Hello
Hello
> :reload
Hello
> killThread t
error: Variable not in scope: t :: GHC.Conc.Sync.ThreadId
```

There are work arounds, but most introduce a dependency on
[foreign-store](http://hackage.haskell.org/package/foreign-store)
and/or require modification of your code.

## Using `:fork`

`:fork` forks a thread and stores it in the process environment.
If another thread was already in the selected "slot" it uses
`killThread` and waits for it to terminate before starting
the new thread.

For example:

``` Haskell
> import Control.Concurrent (forkIO, threadDelay, killThread)
> import Control.Monad (forever)
> :fork slotName forever $ putStrLn "Hello" >> threadDelay 5000000
Hello
> :reload
Hello
> :fork slotName forever $ putStrLn "World" >> threadDelay 5000000
World
World
...
```

The `slotName` identifies where the thread id is to be stored (any
combination of `isAlphaNum` characters or `'_'`).

To stop a thread just replace it with something that terminates:

``` Haskell
> :fork slotName return ()
```

## Installation

Paste the following into `ghci` or add it to a suitable `.ghci` file.
Feel free to add it to the startup code of tools that use `ghci` too.

``` Haskell
:{
:def! fork (\s ->
  let (slot, code) = Data.List.span (\c -> case c of
          '_' -> True
          ' ' -> False
          _ -> if Data.Char.isAlphaNum c
                  then True
                  else GHC.Base.error "Slot name must contain alpha, numbers and '_' only. Usage :fork slotName putStrLn \"Hello World\"") s
  in Control.Monad.return $ Data.String.unlines
    [":{"
    ,"System.Environment.lookupEnv \"GHCI_FORK_" Data.Monoid.<> slot Data.Monoid.<> "\" Control.Monad.>>="
    ,"(\\s ->"
    ,"  ( case s Control.Monad.>>= Text.Read.readMaybe of"
    ,"      Just n ->"
    ,"        let sPtr = Foreign.StablePtr.castPtrToStablePtr (Foreign.Ptr.wordPtrToPtr n)"
    ,"        in  Foreign.StablePtr.deRefStablePtr sPtr Control.Monad.>>="
    ,"            (\\(t, running) -> Control.Concurrent.killThread t Control.Monad.>>"
    ,"            Foreign.StablePtr.freeStablePtr sPtr Control.Monad.>>"
    ,"            Control.Monad.return running)"
    ,"      Nothing -> Control.Concurrent.newEmptyMVar"
    ,"  ) Control.Monad.>>="
    ,"(\\running -> Control.Concurrent.newEmptyMVar >>="
    ,"(\\sPtrSet -> Control.Concurrent.forkFinally"
    ,"  ( Control.Concurrent.takeMVar sPtrSet Control.Monad.>>"
    ,"    Control.Concurrent.putMVar running () Control.Monad.>>"
    ,"    ("
    ,     code
    ,"    )"
    ,"  ) (\\_ -> Control.Concurrent.takeMVar running) Control.Monad.>>="
    ,"(\\t -> Foreign.StablePtr.newStablePtr (t, running) >>="
    ,"(\\sPtr -> System.Environment.setEnv \"GHCI_FORK_" Data.Monoid.<> slot Data.Monoid.<> "\" (GHC.Show.show"
    ,"  (Foreign.Ptr.ptrToWordPtr (Foreign.StablePtr.castStablePtrToPtr sPtr))) Control.Monad.>>"
    ,"Control.Concurrent.putMVar sPtrSet ())))))"
    ,":}"
    ])
:}
```

## FAQ

### Why the strange code style?

This was done to avoid dependencies on the imported modules and enabled extensions.

### How can I redefine the `:reload` command so that it also restarts my thread?

``` Haskell
:def! reload (const $ return "::reload\n:fork mySlot MyModule.myMainFunction")
``` 
