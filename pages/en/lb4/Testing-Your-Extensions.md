---
lang: en
title: 'Testing your extension'
keywords: LoopBack 4.0, LoopBack 4
tags:
sidebar: lb4_sidebar
permalink: /doc/en/lb4/Testing-your-extension.html
summary:
---

## Overview

An extension for LoopBack 4 is likely to be used by thousands and as such it's
important to have a thorough test suite for an extension because:
- Ensure your extension works correctly
- Prevent regressions when new features are added or bugs are fixed
- Helps demonstrate usage

## Project Setup

If using `@loopback/cli` to create a new extension project, it will come with a
`test` folder with a recommended folder for various types of tests.
An automated test suite requires a test runner to execute the tests and we
recommend `mocha`. It is enabled by default if `@loopback/cli` was used to
create the extension project. This will install and configure `mocha` as well
as create a `test` command in your `package.json`.

Assertion libraries such as [ShouldJS](http://shouldjs.github.io/) (as `expect`),
[SinonJS](http://sinonjs.org/), and a swagger validator are made available
through the convenitent `@loopback/testlab` package. This is installed if the
project was created using the CLI.

### Manual Setup - Using Mocha
- Install `mocha` by running `npm i mocha`. This will save the package in
`package.json` as well.
- In `package.json` add under `scripts` the following:
`test: npm run build && mocha --recursive ./dist/test`

## Types of tests

A comprehensive test suite tests many aspects of your code. It is encouraged to
test as many aspects of your application from a variety of perspectives to
ensure correctness, integration and ensuring future compatibility. To accomplish
this you should write unit, integration and acceptance tests.

You may use any development methodology you want to write your extension, the
important thing is to have an automated test suite.
- **Traditional development:** write the code first and then write the tests
- **Test-driven development:** write tests first, see them fail, then write the
code to pass the tests

### Unit Tests

A unit test checks the smallest unit of code possible, in this case a function.
This helps to ensure variable and state changes by outside actors don't affect
the result. If the function has dependencies, you should substitute them with
[test doubles](https://en.wikipedia.org/wiki/Test_double) to ensure only the
function you are testing affects the results. SinonJS is our recommended
framework to create test doubles in the form of spies, stubs, and mocks.

#### Controllers

At it's core, a controller is a simple class that is responsible for related
actions on a object. Unit testing a controller in an extension is the same as
unit testing a controller for an application.

To test a controller you want to instantiate a new instance of your controller
class and test a function, providing a test double for constructor arguments as
needed.

**`src/controllers/ping.controller.ts`**
```ts
export class PingController {
  @get('/ping')
  ping(msg?: string) {
    return `You pinged with ${msg}`;
  }
}
```

**`test/unit/controllers/ping.controller.ts`**
```ts
import {PingController} from '../../..';
import {expect} from '@loopback/testlab';

describe('PingController() unit', () => {
  it('.ping() (no arg)', () => {
    const controller = new PingController();
    const result = controller.ping();
    expect(result).to.equal('You pinged with undefined');
  });

  it('.ping(\'hello\')', () => {
    const controller = new PingController();
    const result = controller.ping('hello');
    expect(result).to.equal('You pinged with hello');
  });
});
```

You can find a more advanced example on testing controllers in [Unit test your Controllers](http://loopback.io/doc/en/lb4/Testing-your-application.html#unit-test-your-controllers).

#### Decorators

The recommended usage of a decorator is to store metadata about a class or a
class method. The decorator implementation will usually also provide a function
to retrieve the metadata stored by it based on the className and method name.
So for a unit test, it is important to test that we can store metadata via a
decorator and retrieve the correct metadata. *The retrieval gets tested as a
consequence of validating the metadata was stored.*

**`src/decorators/test.decorator.ts`**
```ts
export function test(file: string) {
  return function(target: Object, methodName: string): void {
    Reflector.defineMetadata(
      'example.msg.decorator.metadata.key',
      {file},
      target,
      methodName,
    );
  };
}

export function getTestMetadata(
  controllerClass: Constructor<{}>,
  methodName: string,
): {file: string} {
  return Reflector.getMetadata(
    'example.msg.decorator.metadata.key',
    controllerClass.prototype,
    methodName,
  );
}
```

**`test/unit/decorators/test.decorator.ts`**
```ts
import {test, getTestMetadata} from '../../..';
import {expect} from '@loopback/testlab';

describe('test.decorator (unit)', () => {
  it('can store test name via a decorator', () => {
    class TestClass {
      @test('me.test.ts')
      me() {}
    }

    const metadata = getTestMetadata(TestClass, 'me');
    expect(metadata).to.be.a.Object();
    expect(metadata.file).to.be.eql('me.test.ts');
  });
});
```

#### Mixins

A Mixin is a function that extends a base Class (usually `Application` Class)
and returns an anonymous class, adding new contructor properties, methods, etc.
to the base Class. Due to the fact that the Mixin returns an anonymous class,
it is not possible to write a unit test in isolation for a Mixin. The
recommended practice is to write an integration test for a Mixin as shown [here: Integrations Tests Mixins]().

#### Providers

A Provider is a Class, similar to a Controller but it implements the Provider
interface. This interface just requires the Class to have a `value()` function.
A unit test for a provider should test the `value()` function by instantiating
a new Provider class, using a test double for any constructor arguments.

**`src/providers/random-number.provider.ts`**
```ts
import {Provider} from '@loopback/context';

export class RandomNumberProvider implements Provider<number> {
  value(): number {
    return (max: number): number => {
      return Math.floor(Math.random() * max) + 1;
    };
  }
}
```

**`test/unit/providers/random-number.unit.test.ts`**
```ts
import {RandomNumberProvider} from '../../..';
import {expect} from '@loopback/testlab';

describe('RandomNumberProvider (unit)', () => {
  it('generates a random number within range', () => {
    const provider = new RandomNumberProvider().value();
    const random: number = provider(3);

    expect(random).to.be.a.Number();
    expect(random).to.equalOneOf([1, 2, 3]);
  })
})
```

#### Repositories

*TBD ... How / What?*

### Integration Tests

An integration test plays an important part in your test suite by ensuring your
extension artifacts play well with each other as well as `@loopback`. It is
recommended to test two items together and other integrations can be
substituted by test doubles so it becomes apparent where the integration breaks
(if it does).

#### Controllers

*TBD ... Repositories? Other Services?*

#### Decorators

*TBD ... How / What?*

#### Mixins

As mentioned under unit/mixins section, a Mixin extends a base Class by returning
an anonymous class. As such to test a Mixin, it should be done by actually
using the mixin with it's base Class. Since this requires two Classes to work
together, an integration test will be needed. A Mixin should be tested to ensure
new / overridden methods exist and work as expected in the new Mixed class. 

**`src/mixins/time.mixin.ts`**
```ts
import {Constructor} from '@loopback/context';
export function TimeMixin<T extends Constructor<any>>(superClass: T) {
  return class extends superClass {
    constructor(...args: any[]) {
      super(...args);
      if (!this.options) this.options = {};

      if (typeof this.options.timeAsString !== 'boolean') {
        this.options.timeAsString = false;
      }
    }

    time() {
      if (this.options.timeAsString) {
        return new Date().toString();
      }
      return new Date();
    }
  };
}
```

**`test/integration/mixins/time.intg.test.ts`**
```ts
import {expect} from '@loopback/testlab';
import {Application} from '@loopback/core';
import {TimeMixin} from '../../..';

describe('TimeMixin (integration)', () => {
  it('mixed class has .time()', () => {
    const myApp = new AppWithTime();
    expect(typeof myApp.time).to.be.eql('function');
  });

  it('returns time as string', () => {
    const myApp = new AppWithLogLevel({
      timeAsString: true,
    });

    const time = myApp.time();
    expect(time).to.be.a.String();
  });

  it('returns time as Date', () => {
    const myApp = new AppWithLogLevel();

    const time = myApp.time();
    expect(time).to.be.a.Date();
  });

  class AppWithTime extends TimeMixin(Application) {}
});
```

#### Providers

*TBD ... How / What?*

#### Repositories

*TBD ... How / What*

### Acceptance Test

An Acceptance test is the ultimate test. This is a comprehensive test, written
end-to-end that is responsible for covering the user scenario. This acceptance
test should make use of all extension artifacts such as decorators, mixins,
providers, repositories, etc. No test doubles should need to be used for an
Acceptance Test. This is a black box test where you don't know / care about the
internals of the extensions. You will be thinking like the consumer of the
extension (using it, and it doing a thing as expected).

Due to the complexity of an acceptance test, there is no example given here.
Have a look at a [loopback4-example-log-extension](https://github.com/strongloop/loopback4-example-log-extension)
to understand the extension artifacts and usage. Then you can see the acceptance
test [here: test/acceptance/log.extension.acceptance.ts](https://github.com/strongloop/loopback4-example-log-extension/blob/master/test/acceptance/log.extension.acceptance.ts).
