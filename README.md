# react-reverse-portal [![Travis Build Status](https://img.shields.io/travis/httptoolkit/react-reverse-portal.svg)](https://travis-ci.org/httptoolkit/react-reverse-portal) [![Available on NPM](https://img.shields.io/npm/v/react-reverse-portal.svg)](https://npmjs.com/package/react-reverse-portal)

> _Part of [HTTP Toolkit](https://httptoolkit.tech): powerful tools for building, testing & debugging HTTP(S)_

**Build an element once, move it anywhere**

[React Portals](https://reactjs.org/docs/portals.html) let you render an element in a meaningful location within the React tree, but then place the output into a DOM node elsewhere.

Reverse portals let you do the opposite: pull a rendered element from elsewhere into a meaningful location within your React tree. This allows you to reparent DOM nodes, so you can move React-rendered elements around your React tree and the DOM without re-rendering them. Reverse portals also allow you to pull a rendered node out of the tree entirely, and return it later, all without rerendering the node.

This is useful in a few cases:

* Your react elements have internal state, and you'd like to persist that state but render the element somewhere new.
* Your DOM elements have built-in state (e.g. a playing `<video>` element), and you'd like to move the element elsewhere without losing that.
* Your elements are expensive to render, and you'd like to render them once and then place/unplace them later (e.g. a reusable pool of expensive-to-render elements that can be shared among different parts of your application).

## Features

* Reparent rendered React elements without rerendering
* Provide props at either the creation (in-portal) or usage (out-portal) locations, or both
* Single tiny file (2.5kB unminified, ungzipped) with zero dependencies
* Works with React 16+
* Written in TypeScript

## Getting Started

```
npm install react-reverse-portal
```

Create a portal node and initiate it somewhere:

```js
import * as portals from 'react-reverse-portal';

const MyComponent = (props) => {
    const portalNode = React.useMemo(() => portals.createPortalNode());

    return <div>
        {/*
            Create the content that you want to move around.
            InPortals render as normal, but to detached DOM.
            Until this is used MyExpensiveComponent will not
            appear anywhere in the page.
        */}
        <portals.InPortal node={portalNode}>
            <MyExpensiveComponent
                // Optionally provide props to use before this enters the DOM
                myProp={"defaultValue"}
            />
        </portals.InPortal>

        {/* ... The rest of your UI ... */}

        {/* Pass the node to whoever might want to show it: */}
        { props.componentToShow === 'component-a'
            ? <ComponentA portalNode={portalNode} />
            : <ComponentB portalNode={portalNode} /> }
    </div>;
}

const ComponentA = (props) => {
    return <div>
        {/* ... Some more UI ... */}

        {/* Show the content of the portal node here: */}
        <portals.OutPortal node={props.portalNode} />
    </div>;
}

const ComponentB = (props) => {
    return <div>
        {/* ... Some more UI ... */}

        <portals.OutPortal
            node={props.portalNode}
            myProp={"newValue"}     // Optionally, override default values
            myOtherProp={123}       // Or add new props

            // These props go back to the InPortal, and trigger a component
            // render (but on the same component instance) as if they had
            // been passed directly.
        />
    </div>;
}
```

## What just happened?

React-reverse-portal lets you render a node once, and then place it elsewhere, without being recreated from scratch.

Normally in `ComponentA`/`ComponentB` examples like the above, switching from `ComponentA` to `ComponentB` would cause every child component to be rebuilt, due to how React's DOM diffing & reconciliation works. Here, they're not, and you can reparent however you'd like.

How does it work under the hood?

### `portals.createPortalNode`

This creates a detached DOM node, with a little extra functionality attached to allow transmitting props later on.

This node will contain your portal contents later, and eventually be attached in the target location. By default it's a `div`, but you can pass your tag of choice (as a string) to override this if necessary. It's a plain DOM node, so you can mutate it to set any required props (e.g. `className`) with the standard DOM APIs.

### `portals.InPortal`

InPortal components take a `node` prop for the portal node, along with React children that will be rendered.

It reads any extra props for the children from the live OutPortals if any (via the node itself), and then renders the children (using these extra props if available) into the detached DOM node. At this point, the children are mounted & rendered React nodes, but living entirely outside the page DOM.

All extra props are given to _all_ children of the InPortal (except strings/null). If you need to give different props to different children, create a wrapper component to do so.

### `portals.OutPortal`

OutPortal components take a `node` for the portal node, and any extra props that should be passed through to the portal children.

It extra props to the InPortal (using the extra functionality we've attached to the portal node), and when rendered it attaches the detached DOM node in its place in the DOM. More specifically: the DOM node returned by `createPortalNode` will be attached in the place of `<portals.OutPortal>` in your React tree.

## Important notes

* **This is new and experimental! Try it at your own risk!**
* Nodes should be rendered to at most one OutPortal at any time. Rendering a node in two OutPortals simultaneously will make bad things happen, rendering the node in a random one or maybe none of the OutPortals, and probably setting up React to explode later. Each portal node should only be used in one place at any time.
* Rendering to zero OutPortals is fine: the node will be rendered as normal, using just the props provided inside the InPortal definition, and will continue to happily exist but just not appear in the DOM anywhere.
* Reverse portals tie rebuilding of the contents of the portal to InPortal (where it's defined), rather than the parent of the OutPortal (where it appears in the tree). That's great (that's the whole point really), but the contents of the InPortal will still be rebuilt anytime the InPortal itself is, e.g. if the InPortal's parent is rebuilt.
* Be aware that if you call `createPortalNode` in the render method of the parent of an InPortal, you'll get a different node to render into each time, and this will cause unnecessary rerenders, one every time the parent updates. It's generally better to create the node once and persist it, either using the useMemo hook, or in the initial construction of your class component.
* By default, the types for nodes, InPortals and OutPortals allow any props and any components. Pass a component type to them to be more explicit and enforce prop types, e.g. `createPortalNode<MyComponent>`, `<portals.InPortal<MyComponent>>`, `<portals.OutPortal<MyComponent>>`.