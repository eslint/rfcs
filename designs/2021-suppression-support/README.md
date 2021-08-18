- Repo: eslint/eslint
- Start Date: 2021-07-28
- RFC PR: (leave this empty, to be filled in later)
- Authors: Yiwei Ding

# Add suppression information in the output of ESLint

## Summary

Violations can be suppressed by inline directive comments. We propose to add an `--output-suppression` CLI option for adding suppression information, including its kind and justification, in the output of ESLint.

The code below shows the example in the format of SARIF and JSON:

```json
SARIF format
{
  ... Some other info ...
  =======================
  "results": [
    {
      "level": "error",
      "message": {
        "text": "'a' is not defined."
      },
      "locations": [
        {
          "physicalLocation": {
            "artifactLocation": {
              "uri": "file:///C:/Users/yiweiding/Documents/projects/test/1.js",
              "index": 0
            },
            "region": {
              "startLine": 1,
              "startColumn": 1
            }
          }
        }
      ],
      "suppressions": [
        {
          "kind": "inSource",
          "justification": "Sample justification message."
        }
      ],
      "ruleId": "no-undef",
      "ruleIndex": 0
    }
  ]
}
```

```json
JSON format
[
  {
    "filePath":"C:\\Users\\yiweiding\\Documents\\projects\\test\\1.js",
    "messages":[
      {
        "ruleId":"no-undef",
        "severity":2,
        "message":"'a' is not defined.",
        "line":1,
        "column":1,
        "nodeType":"Identifier",
        "suppressions":[
          {
            "kind":"inSource",
            "justification":"Sample justification message."
          }
        ],
        "messageId":"undef",
        "endLine":1,
        "endColumn":2
      }
    ],
    "errorCount":0,
    "warningCount":0,
    "fixableErrorCount":0,
    "fixableWarningCount":0,
    "usedDeprecatedRules":[]
  }
]
```

## Motivation

### Problem

ESLint is a famous tool for identifying and reporting on patterns found in ECMAScript/JavaScript code. When codes violate one or more defined rules, ESLint will report warnings or errors:

```json
"messages": [
  {
    "ruleId":"no-undef",
    "severity":2,
    "message":"'b' is not defined.",
    "line":2,
    "column":1,
    "nodeType":"Identifier",
    "messageId":"undef",
    "endLine":2,
    "endColumn":2
  }
]
```

![Violation](violation.png)

In our scenarios, when violations are suppressed, Security Development Lifecycle (SDL) tools, such as ESLint, are expected to export suppression information, including its kind and justification. We could take advantage of this info to generate corresponding signals for auditing purposes.

```json
"messages": [
  {
    "ruleId":"no-undef",
    "severity":2,
    "message":"'a' is not defined.",
    "line":2,
    "column":1,
    "nodeType":"Identifier",
    "messageId":"undef",
    "endLine":2,
    "endColumn":2,
    "suppressions": [
      {
        "kind":"inSource",
        "justification":"Sample justification message."
      }
    ]
  }
]
```

### Suppression in ESLint

There are 2 ways to suppress (or disable in ESLint) warnings/errors (violations), according to [disabling-rules](https://github.com/eslint/eslint/blob/master/docs/user-guide/configuring/rules.md#disabling-rules) and [comments in GitHub](https://github.com/eslint/eslint/issues/14784#issuecomment-878522942):

1. **inSource**: use directive comments inside of a file

    1. Block comment - Codes between `eslint-disable` and `eslint-enable` are suppressed by defined rules. If rules are not specified, any warnings or errors will be suppressed.
        ```javascript
        /* eslint-disable no-undef -- Justification example */
        a = 1;
        /* eslint-enable no-undef */
        ```
    2. Line comment - `eslint-disable-line` can be used to suppress current line of code, and `eslint-disable-next-line` to suppress the next line of code.
        ```javascript
        console.log('foo'); // eslint-disable-line no-console -- Justification example
        ```
        ```javascript
        // eslint-disable-next-line no-console -- Justification example
        console.log('bar');
        ```
2. **External**: switch rules to `off` in configuration files

    Configuration files can be in JavaScript/YAML/JSON formats. For example, in JSON, rules can be switched to `off`:

    ```json
    rules: {
      "no-undef": "off"
    }
    ```

    Then `no-undef` rule will be suppressed globally.

Violations can be suppressed inline more than one times:

```javascript
/* eslint-disable no-undef -- Justification example 1 */
a = 1; // eslint-disable-line no-undef -- Justification example 2
/* eslint-enable no-undef */
```

In that case, the first suppression will work while the rest of them will be treated as `unusedDisableDirectives`.

When an error or warning is suppressed, it will not be output. Particularly, an option `--no-inline-config` can disable all inline suppressions, where all errors and warnings not disabled by configuration files will be output.

Currently, ESLint cannot dump these output messages into the SARIF format. A tool [ESLint.Formatter](https://github.com/microsoft/sarif-sdk/blob/main/src/ESLint.Formatter/sarif.js) can be used to convert the output of ESLint to SARIF format:

```
.\node_modules\.bin\eslint.cmd 1.js -f @microsoft/eslint-formatter-sarif -o 1.sarif
```

```json
SARIF format
{
  ... Some other info ...
  =======================
  "results": [
    {
      "level": "error",
      "message": {
        "text": "'a' is not defined."
      },
      "locations": [
        {
          "physicalLocation": {
            "artifactLocation": {
              "uri": "file:///C:/Users/yiweiding/Documents/projects/test/1.js",
              "index": 0
            },
            "region": {
              "startLine": 1,
              "startColumn": 1
            }
          }
        }
      ],
      "ruleId": "no-undef",
      "ruleIndex": 0
    }
  ]
}
```

JSON is a more popular format for the users of ESLint:

```
.\node_modules\.bin\eslint.cmd 1.js -f json -o 1.json
```

```json
JSON format
[
  {
    "filePath":"C:\\Users\\yiweiding\\Documents\\projects\\test\\1.js",
    "messages":[
      {
        "ruleId":"no-undef",
        "severity":2,
        "message":"'a' is not defined.",
        "line":1,
        "column":1,
        "nodeType":"Identifier",
        "messageId":"undef",
        "endLine":1,
        "endColumn":2
      }
    ],
    "errorCount":0,
    "warningCount":0,
    "fixableErrorCount":0,
    "fixableWarningCount":0,
    "usedDeprecatedRules":[]
  }
]
```

Figure Data Flow below is a high-level understanding of the current design.

![ESLint](ESLint.png)

- When rules are suppressed globally (external configuration files), ESLint does not do any tracking or run these suppressed rules at all, and suppression information will not be returned to output.
- When rules are suppressed in-source (`eslint-disable` inside of a file), ESLint will still run the rules but then remove any lint messages that were triggered with the specified rule(s).

### What we need

As mentioned above, what we need is to allow engineers to provide justification in the suppression and output the suppression information.

To minimize changes, we propose to use the `-- Description` part of the suppression as "Justification".

More specifically, for inSource suppression (or inline directive comments in ESLint), the description after two or more consecutive dashes can be treated as the justification.

For example:

```javascript
/* eslint-disable no-undef -- Justification example */
a = 1;
/* eslint-enable no-undef */
```
```javascript
console.log("foo"); // eslint-disable-line no-console -- Justification example
```
```javascript
// eslint-disable-next-line no-console -- Justification example
console.log("bar");
```

Then we may get:

```json
{"kind": "inSource", "justification": "Justification example"}
```

If a violation is inline-suppressed multiple times, all suppression entries will be recorded, according to [suppressions property](https://docs.oasis-open.org/sarif/sarif/v2.0/csprd02/sarif-v2.0-csprd02.html#_Toc10127852). So `unusedDisableDirectives` might always be empty.

For external suppression (or configuration files in ESLint), there are no good places to put a justification. We may put something general as a justification, like "Globally disabled".

For example:

```json
{"kind": "external", "justification": "Globally disabled"}
```

If a line is suppressed by multiple rules, a suppression list would be preferred to gather all suppression information:

```json
suppressions: [
  {"kind": "inSource", "justification": "Justification message 1"},
  {"kind": "inSource", "justification": "Justification message 2"},
  {"kind": "external", "justification": "Globally disabled"}
]
```

## Detailed Design

<!--
   This is the bulk of the RFC.

   Explain the design with enough detail that someone familiar with ESLint
   can implement it by reading this document. Please get into specifics
   of your approach, corner cases, and examples of how the change will be
   used. Be sure to define any new terms in this section.
-->

### Goal

1.	Keep the current behavior of ESLint, i.e., errors/warnings will not be shown on the command line if the related rules are disabled, unless `--no-inline-config` is used.
2.	When the new CLI option `--output-suppression` is used, the violations (problems) that disabled by inline comments will not be removed.
3.	Add suppression info in the output of ESLint
    1. For inSource suppressions, we generate suppression info where `kind` is `inSource` and `justification` is the description in the directive comments (words after two or more consecutive dashes mentioned above).
    
        For example:
        
        ```javascript
        /* eslint-disable no-undef -- Justification example */
        a = 1;
        /* eslint-enable no-undef */
        ```
        ```javascript
        console.log("foo"); // eslint-disable-line no-console -- Justification example
        ```
        ```javascript
        // eslint-disable-next-line no-console -- Justification example
        console.log("bar");
        ```

        Output:

        ```json
        {"kind": "inSource", "justification": "Justification example"}
        ```
    
    2. For external suppressions (or disabling rules inside of configuration files in ESLint), we generate suppression info where `kind` is `external` and `justification` is a certain sentence such as "Globally disabled".

        For example:

        ```json
        rules: {
          "no-undef": "off"
        }
        ```

        Output:

        ```json
        {"kind": "external", "justification": "Globally disabled"}
        ```

### General Design

We are proposing to add a new CLI option `--output-suppression` to output suppressions without prompting disabled errors/warnings.

![Design diagram](design_diagram.png)

Input:

```
.\node_modules\.bin\eslint.cmd 1.js --output-suppression -f @microsoft/eslint-formatter-sarif -o 1.sarif
```

Output:

```json
SARIF format
{
  ... Some other info ...
  =======================
  "results": [
    {
      "level": "error",
      "message": {
        "text": "'a' is not defined."
      },
      "locations": [
        {
          "physicalLocation": {
            "artifactLocation": {
              "uri": "file:///C:/Users/yiweiding/Documents/projects/test/1.js",
              "index": 0
            },
            "region": {
              "startLine": 1,
              "startColumn": 1
            }
          }
        }
      ],
      "suppressions": [
        {
          "kind": "inSource",
          "justification": "Sample justification message."
        }
      ],
      "ruleId": "no-undef",
      "ruleIndex": 0
    }
  ]
}
```

Input:

```
.\node_modules\.bin\eslint.cmd 1.js --output-suppression -f json -o 1.json
```

Output:

```json
JSON format
[
  {
    "filePath":"C:\\Users\\yiweiding\\Documents\\projects\\test\\1.js",
    "messages":[
      {
        "ruleId":"no-undef",
        "severity":2,
        "message":"'a' is not defined.",
        "line":1,
        "column":1,
        "nodeType":"Identifier",
        "suppressions":[
          {
            "kind":"inSource",
            "justification":"Sample justification message."
          }
        ],
        "messageId":"undef",
        "endLine":1,
        "endColumn":2
      }
    ],
    "errorCount":0,
    "warningCount":0,
    "fixableErrorCount":0,
    "fixableWarningCount":0,
    "usedDeprecatedRules":[]
  }
]
```

### Add new CLI option `--output-suppression`

We propose to add a new CLI option `--output-suppression` for the new feature. Without the option, ESLint will work as it is currently, and `suppressions` property should not exist in the output. With this option, ESLint is expected to export all violations, including suppressed ones. `suppressions` should be _empty_ for non-suppressed violations, while `suppressions` should be a list for suppressed ones, according to [suppressions property](https://docs.oasis-open.org/sarif/sarif/v2.0/csprd02/sarif-v2.0-csprd02.html#_Toc10127852).

### Parse description in comments as `justification`

We propose not to ignore the text after two or more dashes in comments and considered it as the justification of a suppression. The modification should be in `getDirectiveComments`.

### Add suppressions attribute in `LintMessage`

`LintMessage` is the type that finally output to the formatter. Currently it contains `ruleId`, `severity`, `message`, etc. What we desire is a `suppressions` attribute in `LintMessage` when the CLI option `--output-suppression` used. It should be a list that contains a suppression entry (`{kind: "inSource", justification: "Fake justification message."}`).

### Reserve external suppression

In the step of apply external suppression (or disabling rules inside of configuration files), ESLint will not skip rules disabled in configuration files if it gets the `--output-suppression` option. Violations (or problems in ESLint) will be created with `{kind: "external", justification: "Globally disabled"}` if the related rules are switched to `off`.

More specifically, when traversing `configuredRules` in `runRules`, ESLint will not [skip disabled rules](https://github.com/eslint/eslint/blob/81c60f4a8725738f191580646562d1dca7eee933/lib/linter/linter.js#L893) if the `--output-suppression` option used. Then the suppression info `{kind: "external", justification: "Globally disabled"}` will be put into [`createReportTranslator`](https://github.com/eslint/eslint/blob/81c60f4a8725738f191580646562d1dca7eee933/lib/linter/linter.js#L924) and [`createProblem`](https://github.com/eslint/eslint/blob/81c60f4a8725738f191580646562d1dca7eee933/lib/linter/report-translator.js#L357) to create reports with suppressions for disabled rules.

NOTE that rules disabled inside of configuration files could not be enabled by inline directive comments (e.g., `/* eslint-enable no-undef */`). But the disabling directive comments for disabled rules will still be recorded as an entry of `suppressions`, as described in [What we need](#What%20we%20need).

### Reserve inSource suppression

In the step of apply inSource suppression, ESLint will not remove the problems that should be disabled by directive comments if it gets the `--output-suppression` option. These problems will be reserved and added `{kind: "inSource", justification: "Justification message example."}`.

### Modify `CLIEngine` and related formatters

As mentioned, we are not going to change the current behavior of ESLint. Without `--no-inline-config`, ESLint should not report errors/warnings in CMD whether with `--output-suppression` option or not. In `CLIEngine`, when `suppressions` of a message is not null or empty (which means this message should be suppressed and not be reported), this message would be skipped or ignored. Additionally, the related formatters, such as stylish, should also be modified.

## Documentation

Add examples and explanations of the option `--output-suppression` to _User guide – Command Line Interface_.

## Drawbacks

- Additional contexts must be transferred across modules, such as `justification` in `directives` and `suppressions` in `LintMessage`.
- Related formatters, such as stylish, should be modified to keep the current behavior.

## Backwards Compatibility Analysis

Basically, current users would not feel any difference, since the proposed feature would only work with the new option. Without the option `--output-suppression`, any current behaviors would not be changed.

## Alternatives

No.

## Open Questions

Currently no.

## Help Needed

No.

## Frequently Asked Questions

TBD.

## Related Discussions

[eslint/eslint#14784](https://github.com/eslint/eslint/issues/14784) Add suppression information in the output of ESLint
