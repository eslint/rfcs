- Start Date: 2019-01-16
- RFC PR: (leave this empty, to be filled in later)
- Authors: Chris Meyer (@EasyRhinoMSFT)

# Providing Rule Metadata to Formatters

## Summary

This proposal describes a design enhancement that provides formatters with details about the rules that have been executed by ESLint.

## Motivation

Currently, formatters only see the ID of each rule for which a violation was identified, plus an instance-specific description, as properties on each result object. Formatters are not able to access usefule rule metadata, such as category, description, and help URL. Formatters are also not aware of the full set of rules that were run, information that may be useful in some cases.

## Detailed Design

Design Summary
1. In `cli.js::printResults`, obtain the rules map from the `Engine` object.
2. Pass the rules map as property in a new, second parameter object to the formatter's exported interface function.

We should use a container object as the parameter, with the rules map as a property, in order to accommodate potential future expansions of the data we pass to formatters. This suggestion was previously made in the discussion of issue [#9841](https://github.com/eslint/eslint/issues/9841).

### Command Line Interface (cli.js) Changes
The implementation of this feature is very simple and straightfoward. The code location that invokes the formatter's exported interface function already has access to the API it should use to obtain the list of rules, and the output of that function is a Map (dictionary) which is the type we want to pass to the formatter. This dataset includes all executed rules. The call to `Engine.getRules` must be made in the try block because `enginge` may be null during unit testing.

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

    const output = formatter(results, { rules: rules });
    ...
}
```

### Formatter Changes

No changes to the existing in-box formatters are necessary. Future versions can make use of the rules data by adding the new parameter to the exported interface function definition. This parameter cannot be added unless it is used, as this will trip the JavaScript validation rule 'no-unused-vars.'

## Documentation

Since custom formatter authors may want to take advantage of the newly-available rules data, a formal announcement may be justified (I don't have sufficient context in this regard so I will defer this determination.)

The [Working with Custom Formatters](https://eslint.org/docs/developer-guide/working-with-custom-formatters) article will have to be updated:
* Code samples will need the new `data` parameter added wherever the exported interface function is shown, *but only when it is used*.
* The `data` parameter should be called out and described, and should include a link to the [Working with Rules](https://eslint.org/docs/developer-guide/working-with-rules) article. The primary goal here is to familiarize formatter author with the structure of the `data` parameter and Rule object.
* It should be noted that Rule metadata properties such as description, category, and help URL are not required and may not be defined, and that custom formatter code should take this into account.
* We should show the use of rule data in one of the examples by either modifying an existing one (maybe the [Detailed formatter](https://eslint.org/docs/developer-guide/working-with-custom-formatters#detailed-formatter) example) or adding a new one. One idea would be to suffix the results output with a list of rules that were violated, using a helper function something like this:

```js
var rulesViolated = {};
...
function printRules() {
    var lines = "*** RULES:\n";
    rulesViolated.forEach(function (rule) {
        lines += rule.ruleId;
        
        if (rule.meta.docs.description) {
            lines += ": " + rule.meta.docs.description;
        }
        
        lines += "\n";
        
        if (rule.meta.docs.url) {
            lines += rule.meta.docs.url + "\n";
        }
    });
    return lines;
}
```

## Drawbacks

This is a fairly innocuous change in that it is additive, non-breaking, and does not change any of ESLint's core functionality. A downside is that we will be exposing the Rule data model to third-party developers, so future changes could break existing formatters. For example, removing or renaming an existing property, or changing the structure of the Rule object.

## Backwards Compatibility Analysis

Since this change is manifested as a new parameter to the formatter's exported interface function, existing formatter code will not be affected and will continue to function even without adding the new parameter to their exported function. I have confirmed this assertion via manual testing.

## Alternatives

<!--
    What other designs did you consider? Why did you decide against those?

    This section should also include prior art, such as whether similar
    projects have already implemented a similar feature.
-->
* Including the rule metadata in the result object. This approach results in redundant data being returned, and includes external metadata properties that are not directly relevant. (As an analogy, it's like ...)
* Pass the rules map itself as a parameter to the formatter's exported interface function. This approach makes it messier to add additional data in the future, since new parameters would be necessary.

## Open Questions

<!--
    This section is optional, but is suggested for a first draft.

    What parts of this proposal are you unclear about? What do you
    need to know before you can finalize this RFC?

    List the questions that you'd like reviewers to focus on. When
    you've received the answers and updated the design to reflect them, 
    you can remove this section.
-->
* Is it possible for a formatter to be invoked even though no rules have been run? IOW, could the caller suppress the inbox rules without providing any custom rules? The rules collection would be empty in this case, which formatters could potentially mishandle.
* Is there any opportunity for malicious manipulation of the rule data? I think not, since the analysis has completed by the time the formatter is invoked.

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
