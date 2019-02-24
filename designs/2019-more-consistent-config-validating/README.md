[](- Start Date: 2019-02-24
- RFC PR: TBF
- Authors: 唯然 ([@aladdin-aad](https://github.com/@aladdin-aad))

# More consistent and simplier eslint config validating

## Summary

This proposal is to seek a way to make ESLint config validating more consistent, simpler, and easier to maintain.

 * validating phase:
   * unparseable env/rule config in a config file will cause a fatal error.
   * env/rule config will be validated.
   * invalid env/rule-options will emit a warning, but not block linting, and the invalid config will not be applied.

 * linting phase:
   * rule config in `/* eslint */` comment will be validated.
   * env config in `/* eslint-env */` comment will be validated.
   * unparseable env/rule config in comment will be reported as a linting warning.
   * invalid env/rule config in comment will be reported as a linting warning, and the invalid config will not be applied.
## Motivation

I was trying to fix [eslint/eslint#9505](https://github.com/eslint/eslint/issues/9505), and I didn't find an easy way to fix it.

Currently, ESLint's config validating is quite complicated, and sometimes inconsistent across different error scenarios.

| **Problem**                                                      | **Current error-handling behavior** |
|--------------------------------------------------------------|---------------------------------|
| enabling unrecognized rule in config file                    | Linting error                   |
| enabling unrecognized rule in `/* eslint */` comment         | Linting error(warning)                   |
| unrecognized rule in `/* eslint-disable */`                  | No error                        |
| Invalid rule config in config file                           | Fatal error                     |
| Invalid rule config in `Linter#verify`                       | No error                        |
| Unrecognized rule in `/* eslint */` comment                  | Linting error                   |
| Unparseable rule config in `/* eslint */` comment            | *Linting error*                   |
| Parseable, but invalid rule config in `/* eslint */` comment | *Fatal error*                     |
| Unrecognized rule in `/* eslint-disable */` comment          | No error                        |
| Unrecognized env in config file                              | Fatal error                       |
| Unrecognized env in `/* eslint-env */`                       | No error                            |

Additionally, it does not validate the rule schema at all.

## Detailed Design

### Changes to `Rules`

At the moment, `Rules.getRule()` will return a stub rule if passed a unrecognized ruleId. This is to reporting a linting error -- somewhat counter-intuitive! After the change, it will return `null` in this case.

### Changes to `config-validator`

At the moment, `validator.validate()` has some drawbacks:
+ it does not report if passed a non-exsitent ruleName.
+ it does not validate env at all

After the change, it will
+ export `validateRule()`, `validateEnv()`, `validate()`
+ throw an error when any invalid config found.

### Changes to `linter.js`

repalced the validating with `validator.validate()`.


### Changes to `rule-tester`

Use another ajv instance, which enabled validating rule schema.

## Documentation

TBF.

## Compatibility Analysis

### For end users:
* it has no impact if no invalid config found.
* it will slow the validating since more items need to be validated.
* it will fasten the linting when having a non-existent rule in the config, since it does not genererate a stub rule to run.

### For lib authors:
* it has no impact if no invalid rule schema found.
* it will slow the test runner.

## Open Questions

TBF

## Related Discussions

* [eslint/eslint#9373](https://github.com/eslint/eslint/issues/9373)
* [eslint/eslint#9505](https://github.com/eslint/eslint/issues/9505)
* [eslint/eslint#10713](https://github.com/eslint/eslint/pull/10713)