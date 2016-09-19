# auto-block
Simplified controller creation built around async.auto

[![NPM version](https://img.shields.io/npm/v/auto-block.svg)](https://www.npmjs.com/package/auto-block)
[![Build Status](https://img.shields.io/travis/TrackIF/auto-block/master.svg)](https://travis-ci.org/TrackIF/auto-block)

## Example usage

```javascript
    var controller = {
        data: {
            context: context,
            event: event,
        },
        optionsMapping: {
            'slug': 'event.slug',
            'feed': 'event.feed',
            'dryrun': 'event.dryrun'
        },
        responseMapping: 'results.query'
    }

    controller.block = {
        'feedConfig': {
            func: helpers.clients.getClientConfig,
            after: ['options'],
            with: [
            'options.slug',
            'options.feed'
            ]
        },
        'redshiftPassword': {
            func: helpers.secrets.kmsDecrypt,
            after: ['feedConfig'],
            with: {
            'payload': 'feedConfig.import.redshiftPassword'
            }
        }
    }

    autoBlock.run(controller, context.done)
```

## auto-block versus async.auto

async.auto is extremely useful for determining running order of interdependent
async functions:

```javascript
    async.auto({
        'one': function (results, cb) {
            // step one
        },
        'two': ['one', function (results, cb) {
            // step two
        }]
    }, function (err, results) {
        // all done
    })
```

When dealing with complex controllers, you often find it easier to move the
step implementations into their own functions:

```javascript
    function one(results, cb) {
        // step one
    }

    function two(results, cb) {
        // step two
    }

    async.auto({
        'one': one,
        'two': ['one', two]
    }, function (err, results) {
        // all done
    })
```

This allows for easier unit testing and debugging and helps keep your code
clean. But you may often find yourself adding lots of small wrappers to
convert values between the various steps:

```javascript
    function one(results, cb) {
        cb(null, {
            'alpha': 'foo',
            'beta': 'bar'
        })
    }

    function two(results, cb) {
        var params = {
            'alpha': results.one.alpha,
            'beta': results.one.beta
        }

        // step two
    }
```

auto-block allows you to declare all of the mappings in the same place you
declare the dependencies:

```javascript
    block = {
        'one': {
            func: one
        },
        'two': {
            func: two,
            with: {
                'alpha': 'one.alpha',
                'beta': 'one.beta'
            }
        }
    }
```

Similarly, controllers often need to detect errors or attaching extra data if
a step fails:

```javascript
    async.auto({
        'one': utility.doStuff,
        'two': function (results, cb) {
            var options = {
                'alpha': results.one.alpha,
                'beta': results.one.beta
            }
            utility.findUser(options, function (err, result) {
                if (err) {
                    err.status = 404
                    err.alpha = options.alpha
                }
                cb(err, result);
            })
        },
        'two': ['one', function (results, cb) {
            // step two
        }]
    }, function (err, results) {
        if (err && err.status) {
            res.status(err.status).send({
                message: err.message
            })
        }
    })
```

auto-block lets you declare these error mappings using `errorDefaults` and
`errorMappings`:

```javascript
    var controller: {
        block: {
            'one': {
                func: one
            },
            'two': {
                func: two,
                with: {
                    'alpha': 'one.alpha',
                    'beta': 'one.beta'
                },
                errorDefaults: {
                    'status': 404
                },
                errorMappings: {
                    'alpha': 'one.alpha'
                }
            }
        },
        done: function (err, results) {
            if (err && err.status) {
                res.status(err.status).send({
                    message: err.message
                })
            }
        }
    }
```

These, along with other features, allow you to build complex controllers with
needing to explicitly write function wrappers. auto-block does all of that for
you.

Without the need for extra wrappers, the temptation to inline business logic is
removed. You can comfortably move the business logic out of the controller
without needing to know the details of which controller module you used.

## Controller configuration

### .done

```javascript
    var controller = {
        done: function (error, response) {
            console.log('do stuff');
        }
    }
```

Called after the entire block has been completed. The values of `error` and
`response` are generated by their respective mapping configurations (see
below).

The `done` function can also be provided as the second parameter to `autoBlock.run`:

```javascript
    autoBlock.run(controller, function (error, response) {
        console.log('do stuff');
    })
```

This second parameter will _not_ override the `.done` field.

### .data

```javascript
    var controller = {
        data: {
            'foo': 'bar',
            'fizz': 'buzz'
        }
    }
```

Values provided in the `data` fields are available during `options`, `error` and
`response` mappings but _not_ during `results` mapping.

### .block

```javascript
    controller = {
        block: {
            'alpha': {
                // ...
            },
            'beta': {
                // ...
            }
        }
    }
```

`block` holds the actual steps used during `autoBlock.run`. See below for details on how to configure steps properly.

### Option mapping

The values in `.data` are not exposed to the individual steps. You can
explicitly expose them, however, using `optionsMapping`:

```javascript
    controller = {
        data: {
            'foo': {
                'bar': 'zaz'
            }
        },
        optionsDefaults: {
            'fizz': 'buzz'
            'bar': 'not zaz'
        },
        optionsMapping: {
            'bar': 'foo.bar'
        },
        block: {
            'alpha': {
                func: utility.doAlpha,
                with: {
                    'bar': 'options.bar', // resolves to 'zaz'
                    'fizz': 'options.fizz' // resolves to 'buzz'
                }
            }
        }
    }
```

Behind the scenes, auto-block builds a special `options` step that runs before
any other step you've declared on `.block`. If you don't need `optionsMapping`
or `optionsDefaults`, you can set your own `options` step:

```javascript
    controller = {
        block: {
            'options': {
                value: {
                    'fizz': 'buzz'
                }
            },
            'alpha': {
                func: utility.doAlpha,
                with: {
                    'fizz': 'options.fizz' // resolves to 'buzz'
                }
            }
        }
    }
```

### Error mapping

If a step produces an error, you can map additional fields onto the error before it is sent to `.done`:

```javascript
    controller = {
        data: {
            'foo': {
                'bar': 'zaz'
            }
        },
        errorsDefaults: {
            'fizz': 'buzz'
            'bar': 'not zaz'
        },
        errorsMapping: {
            'bar': 'foo.bar',
            'alpha': 'results.alpha'
        },
        block: {
            'alpha': {
                func: utility.doAlpha // result is 'alpha'
            },
            'beta': {
                func: utility.doBeta, // generates new Error('bad news')
                with: ['alpha']
            }
        },
        done: function (error, response) {
            // error will be similar to:
            // {
            //     message: 'bad news',
            //     data: {
            //         fizz: 'buzz',
            //         bar: 'zaz',
            //         alpha: 'alpha'
            //     }
            // }
        } 
    }
```

If necessary, you can also add error mapping for particular steps:

```javascript
    controller = {
        data: {
            'foo': {
                'bar': 'zaz'
            }
        },
        block: {
            'alpha': {
                func: utility.doAlpha // result is 'alpha'
            },
            'beta': {
                func: utility.doBeta, // generates new Error('bad news')
                with: ['alpha']
                errorsDefaults: {
                    'fizz': 'buzz'
                    'bar': 'not zaz'
                },
                errorsMapping: {
                    'bar': 'foo.bar',
                    'alpha': 'results.alpha'
                },
            }
        },
        done: function (error, response) {
            // error will be similar to:
            // {
            //     message: 'bad news',
            //     data: {
            //         fizz: 'buzz',
            //         bar: 'zaz',
            //         alpha: 'alpha'
            //     }
            // }
        } 
    }
```

### Response mapping

After all steps are completed, you can map values into the response parameter of `.done`:

```javascript
    controller = {
        data: {
            'foo': {
                'bar': 'zaz'
            }
        },
        responseDefaults: {
            'fizz': 'buzz'
            'bar': 'not zaz'
        },
        responseMapping: {
            'bar': 'foo.bar',
            'alpha': 'results.alpha'
        },
        block: {
            'alpha': {
                func: utility.doAlpha // result is 'alpha'
            }
        },
        done: function (error, response) {
            // response will be similar to:
            // {
            //    fizz: 'buzz',
            //    bar: 'zaz',
            //    alpha: 'alpha'
            // }
        } 
    }
```

### Hooks

auto-block provides five hooks that can be used for things like logging or debugging:

* `onStart(data)` -- called exactly once before any step is run
* `onStartStep(name, data)` -- called immediately after a step starts
* `onFinishStep(name, data, stepData)` -- called just before the step callback is run
* `onSuccess(response, data)` -- called after the response has been mapped if no error exists
* `onFailure(error, data)` -- called after the response has been mapped if an error exists

The `data` parameter noted above is the same as the `.data` configuration with a few extra fields added.

`stepData` is a string with some debugging information in it but is not well defined.

## Block configuration

Each key in `.block` represents one step that should be run. The definition for each step can include any number of settings:

### .func

`.func` is the asynchronous function that will be run during `.run`:

```javascript
    block: {
        'alpha': {
            func: utility.doAlpha
        }
    }
```

The last parameter of the function must be a callback. The number of other parameters is flexible (see `.when` below).

### .value

`.value` will merely add an object to the internal results payload. This can be useful for adding extra fields for mapping:

```javascript
    block: {
        'options': {
            value: {
                'foo': 'bar'
            }
        },
        'alpha': {
            func: utility.doAlpha,
            with: {
                'foo': 'options.foo'
            }
        }
    }
```

### .with

`.with` defines the parameter mapping to be used with `.func`. You can either define an object:

```javascript
    block: {
        'alpha': {
            func: utility.doAlpha,
            with: {
                'foo': 'options.foo',
                'fizz': 'fizz.buzz'
            }
        }
    }

    // calls utility.doAlpha({ 'foo': '...', 'fizz': '...' }, cb)
```

Or an array:

```javascript
    block: {
        'alpha': {
            func: utility.doAlpha,
            with: [
                'options.foo',
                'fizz': 'fizz.buzz'
            ]
        }
    }

    // calls utility.doAlpha('...', '...', cb)
```

The dot syntax starts with the results from all previous steps and will
automatically wait for those steps to complete:

```javascript
    block: {
        'beta': {
            func: utility.doBeta, // runs after doAlpha completes
            with: {
                'foo': 'alpha.foo'
            }
        },
        'alpha': {
            func: utility.doAlpha, // runs immediately
        }
    } 
```

### .after

You can add explicit dependencies using `.after`:

```javascript
    block: {
        'delta': {
            func: utility.doDelta, // runs after doAlpha and doBeta complete
            after: ['beta'],
            with: {
                'foo': 'alpha.foo'
            }
        },
        'alpha': {
            func: utility.doAlpha, // runs immediately
        },
        'beta': {
            func: utility.doBeta, // runs immediately
        }
    } 
```

### .when

Some steps are contingent on specific values or results from previous steps.
You can add these sorts of value dependencies using `.when`:

```javascript
    block: {
        'alpha': {
            func: utility.doAlpha,
        },
        'beta': {
            func: utility.doBeta,
            when: 'alpha.flag'
        }
    } 

    // doBeta will only run if doAlpha results in a value similar to:
    // {
    //     flag: true
    // }
```

`.when` settings will automatically add dependencies. In the above example, the
"beta" step will still occur after the "alpha" step.

Negative checks can be made by using a `!` prefix:

```javascript
    block: {
        'alpha': {
            func: utility.doAlpha,
        },
        'beta': {
            func: utility.doBeta,
            when: '!alpha.flag'
        }
    } 
```
