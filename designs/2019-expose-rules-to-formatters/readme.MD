- Start Date: 2019-01-16
- RFC PR: https://github.com/eslint/rfcs/pull/10
- Authors: Chris Meyer (@EasyRhinoMSFT)

# Providing Rule Metadata to Formatters

## Summary

This proposal describes a design enhancement that provides formatters with details about the rules that have been executed by ESLint.

## Motivation

Currently, formatters only see the ID of each rule for which a violation was identified, plus an instance-specific description, as properties on each result object. Formatters are not able to access useful rule metadata, such as category, description, and help URL. Formatters are also not aware of the full set of rules that were run, information that may be useful in some cases.

## Detailed Design

Design Summary
1. In `cli.js::printResults`, obtain the rules map from the `Engine` object.
2. Add a second argument to the formatter's exported interface function. The value should be an object with a `rulesMeta` property that is a map with the rule name as the key and the `rule.meta` object as the value. See the "Command Line Interface (cli.js) Changes" section below for implementation.

We should use a container object as the argument, with a ruleId/rule.meta map as a property, in order to accommodate potential future expansions of the data we pass to formatters. This suggestion was previously made in the discussion of issue [#9841](https://github.com/eslint/eslint/issues/9841).

### Command Line Interface (cli.js) Changes
The implementation of this feature is very simple and straightfoward. The code location that invokes the formatter's exported interface function already has access to the API it should use to obtain the list of all executed rules. The call to `Engine.getRules` must be made in the try block because `engine` may be null during unit testing.

```js
function printResults(engine, results, format, outputFile) {
    let formatter;
    let rules;

    try {
        formatter = engine.getFormatter(format);
        rules = engine.getRules();
    } catch (e) {
        log.error(e.message);
        return false;
    }

    const rulesMeta = {};
    rules.forEach(function(rule, ruleId) {
        rulesMeta[ruleId] = rule.meta;
    });
    const output = formatter(results, { rulesMeta: rulesMeta });
    ...
}
```

### Formatter Changes

Formatters that implement the exported interface function would no changes. Future versions can make use of the rules data by adding the new argument to the exported interface function definition. This argument cannot be added unless it is used, as this will trip the JavaScript validation rule 'no-unused-vars.'

A formatter that assigns a function reference to the exported interface function could exhibit unexpected behavior depending on the signature of the referenced function. For example, since this change's second argument is a complex object, a referenced function that expects a Number as its second argument could cause an exception.

Currently the `html` formatter creates incorrect links rooted at the eslint.org domain for rules from plugins. We should fix this issue by using the meta.docs.url property that will become available with this change.

The `json` formatter also requires attention. It simply stringifies the `results` object, and would therefore provide incomplete data by ignoring the new `data` argument. To avoid a breaking change to the existing `json` formatter, we should a new built-in formatter, perhaps named `json-with-metadata`, which returns a stringified object containing both objects:

```js
module.exports = function(results, data) {
    return JSON.stringify({
        results: results,
        rulesMeta: data.rulesMeta
    });
};
```

## Documentation

Since custom formatter authors may want to take advantage of the newly-available rule metadata, a formal announcement may be justified (I don't have sufficient context in this regard so I will defer this determination.)

The [Working with Custom Formatters](https://eslint.org/docs/developer-guide/working-with-custom-formatters) article will have to be updated:
* Code samples will need the new `data` argument added wherever the exported interface function is shown, *but only when it is used*.
* The `data` argument should be called out and described, and should include a link to the [Working with Rules](https://eslint.org/docs/developer-guide/working-with-rules) article. The primary goal here is to familiarize formatter author with the structure of the `data` argument and rulesMeta property.
* It should be noted that the rulesMeta dictionary will be empty in cases where no rules have been run.
* It should be noted that rule metadata properties such as description, category, and help URL are not required and may not be defined, and that custom formatter code should take this into account.
* We should show the use of rule metadata in one of the examples by either modifying an existing one (maybe the [Detailed formatter](https://eslint.org/docs/developer-guide/working-with-custom-formatters#detailed-formatter) example) or adding a new one. One idea would be to suffix the results output with a list of rules that were violated, using a helper function something like this:

```js
var rulesViolated = {};
...
function printRules() {
    var lines = "*** RULES:\n";
    rulesViolated.forEach(function (ruleMetadata, ruleId) {
        lines += ruleId;
        
        if (ruleMetadata.docs.description) {
            lines += ": " + ruleMetadata.docs.description;
        }
        
        lines += "\n";
        
        if (ruleMetadata.docs.url) {
            lines += ruleMetadata.docs.url + "\n";
        }
    });
    return lines;
}
```

## Drawbacks

This is a fairly innocuous change in that it is additive, non-breaking (mostly, see Backwards Compatibility), and does not change any of ESLint's core functionality. A downside is that we will be exposing the Rule data model to third-party developers, so future changes could break existing formatters. For example, removing or renaming an existing property, or changing the structure of the Rule.meta object.

## Backwards Compatibility Analysis

Since this change is manifested as a new argument to the formatter's exported interface function, existing formatter code that implements the exported interface function will not be affected and will continue to function even without adding the new argument to their exported function.

(The following paragraph also appears in the Formatters section.)
A formatter that assigns a function reference to the exported interface function could exhibit unexpected behavior depending on the signature of the referenced function. For example, since this change's second argument is a complex object, a referenced function that expects a Number as its second argument could cause an exception.

## Alternatives

<!--
    What other designs did you consider? Why did you decide against those?

    This section should also include prior art, such as whether similar
    projects have already implemented a similar feature.
-->
* Including the rule metadata in the result object. This approach results in redundant data being returned, and includes external metadata properties that are not directly relevant.
* Pass the rules map itself as a argument to the formatter's exported interface function. This approach makes it messier to add additional data in the future, since new arguments would be necessary.

## Help Needed

No help needed, I have implemented the change.

## Frequently Asked Questions

<!--
    This section is optional but suggested.

    Try to anticipate points of clarification that might be needed by
    the people reviewing this RFC. Include those questions and answers
    in this section.
-->

## Related Discussions

Issue for this change:
https://github.com/eslint/eslint/issues/11273

Earlier related issue:
https://github.com/eslint/eslint/issues/9841

Initial inquiry:
https://groups.google.com/forum/#!topic/eslint/kpHrxkeilwE
