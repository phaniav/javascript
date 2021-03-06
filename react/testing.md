# Eventbrite React Testing Best Practices

Guidelines and best practices used by Eventbrite to provide consistency and prevent errors in testing React components. This does not cover testing utility/helper functions (including Redux reducers) that are used in conjunction with React as those follow general testing guidelines.

## Table of Contents

0. [Testing environment](#testing-environment)
0. [Testing philosophy](#testing-philosophy)
0. [Writing a test case](#writing-a-test-case)
0. [Finding nodes](#finding-nodes)
0. [Finding components](#finding-components)
0. [Testing existence](#testing-existence)
0. [Assertion helpers](#assertion-helpers)
0. [Types of renderers](#types-of-renderers)
0. [Testing render](#testing-render)
0. [Testing events](#testing-events)
0. [Testing state](#testing-state)
0. [Testing updated props](#testing-updated-props)

## Testing environment

Eventbrite uses [`chai`](http://chaijs.com) (`expect` [BDD style](http://chaijs.com/api/bdd/)), [`enzyme`](http://airbnb.io/enzyme/) and [`sinon`](http://sinonjs.org/) for unit testing React components. We also leverage [`chai-enzyme`](https://github.com/producthunt/chai-enzyme) and [`sinon-chai`](https://github.com/domenic/sinon-chai) assertion helpers. Enzyme wraps [`ReactTestUtils`](https://facebook.github.io/react/docs/test-utils.html), which contains a bunch of primitives for testing components. Don't use `ReactTestUtils` directly; use Enzyme!

**[⬆ back to top](#table-of-contents)**

## Testing philosophy

Unit testing React components can be a little tricky compared to testing the input/output of traditional JavaScript functions. But it's still doable! Just like with "normal" unit testing, we want to test all of the logic within the component via its public interface. The public _input_ to a component is its props. The public _output_ of a component is the combination of the elements it specifically renders (see [Testing render](#testing-render)) as well as the callback handlers it invokes (see [Testing events](#testing-events)). The goal is to render components with various configurations of their props, so that we can assert that what is rendered and what callbacks are invoked is as expected.

**[⬆ back to top](#table-of-contents)**

## Writing a test case

Use [arrow functions](https://www.eventbrite.com/engineering/learning-es6-arrow-functions/) to force functional test cases:

```js
it('does what it is supposed to do', () => {

});
```

Using arrow functions prevents being able to use `beforeEach` & `afterEach` because `this` is now lexically scoped. In the past, data common to each test case was stored on `this` in `beforeEach` (and cleaned up in `afterEach`) so that each individual test case didn't have to generate the data itself. However, `beforeEach` devolved into a dumping ground for _anything_ that _may_ get used by more than one test case. As such way more data was generated than was needed, unnecessarily slowing down test execution.

Instead, factor out helper data generation functions and call them as needed in the test cases:

```js
const generateComponent = (additionalProps={}) => (
    <Component {...additionalProps} />
);

it('does what it is supposed to do', () => {
    let wrapper = mount(generateComponent());
});
```

**[⬆ back to top](#table-of-contents)**

## Finding nodes

Search for nodes within a component by adding `data-spec` attributes to them. In the past, Eventbrite used special `js-*` CSS classes for references to nodes in JavaScript code. These `js-*` classes were used when testing as well. Now with React testing, instead of using special CSS classes, [refs](https://github.com/eventbrite/javascript/tree/master/react#refs), or attempting to traverse the DOM with Enzyme's [`find`](http://airbnb.io/enzyme/docs/api/ReactWrapper/find.html) helper, we use `data-spec` attributes.

The `data-spec` attribute is specific to testing and not tied to presentation like CSS classes would be. If we decide to rename or remove a CSS class, the tests should not be impacted because there is no implicit link between styles and tests. We leverage two helpers, `getSingleSpecWrapper` & `getSpecWrappers` to find nodes with the `data-spec` attribute. Suppose we had the following (simplified) generated markup for a `Notification` component:

```html
<div class="notification">
    <button class="notification__close" data-spec="notification-close">X</button>
    <p class="notification__message">
        You have successfully registered for this event!
    </p>
    <a href="https://www.eventbrite.com/d/" class="notification__more-link" data-spec="notification-more-link">Browse all events</a>
</div>
```

Tests using `getSingleSpecWrapper` would look like:

```js
// good
it('has more link pointing to browse URL when `type` is browse', () => {
    let wrapper = mount(<Notification type="browse" />);
    let moreLinkWrapper = getSingleSpecWrapper(wrapper, 'notification-more-link');

    expect(moreLinkWrapper).to.have.attr('href', 'https://www.eventbrite.com/d/');
});

// bad (searches by tag name and CSS class)
it('has more link pointing to browse URL when `type` is browse', () => {
    let wrapper = mount(<Notification type="browse" />);
    let moreLinkWrapper = wrapper.find('a.notification__more-link');

    expect(moreLinkWrapper).to.have.attr('href', 'https://www.eventbrite.com/d/');
});
```

As a reference, here are the implementations for both helpers:

```js
// utils/unitTest.js

export const DATA_SPEC_ATTRIBUTE_NAME = 'data-spec';

/**
* Finds all instances of components in the rendered `componentWrapper` that are DOM components
* with the `data-spec` attribute matching `name`.
* @param {ReactWrapper} componentWrapper - Rendered componentWrapper (result of mount, shallow, or render)
* @param {string} specName - Name of `data-spec` attribute value to find
* @param {string|Function} typeFilter - (Optional) Expected type of the wrappers (defaults to all HTML tags)
* @returns {ReactComponent[]} All matching DOM components
*/
export const getSpecWrappers = (componentWrapper, specName, typeFilter) => {
    let specWrappers;

    if (!typeFilter) {
        specWrappers = componentWrapper.find(`[${DATA_SPEC_ATTRIBUTE_NAME}="${specName}"]`);
    } else {
        specWrappers = componentWrapper.findWhere((wrapper) => (
            wrapper.prop(DATA_SPEC_ATTRIBUTE_NAME) === specName && wrapper.type() === typeFilter
        ));
    }

    return specWrappers;
};

/**
* Like getSpecWrappers() but expects there to be one result, and returns that one result,
* or throws exception if there is any other number of matches besides one.
* @param {ReactWrapper} componentWrapper - Rendered componentWrapper (result of mount, shallow, or render)
* @param {string} specName - Name of `data-spec` attribute value to find
* @param {string|Function} typeFilter - (Optional) Expected type of the wrappers (defaults to all HTML tags)
* @returns {ReactComponent} Single matching DOM component
*/
export const getSingleSpecWrapper = (componentWrapper, specName, typeFilter) => {

    let specWrappers = getSpecWrappers(componentWrapper, specName, typeFilter);

    if (specWrappers.length !== 1) {
        throw new Error(`Expected single "${specName}" spec wrapper. Received: ${specWrappers.length}.`);
    }

    return specWrappers.first();
};
```

**[⬆ back to top](#table-of-contents)**

## Finding components

You can find a component simply by using Enzyme's [`find`](http://airbnb.io/enzyme/docs/api/ReactWrapper/find.html) and passing the component class:

```js
it('should render a checked checkbox if it is selected', () => {
    let wrapper = mount(<Component isSelected={true} />);
    let checkboxWrapper = wrapper.find(Checkbox);

    expect(checkboxWrapper).to.have.prop('isChecked', true);
});
```

This works as long as there's only one `Checkbox` rendered within `Component`. If there are multiple `Checkbox` components within `Component`, `checkboxWrapper` would have multiple elements in it. Instead you can add a `data-spec` attribute to the specific `Checkbox` and use `getSingleSpecWrapper`:

```js
// good
it('should render a checked checkbox if it is selected', () => {
    let wrapper = mount(<Component isSelected={true} />);

    // pass the component class as the third parameter to `getSingleSpecWrapper`
    let selectAllCheckboxWrapper = getSingleSpecWrapper(wrapper, 'component-selectAll', Checkbox);

    expect(selectAllCheckboxWrapper).to.have.prop('isChecked', true);
});

// bad (finds the appropriate Checkbox based on source order)
it('should render a checked checkbox if it is selected', () => {
    let wrapper = mount(<Component isSelected={true} />);
    let selectAllCheckboxWrapper = wrapper.find(Checkbox).at(2);

    expect(selectAllCheckboxWrapper).to.have.prop('isChecked', true);
});
```

The key in the "good" example is the third parameter passed to `getSingleSpecWrapper`. By default `getSingleSpecWrapper` will try to find a node with the specified `data-spec`. But if you specify the component class (`Checkbox` in this case), it'll return a reference to the component wrapper.

**[⬆ back to top](#table-of-contents)**

## Testing existence

### Testing node existence

To [find nodes](#finding-nodes) you use the `getSingleSpecWrapper` & `getSpecWrappers` helpers.

`getSingleSpecWrapper` throws an error if a single node is not found. Therefore, in order to determine if a node exists you actually have to test whether or not calling the helper throws an `Error`:

```js
let wrapper = mount(<Spinner />);

// wrap the call to retrieve the node in a function so that
// we can test to see if calling the function throws an Error
let findSvgWrapper = () => getSingleSpecWrapper(wrapper, 'spinner-svg');

// assert that node exists (doesn't throw an Error)
expect(findSvgWrapper).to.not.throw();

// assert that node doesn't exist (throws an Error)
expect(findSvgWrapper).to.throw();

// if node exists, you can retrieve it to run more assertions
// by calling the function
let svgWrapper = findSvgWrapper();
```

If you're not looking for a single node, you use `getSpecWrappers` which returns an Enzyme [`ReactWrapper`](https://github.com/airbnb/enzyme/tree/master/docs/api/ReactWrapper) of 0 or more node wrappers:

```js
let wrapper = mount(<Select values={dummyValues} />);
let selectOptionWrappers = getSpecWrappers(wrapper, 'select-option');

// assert that there are no found nodes
expect(selectOptionWrappers).to.be.blank();

// assert that there are more than zero found nodes
expect(selectOptionWrappers).to.not.be.blank();

// assert there to be a specific number of found nodes
expect(selectOptionWrappers).to.have.length(dummyValues.length);
```

**[⬆ back to top](#table-of-contents)**

### Testing component existence

Typically, you'll [find components](#finding-components) by using Enzyme's `find` method which returns an an Enzyme [`ReactWrapper`](https://github.com/airbnb/enzyme/tree/master/docs/api/ReactWrapper):

```js
let wrapper = mount(<Select values={dummyValues} />);
let selectOptionWrappers = wrapper.find(SelectOption);

// assert that there are no found nodes
expect(selectOptionWrappers).to.be.blank();

// assert that there are more than zero found nodes
expect(selectOptionWrappers).to.not.be.blank();

// assert there to be a specific number of found nodes
expect(selectOptionWrappers).to.have.length(dummyValues.length);
```

**[⬆ back to top](#table-of-contents)**

## Assertion helpers

Whenever possible, use `chai-enzyme` & `sinon-chai` assertion helpers in favor of the normal assertion helpers that just come with `chai`:

```js
// good (leverages `.prop` from `chai-enzyme`)
it('should render a checked checkbox if it is selected', () => {
    let wrapper = mount(<Component isSelected={true} />);
    let checkboxWrapper = wrapper.find(Checkbox);

    expect(checkboxWrapper).to.have.prop('isChecked', true);
});

// bad (just uses `enzyme` with vanilla `chai`)
it('should render a checked checkbox if it is selected', () => {
    let wrapper = mount(<Component isSelected={true} />);
    let checkboxWrapper = wrapper.find(Checkbox);

    expect(checkboxWrapper.prop('isChecked')).to.equal(true);
});
```

Functionally the "good" and "bad" assertions are the same. The assertions will both pass when the `isChecked` prop is `true` and both fail when it's `false`. The difference is in the reported error when they fail.

When the "good" assertion (using `chai-enzyme`'s [`prop`](https://github.com/producthunt/chai-enzyme#propkey-val) helper) fails, you'll receive an error such as:

```console
AssertionError: expected the node in <div /> to have a 'isChecked' prop with the value true, but the value was false
```

However, when the "bad" assertion fails, you'll receive a more cryptic (and less helpful) error such as:

```console
AssertionError: expected false to equal true
```

The "good" example has significantly more context and should be significantly more helpful when looking through failed test logs.

**[⬆ back to top](#table-of-contents)**

## Types of renderers

Enzyme provides three types of renderers for testing React components:

- [`mount`](http://airbnb.io/enzyme/docs/api/mount.html) - for components that may interact with DOM APIs, or may require the full lifecycle in order to fully test the component (i.e., `componentDidMount` etc.)
- [`shallow`](http://airbnb.io/enzyme/docs/api/shallow.html) - performant renderer because it renders only single level of children (no descendants of those children) in order to ensure that tests aren't indirectly asserting on behavior of child components
- [`render`](http://airbnb.io/enzyme/docs/api/render.html) - renders the components to traversable static HTML markup

Eventbrite uses `mount` for rendering **all** components when testing.

For components that just render markup ([atoms](http://bradfrost.com/blog/post/atomic-web-design/#atoms) in atomic web design), rendering with `mount` makes the most sense because they are the most likely to access the DOM API. Shallow rendering (via `shallow`) would be of little to no use.

For components that are a mix of markup and small components ([molecules](http://bradfrost.com/blog/post/atomic-web-design/#molecules) in atomic web design), rendering with `mount` also makes the most sense because of all the markup that still exists. It's simpler to stay consistent without the test file and use `mount` for all tests.

For components that are basically a composite of other components ([organisms](http://bradfrost.com/blog/post/atomic-web-design/#organisms) in atomic web design), we would ideally render with `shallow` because you're basically just testing that that the child components are receiving the correct props. Furthermore, it's faster to just render one level than render the entire markup tree, especially when the component is big. But in practice we make heavy use of [helper components](README.md#helper-components) in order to keep `render()` lean. As a result, what ends up being shallow rendered is not the actual child component, but an intermediary helper component. This means that if you wrote a test using `shallow` and then refactored the code to use helper components, your tests will break when the resultant render is actually still the same. Because of this nuance of when and where `shallow` can work, we've chosen to opt for `mount` because it always works. The trade-off is performance, which for now hasn't been a big enough issue.

**[⬆ back to top](#table-of-contents)**

## Testing render

When testing what a React component is rendering, _only test what the component itself renders_. Therefore if a parent component renders child components, such as a `TextFormField` component rendering `Label` & `TextInput` components, only test that the parent renders those child components and passes the appropriate props to them. **Do not** test the markup rendered by the child components because the tests for that child component will cover that.

```js
// good
it('displays label when `labelText` is specified', () => {
    let wrapper = mount(<TextFormField labelText="Name" />);
    let labelWrapper = wrapper.find(Label);

    // assert that when `labelText` is specified
    // the Label component is rendered
    expect(labelWrapper).to.not.be.blank();

    // assuming that `labelText` gets passed like:
    // <Label>{labelText}</Label>
    // asserts that it's properly passed
    expect(labelWrapper).to.have.prop('children', 'Name');
});

// bad (assumes the markup the child component is rendering)
it('displays label when `labelText` is specified', () => {
    let wrapper = mount(<TextFormField labelText="Name" />);
    let findLabelWrapper = () => getSingleSpecWrapper(wrapper, 'label');

    // assert that node exists (doesn't throw an Error)
    expect(findLabelWrapper).to.not.throw();
    let labelWrapper = findLabelWrapper();

    expect(labelWrapper).to.not.be.blank();
    expect(labelWrapper).to.have.text('Name');
});
```

The "bad" example assumes that the `Label` component is rendering a `<label>` tag, but the `TextFormField` component shouldn't really know or care what `Label` renders. It treats `Label` as a black box in its implementation so the test should do the same. Imagine if the `Label` component changed to render a `<div>` instead of a `<label>`. All of the tests for components using a `Label` component would now unnecessarily fail. On the other hand, the "good" example tests that the `TextFormField` properly renders the `<Label>` component and that the `labelText` prop is passed as its content (the `children` prop).

For HTML elements and their attributes, focus on the markup that's displayed based on logic. Static markup _can_ be tested, but has very diminishing returns.

```js
// good
it('includes the disabled CSS class when `isDisabled` is `true`', () => {
    let wrapper = mount(<Spinner isDisabled={true} />);

    // assert that the Spinner's container/root node has
    // "spinner--disabled" CSS class added
    expect(wrapper).to.have.className('spinner--disabled');
});

// bad (also asserts that the container node is a <div>)
it('includes the disabled CSS class when `isDisabled` is `true`', () => {
    let wrapper = mount(<Spinner isDisabled={true} />);

    // assert that the Spinner's container/root node
    // is a <div>
    expect(wrapper).to.have.tagName('div');

    // assert that the Spinner's container/root node has
    // "spinner--disabled" CSS class added
    expect(wrapper).to.have.className('spinner--disabled');
});
```

There's nothing fundamentally wrong with testing that the container/root node is a `<div>` in the "bad" example, but it os unnecessary since it's static and never changes. In fact, if your test file was filled with these sorts of additional assertions, it could make your test cases more fragile because presentational changes in your component could cause your component's tests to fail.

**[⬆ back to top](#table-of-contents)**

## Testing events

As mentioned in our [Testing philosophy](#testing-philosophy), part of the output of your component are the callback handlers it invokes. These event callbacks are functions passed as props to your component and need to be tested.

Test event callbacks by triggering the events that in turn will invoke the callback handler. The type of event triggered depends on whether the component contains HTML markup or child components.

### Testing events triggered by DOM

If you are testing an event callback that is triggered by a DOM event (such as `onClick` of an `<button>` node), you will need to simulate that DOM event. You will also need to stub the event callback prop to assert that it is being called with the correct arguments.

Let's say that there is a `TextInput` component that wraps an `<input type="text" />` DOM node. The `TextInput` has an `onChange` prop that gets called whenever the input field value changes. The `onChange` prop is also called with the current value that's in the input field. The test case would be set up like:

```js
it('properly fires `onChange` when input changes', () => {
    let stubbedOnChange = sinon.stub();

    // pass stubbed function to component as `onChange` prop
    let wrapper = mount(<TextInput onChange={stubbedOnChange} />);
    let inputWrapper = getSingleSpecWrapper(wrapper, 'text-input');
    let inputValue = 'Here is a value';

    // Create a fake event with the properties needed by the component
    let mockEvent = {
        target: {
            value: inputValue
        }
    };

    // simulate onChange event on input DOM
    inputWrapper.simulate('change', mockEvent);

    // assert that the stubbed function was called with the
    // expected value
    expect(stubbedOnChange).to.have.been.calledWith(inputValue);
});
```

The test case above relies on [Sinon.JS](http://sinonjs.org) for creating the stub. The stub is passed as the `TextInput` component's `onChange` prop so that we can make assertions on it at the end. After [finding a reference](#finding-nodes) to the input field, we simulate a fake `onChange` DOM event on the input field (using Enzyme's [`.simulate`](https://github.com/airbnb/enzyme/blob/master/docs/api/ReactWrapper/simulate.md) helper). Because the `TextInput` implementation expects to read `e.target.value` from an actual DOM event when it's running the browser, we have to mock that event with an object of the same structure. We don't need a full mock DOM event; we only need to mock what the code is actually calling.

Simulating the fake event on the input field will ultimately call our `stubbedOnChange` with its current value. Therefore, our assertion is that `stubbedOnChange` was not only called, but also called with the expected input value. This assertion leverages the `.calledWith` assertion helper from [`sinon-chai`](https://github.com/domenic/sinon-chai).

**[⬆ back to top](#table-of-contents)**

### Testing events triggered by child components

More than likely instead of your component adding event handlers directly to DOM nodes, it will be adding handlers to child components. Therefore instead of simulating a DOM event, simulate the child component's event handler being invoked.

Let's say you have an `AutocompleteField` component that has a child `TextInput`. The `AutocompleteField` has an `onChange` prop that is invoked whenever its child `TextInput`'s `onChange` event is invoked. The `AutocompleteField`'s `onChange` prop also passes the current input value. The test case would be set up like:

```js
it('properly fires `onChange` when input changes', () => {
    let stubbedOnChange = sinon.stub();

    // pass stubbed function to component as `onChange` prop
    let wrapper = mount(<AutocompleteField suggestions={[]} onChange={stubbedOnChange} />);
    let textInputWrapper = wrapper.find(TextInput);
    let inputValue = 'Here is a value';

    // We don't want to make any assumptions about the markup of `TextInput`. The
    // `AutocompleteField` component handles `onChange` of `TextInput`, so all we need to
    // do is call the prop directly like `TextInput` would and ensure we get the appropriate
    // value
    textInputWrapper.prop('onChange')(inputValue);

    // assert that the stubbed function was called with the
    // expected value
    expect(stubbedOnChange).to.have.been.calledWith(inputValue);
});
```

The test case above relies on [Sinon.JS](http://sinonjs.org) for creating the stub. The stub is passed as the `AutocompleteField` component's `onChange` prop so that we can make assertions on it at the end. After [finding a reference](#finding-components) to the `TextInput`, we simulate how `TextInput` would invoke its `onChange` callback prop. We get a reference to the prop using Enzyme's [`.prop`](https://github.com/airbnb/enzyme/blob/master/docs/api/ReactWrapper/prop.md) helper and call the function with the `inputValue`. This exactly how `TextInput` would call it when its DOM input field changes. However, because we don't want to make any assumptions about the markup of `TextInput` we simulate its `onChange` prop instead of digging into it in order to simulate its DOM.

Invoking the `onChange` prop will ultimately call our `stubbedOnChange` with the value. Therefore, our assertion is that `stubbedOnChange` was not only called, but also called with the expected input value. This assertion leverages the `.calledWith` assertion helper from [`sinon-chai`](https://github.com/domenic/sinon-chai).

**[⬆ back to top](#table-of-contents)**

## Testing state

Although `chai-enzyme` provides a [`.state()`](https://github.com/producthunt/chai-enzyme#statekey-val) helper method for asserting component state, it shouldn't be used in tests because the component's state is internal (and shouldn't be tested). Based on our [testing philosophy](#testing-philosophy), we only want to test the public API of the component.

When a component's state changes, the component is re-rendered, resulting in a change in markup. By testing only the changed markup (part of the component's public output), instead of the component's internal state, we can refactor the component's internals and have all of our test cases still pass. In sum, our test cases are a little less fragile.

Let's say for instance we had a component that has a `Checkbox` child component that toggles the component between inactive and active states. The active state is publicly represented by an `isActive` class added to the root DOM node. The test case could look something like:

```js
// good (tests internal state *indirectly* via re-rendered markup)
it('toggles active state when checkbox is toggled', () => {
    let wrapper = mount(<Component />);
    let checkboxWrapper = wrapper.find(Checkbox);

    // first assert that by default the active class is *not* present
    expect(wrapper).to.not.have.className('component--isActive');

    // simulate toggling the checkbox on by calling its
    // onChange callback handler passing `true` for
    // checked state
    checkboxWrapper.prop('onChange')(true);

    // now assert that the active class *is* present
    expect(wrapper).to.have.className('component--isActive');

    // simulate toggling the checkbox back off
    checkboxWrapper.prop('onChange')(false);

    // finally assert once again that active class is *not*
    // present
    expect(wrapper).to.not.have.className('component--isActive');
});

// bad (tests internal state directly)
it('toggles active state when checkbox is toggled', () => {
    let wrapper = mount(<Component />);
    let checkboxWrapper = wrapper.find(Checkbox);

    // assert that component's `isActive` internal state is
    // initially false
    expect(wrapper).to.have.state('isActive', false);

    // simulate toggling the checkbox on by calling its
    // onChange callback handler passing `true` for
    // checked state
    checkboxWrapper.prop('onChange')(true);

    // now assert that the `isActive` internal state is
    // true
    expect(wrapper).to.have.state('isActive', true);

    // simulate toggling the checkbox back off
    checkboxWrapper.prop('onChange')(false);

    // finally assert once again that `isActive` internal
    // state is false
    expect(wrapper).to.have.state('isActive', false);
});
```

Both the "good" and "bad" test cases are basically the same. The only difference is what is asserted. Ultimately, what we care about is that the root node has the appropriate CSS class; the changing of the internal `isActive` state just happens to be the mechanism that we accomplish it. This is what makes the "good" example better.

See [Testing events triggered by child components](#testing-events-triggered-by-child-components) for more on simulating child component events.

**[⬆ back to top](#table-of-contents)**

## Testing updated props

Typically components are stateless meaning that what is rendered by the component is 100% based upon the props that are based in. In these cases creating a component with initial props when [testing render](#testing-render) and [testing events](#testing-events) as explained above should suffice. There shouldn't be a need to test the re-render of a component receiving new props.

However, when a component leverages internal state and its props are changed, what will be rendered will be based on a combination of those updated props and the existing state. In this case, test that the new markup is as it should be, indirectly verifying that the updated prop(s) either have or have not overridden the existing state.

Let's say we have a `TextInput` component. It has `initialValue` & `value` props (among many others). The `initialValue` prop will initialize the `TextInput` component's underlying `<input>` node's value, but won't override the node if the prop is later updated. However, the `value` prop will both initialize the `<input>` as well as override its value.

To test the `initialValue` prop behavior:

```js
it('does NOT allow `initialValue` to override existing <input> value', () => {
    let initialValue = 'react';
    let newValue = 'enzyme';
    let wrapper = mount(<TextInput initialValue={initialValue} />);
    let inputWrapper = getSingleSpecWrapper(wrapper, 'text-input');

    // ensure that the `initialValue` is properly reflected
    // by checking the <input> node
    expect(inputWrapper).to.have.value(initialValue);

    // update the TextInput's props
    wrapper.setProps({value: newValue});

    // ensure that the <input> node's value hasn't changed
    expect(inputWrapper).to.have.value(initialValue);
});
```

To test the `value` prop behavior:

```js
it('DOES allow `value` to override existing <input> value', () => {
    let initialValue = 'react';
    let newValue = 'enzyme';
    let wrapper = mount(<TextInput initialValue={initialValue} />);
    let inputWrapper = getSingleSpecWrapper(wrapper, 'text-input');

    // ensure that the `initialValue` is properly reflected
    // by checking the <input> node
    expect(inputWrapper).to.have.value(initialValue);

    // update the TextInput's props
    wrapper.setProps({value: newValue});

    // ensure that the <input> node's value has changed
    expect(inputWrapper).to.have.value(newValue);
});
```

The key to passing new props to the existing `TextInput` component is the [`setProps`](https://github.com/airbnb/enzyme/blob/master/docs/api/mount.md#setpropsnextprops--reactwrapper) helper method. It will cause a re-render, which will allow us to assert that the new markup is as it should be.

**[⬆ back to top](#table-of-contents)**
