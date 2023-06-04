- Repo: eslint/eslint
- Start Date: 2023-06-01
- RFC PR:
- Authors: [Balavishnu V J](@balavishnuvj)

# Sharding

## Summary

This RFC proposes adding sharding support to the ESLint package. Sharding is a technique used to divide a large workload into smaller, more manageable parts that can be processed concurrently. By introducing sharding in ESLint, we aim to improve linting performance for large codebases and reduce the overall execution time of linting tasks.

## Motivation

The motivation behind this feature is to address the performance limitations of ESLint when dealing with large codebases. As codebases grow in size, linting the entire codebase in a single execution can become time-consuming and impact developer productivity.

By introducing sharding, ESLint can distribute the linting workload across multiple processes or threads, allowing for parallel execution and faster linting times. It can also improve integration with development tools and CI/CD pipelines, as linting tasks can be completed faster, leading to quicker feedback cycles.
Detailed Design

The implementation of sharding in the ESLint package involves the following steps:

- **Configuration**: Introduce a new configuration option or flag in the ESLint configuration file to enable sharding. This option can specify the number of shards or the desired granularity of the shards.

  ```
  eslint --shard=1/3
  ```

- **Workload Division**: Analyze the input codebase and divide it into smaller units of work called shards. The division can be based on files, directories, or any other logical grouping that allows for parallel processing. Each shard should be self-contained and independent of the others. For example, larger files can be distributed across shards to balance the workload. We could also expose api to get custom division statergy.

- **Parallel Execution**: Launch multiple processes or threads to handle each shard concurrently. Each process/thread should execute the linting process independently on its assigned shard.

- **Results Aggregation**: Once all shards have completed linting, aggregate the results from each process/thread into a unified report. This report should provide a consolidated view of the linting issues found across all shards.

- **Error Handling**: Implement appropriate error handling mechanisms to handle failures in individual shards. If a shard encounters an error during linting, it should be reported as part of the aggregated results, along with any relevant error information.

## Detailed Design

1. Get the files in a deterministic order.
   in `lib/cli.js`, we can get all the files. Update the `executeOnFiles` function

   ```js
   const allFiles = [];
   for (const { config, filePath, ignored } of fileEnumerator.iterateFiles(
     patterns
   )) {
     /* ... */
     allFiles.push({ config, filePath, ignored });
   }
   ```

2. Divide the files into shards. The number of shards is determined by the `--shard` option.

   ```js
   const sortedFiles = allFiles.sort((a, b) =>
     a.filePath.localeCompare(b.filePath)
   );
   ```

3. Lint these sorted files

   ```js
   const shard = parseInt(cliOptions.shard);
   const totalShards = parseInt(cliOptions.totalShards);
   const shardFiles = allFiles.filter(
     (_, index) => index % totalShards === shard - 1
   );

   shardFiles.forEach(({ filePath, config }) => {
     const result = verifyText({
       text: fs.readFileSync(filePath, "utf8"),
       filePath,
       config,
       cwd,
       fix,
       allowInlineConfig,
       reportUnusedDisableDirectives,
       fileEnumerator,
       linter,
     });
     results.push(result);
   });
   ```

## Documentation

The addition of sharding support should be documented in the ESLint user documentation. The documentation should cover the configuration options, explain the benefits and considerations when using sharding, and provide examples of how to set up and utilize sharding effectively. Additionally, a formal announcement on the ESLint blog can be made to explain the motivation behind introducing sharding and its impact on linting performance.

## Drawbacks

Introducing sharding in the ESLint package has several potential drawbacks to consider:

- **Increased Complexity**: Sharding adds complexity to the linting process, requiring additional logic for workload division, parallel execution, and result aggregation. This complexity may make the codebase harder to maintain and debug.

- **Resource Consumption**: Sharding involves launching multiple processes/threads to handle each shard concurrently. This can result in increased resource consumption, such as CPU and memory usage, particularly when dealing with a large number of shards or codebases.

- **Potential Overhead**: The overhead of shard coordination and result aggregation may impact the overall performance gain achieved through parallel execution. Careful optimization and benchmarking should be performed to ensure that the benefits of sharding outweigh the associated overhead.

## Backwards Compatibility Analysis

The introduction of sharding in the ESLint package should not affect existing ESLint users by default. Sharding should be an opt-in feature, enabled through configuration options or flags. Users who do not wish to utilize sharding can continue to use ESLint as before without any disruption.

## Alternatives

Alternative approaches to improve linting performance in ESLint include:

Manually splitting the project to run in parallel example by @bmish in https://github.com/eslint/eslint/issues/16726#issuecomment-1368059618.

```json
{
  "scripts": {
    "lint:js": "npm-run-all --aggregate-output --continue-on-error --parallel \"lint:js:*\"",
    "lint:js:dir1": "eslint --cache dir1/",
    "lint:js:dir2": "eslint --cache dir2/",
    "lint:js:dir3": "eslint --cache dir3/",
    "lint:js:other": "eslint --cache --ignore-pattern dir1/ --ignore-pattern dir2/ --ignore-pattern dir3/ ."
  }
}
```

## Open Questions

- What process/thread management strategy should be employed to handle the shards effectively?
- How to allow custom division statergy?

## Help Needed

As a first-time contributor, I would greatly appreciate guidance and assistance from the ESLint core team and the community in implementing the sharding feature. While I have experience as a programmer, contributing to a large-scale project like ESLint involves understanding the codebase, adhering to coding standards, and following the established contribution process.

## Frequently Asked Questions

Q: Will enabling sharding affect the linting results or introduce any inconsistencies?

A: Enabling sharding should not affect the linting results or introduce any inconsistencies. Sharding is designed to distribute the workload and execute the linting process independently on each shard. The aggregated results should provide a unified view of the linting issues across all shards, ensuring consistent analysis.

Q: Can sharding be used with ESLint plugins and custom rules?

A: Sharding should be compatible with ESLint plugins and custom rules. However, it's important to ensure that the plugins and rules are designed to work correctly in a sharded environment. Compatibility testing and coordination with plugin developers may be necessary to ensure smooth integration.

## Related Discussions

[Change Request: add ability to shard eslint](https://github.com/eslint/eslint/issues/16726)
