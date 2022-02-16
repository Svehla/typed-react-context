## Goal

The goal of this tutorial is to write "strong" state management with 100% type inference from the javascript code.

## TLDR:

[Final example of the state management is available on github](https://github.com/Svehla/typed-react-context/blob/master/index.tsx)

or you can find a fully working example at the end of this article.

## Historical background

React introduced hooks about 2 years ago.
It changed the whole ecosystem and it shows up that we can write an application without using external
state management libraries like *redux* or *mobx* and we'll still have nice minimalist code.

We were able to do the same even before the hooks were introduced,
but the problem was that the `renderProps`/`HOC`/`Classes` API wasn't that nice and elegant as hooks are.

[If you know that you want to use Redux and you're struggling with Typescript type inference, you can check this article](https://dev.to/svehla/typescript-100-type-safe-react-redux-under-20-lines-4h8n)

Tooling of vanilla React is still pretty strong but if you have an application
with tons of lines of code that's too complex for ordinary humans, you can
start to think about some third-party state management libraries.

## Custom state management Wrapper

React context is a nice option on how to split parts of your global application logic into different
files and define a new `React.createContext` for each module.
Then you just import the context instance and use it in the component instance by `useContext` hook.
A great feature of this pattern is that you don't re-render components that are not directly connected to the state that is changed.

In pure vanilla React you can write your state management via context like this.

```javascript
import React, { useState, useContext } from 'react'
const MyContext = React.createContext(null)

const LogicStateContextProvider = (props) => {
  const [logicState, setLogicState] = useState(null)

  return (
    <MyContextontext.Provider value={{ logicState, setLogicState }}>
      {...props}
    </MyContextontext.Provider>
  )
}

const Child = () => {
  const logic = useContext(MyContext)
  return <div />
}

const App = () => (
  <LogicStateContextProvider>
    <Child />
  </LogicStateContextProvider>
)
```

Everything looks nice until you start to add Typescript static types.
Then you realize that you have to define a new data type for each `React.createContext` definition.

```typescript

/* redundant unwanted line of static type */
type DefinedInterfaceForMyCContext = {
  /* redundant unwanted line of static type */
  logicState: null | string
  /* redundant unwanted line of static type */
  setLogicState: React.Dispatch<React.SetStateAction<boolean>>
  /* redundant unwanted line of static type */
}

const MyContext = React.createContext<BoringToTypesTheseCha>(
  null as any /* ts hack to omit default values */
)

const LogicStateContextProvider = (props) => {
  const [logicState, setLogicState] = useState(null as null | string)

  return (
    <MyContext.Provider value={{ logicState, setLogicState }}>
      {...props}
    </MyContext.Provider>
  )
}

/* ... */
```

As you can see, each `React.createContext` takes a few extra lines for defining Typescript static types
which can be easily inferred directly from the raw Javascript implementation.

Above all, you can see that the whole problem with inferring comes from the JSX. It's not impossible to infer data types from it!

So we have to extract raw logic directly from the Component and put it into a custom hook named `useLogicState`.

```typescript
const useLogicState = () => {
  const [logicState, setLogicState] = useState(null as null | string)

  return {
    logicState,
    setLogicState
  }
}

const MyContext = React.createContext<
  /* some Typescript generic magic */
  ReturnType<typeof useLogicState>
>(
  null as any /* ts hack to bypass default values */
)

const LogicStateContextProvider = (props) => {
  const value = useLogicState()

  return (
    <MyContext.Provider value={value}>
      {...props}
    </MyContext.Provider>
  )
}

const Child = () => {
  const logic = useContext(MyContext)
  return <div />
}

const App = () => (
  <LogicStateContextProvider>
    <Child />
  </LogicStateContextProvider>
)
```

As you can see, decoupling logic into a custom hook enable us to infer the data type by `ReturnType<typeof customHook>`.

***

If you don't fully understand this line of TS code `ReturnType<typeof useLogicState>` you can check my other Typescript tutorials.
- https://dev.to/svehla/typescript-inferring-stop-writing-tests-avoid-runtime-errors-pt1-33h7
- https://dev.to/svehla/typescript-generics-stop-writing-tests-avoid-runtime-errors-pt2-2k62

***

I also don't like the fact that there is a lot of redundant characters which you have to have in the code
every time you want to create new *React context* and it's own JSX `Provider` Component which we use to wrap our `<App />`.

So I have decided to extract and wrap all dirty code in its own function.
Thanks to that we can also move that magic Typescript generic into this function and we'll be able to infer the whole state management.


```typescript
type Props = { 
  children: React.ReactNode 
}

export const genericHookContextBuilder = <T, P>(hook: () => T) => {
  const Context = React.createContext<T>(undefined as never)

  return {
    Context,
    ContextProvider: (props: Props & P) => {
      const value = hook()

      return <Context.Provider value={value}>{props.children}</Context.Provider>
    },
  }
}

```

So we can wrap all this magic which is hard to read into a ten-line function.

Now the `genericHookContextBuilder` function takes our state hook as an argument and generates Component which will work
as an App Wrapper and Context which can be imported into `useContext`.

we're ready to use use it in the next example.


## Full example


```typescript
import React, { useState, useContext } from 'react';


type Props = {
  children: React.ReactNode
}

export const genericHookContextBuilder = <T, P>(hook: () => T) => {
  const Context = React.createContext<T>(undefined as never)

  return {
    Context,
    ContextProvider: (props: Props & P) => {
      const value = hook()

      return <Context.Provider value={value}>{props.children}</Context.Provider>
    },
  }
}

const useLogicState = () => {
  const [logicState, setLogicState] = useState(null as null | string)

  return {
    logicState,
    setLogicState
  }
}

export const {
  ContextProvider: LogicStateContextProvider,
  Context: LogicStateContext,
} = genericHookContextBuilder(useLogicState)

const Child = () => {
  const logic = useContext(LogicStateContext)
  return <div />
}

const App = () => (
  <LogicStateContextProvider>
    <Child />
  </LogicStateContextProvider>
)

```

![Alt Text](https://raw.githubusercontent.com/Svehla/typed-react-context/master/imgs/preview-1.png)

![Alt Text](https://raw.githubusercontent.com/Svehla/typed-react-context/master/imgs/preview-2.png)

As you can see, we have written a small wrapper around native React context default verbose API.
The wrapper enhanced it with out-of-the-box Typescript type inference, which enabled us not to duplicate code and to save a lot of extra lines.


I hope that you enjoyed this article the same as me and learned something new. If yes don't forget to like this article