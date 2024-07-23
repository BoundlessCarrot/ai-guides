# Guide to Zig's Testing Module

Zig's standard library provides a robust testing module that allows you to write and run tests easily. This guide will cover the basics of using the testing module and how to avoid common pitfalls, such as incompatible type errors when comparing strings and arrays.

## Basic Test Structure

To create a test in Zig, use the `test` keyword followed by a string describing the test:

```zig
test "my first test" {
    // Test code goes here
}
```

## Using Assertions

Zig's testing module provides several assertion functions:

1. `try std.testing.expect(bool)`: Asserts that a boolean condition is true.
2. `try std.testing.expectEqual(expected, actual)`: Asserts that two values are equal.
3. `try std.testing.expectEqualSlices(T, expected, actual)`: Asserts that two slices are equal.
4. `try std.testing.expectEqualStrings(expected, actual)`: Asserts that two strings are equal.

### Example:

```zig
const std = @import("std");

test "basic assertions" {
    try std.testing.expect(true);
    try std.testing.expectEqual(5, 2 + 3);
}
```

## Avoiding Incompatible Type Errors

One common issue is getting incompatible type errors when comparing strings or arrays. This often happens because Zig treats string literals differently from arrays or slices.

### Incorrect:

```zig
test "incorrect string comparison" {
    const result = "hello";
    try std.testing.expectEqual("hello", result); // Error: incompatible types
}
```

### Correct:

```zig
test "correct string comparison" {
    const result = "hello";
    try std.testing.expectEqualStrings("hello", result);
}
```

For arrays or slices, use `expectEqualSlices`:

```zig
test "array comparison" {
    const expected = [_]u8{ 1, 2, 3 };
    const result = [_]u8{ 1, 2, 3 };
    try std.testing.expectEqualSlices(u8, &expected, &result);
}
```

## Working with Allocators

When testing functions that use allocators, it's common to use a test allocator:

```zig
test "allocator test" {
    var arena = std.heap.ArenaAllocator.init(std.testing.allocator);
    defer arena.deinit();
    const allocator = arena.allocator();

    // Use allocator in your test...
}
```

## Error Handling in Tests

Zig's `try` keyword is used to propagate errors in tests. If an error occurs, the test will fail:

```zig
fn mayFail() !void {
    return error.SomeError;
}

test "error handling" {
    try std.testing.expectError(error.SomeError, mayFail());
}
```

## Running Tests

To run tests, use the `zig test` command:

```
zig test your_file.zig
```

You can also use `zig build test` if you're using Zig's build system.

### Command Line Options for `zig test`

The `zig test` command provides several useful options to control test execution and output:

1. `-h, --help`: Display help information for the `zig test` command.

2. `--test-filter [text]`: Only run tests that match the filter text. This is case-sensitive.
   ```
   zig test your_file.zig --test-filter "array comparison"
   ```

3. `--test-name-prefix [text]`: Add a prefix to all test names. This can be useful for organizing test output.
   ```
   zig test your_file.zig --test-name-prefix "Module1:"
   ```

4. `--test-cmd [cmd]`: Run an external command for each test instead of building an executable.

5. `--test-cmd-bin`: Appends the test executable filename to the test command.

6. `--test-no-exec`: Compiles the test executable but does not run it.

7. `-O [mode], --optimize [mode]`: Set the optimization mode (debug, release-safe, release-fast, release-small).
   ```
   zig test your_file.zig -O ReleaseFast
   ```

8. `--main-pkg-path [path]`: Set the directory of the root package.

9. `--cache-dir [path]`: Override the cache directory.

10. `--global-cache-dir [path]`: Override the global cache directory.

11. `--zig-lib-dir [path]`: Override path to Zig lib directory.

12. `-fno-stage1`: Disable stage1 compiler.

13. `--listen=-`: Listens on stdin for commands to control the test runner.

These options allow you to fine-tune your test execution, from filtering specific tests to run, to controlling optimization levels, to integrating with external tools.

## Conclusion 1

By using the appropriate assertion functions, understanding Zig's type system, and leveraging the command line options of `zig test`, you can write clear, effective tests and avoid common pitfalls like incompatible type errors. Remember to use `expectEqualStrings` for string comparisons and `expectEqualSlices` for array or slice comparisons. The command line options provide powerful tools for controlling test execution and integrating testing into your development workflow.


## Advanced Example: Handling Out-of-Bounds Array Access

Let's examine a more complex example involving 2D arrays and out-of-bounds access handling. This example demonstrates common pitfalls and how to correct them.

### Incorrect Implementation

```zig
fn collectNeighbors(board: [][]u8, x: usize, y: usize) ![]u8 {
    const neighbors = [8]u8{
        board[x - 1][y - 1] catch 0,
        board[x][y - 1] catch 0,
        board[x + 1][y - 1] catch 0,
        board[x - 1][y] catch 0,
        board[x + 1][y] catch 0,
        board[x - 1][y + 1] catch 0,
        board[x][y + 1] catch 0,
        board[x + 1][y + 1] catch 0,
    };

    return neighbors;
}

test "collectNeighbors OOB case" {
    const board = try allocateMatrix(u8, std.testing.allocator, 10, 10, 0);
    defer freeMatrix(board, std.testing.allocator);

    board[0][1] = 1;
    board[1][0] = 1;
    board[1][1] = 1;

    const expected: [8]u8 = .{ 0, 0, 0, 0, 1, 0, 1, 1 };
    const neighbors: []u8 = try collectNeighbors(board, 0, 0);
    try std.testing.expectEqualSlices(u8, &expected, &neighbors);
}
```

### Issues with the Code

1. Array indexing in Zig doesn't return an error union, so `catch` can't be used directly.
2. Subtracting from an unsigned integer (usize) can cause underflow.
3. The function returns a slice (`[]u8`) but is creating a fixed-size array.
4. The test is trying to assign a slice to a fixed-size array.

### Corrected Implementation

Here's a corrected version of the code:

```zig
const std = @import("std");

fn collectNeighbors(board: []const []const u8, x: usize, y: usize) [8]u8 {
    const height = board.len;
    const width = if (height > 0) board[0].len else 0;

    return [8]u8{
        if (x > 0 and y > 0) board[x - 1][y - 1] else 0,
        if (y > 0) board[x][y - 1] else 0,
        if (x + 1 < width and y > 0) board[x + 1][y - 1] else 0,
        if (x > 0) board[x - 1][y] else 0,
        if (x + 1 < width) board[x + 1][y] else 0,
        if (x > 0 and y + 1 < height) board[x - 1][y + 1] else 0,
        if (y + 1 < height) board[x][y + 1] else 0,
        if (x + 1 < width and y + 1 < height) board[x + 1][y + 1] else 0,
    };
}

fn allocateMatrix(comptime T: type, allocator: std.mem.Allocator, rows: usize, cols: usize, init: T) ![][]T {
    var matrix = try allocator.alloc([]T, rows);
    errdefer allocator.free(matrix);

    for (matrix) |*row| {
        row.* = try allocator.alloc(T, cols);
        @memset(row.*, init);
    }

    return matrix;
}

fn freeMatrix(matrix: anytype, allocator: std.mem.Allocator) void {
    for (matrix) |row| {
        allocator.free(row);
    }
    allocator.free(matrix);
}

test "collectNeighbors OOB case" {
    const allocator = std.testing.allocator;
    var board = try allocateMatrix(u8, allocator, 10, 10, 0);
    defer freeMatrix(board, allocator);

    board[0][1] = 1;
    board[1][0] = 1;
    board[1][1] = 1;

    const expected = [8]u8{ 0, 0, 0, 0, 1, 0, 1, 1 };
    const neighbors = collectNeighbors(board, 0, 0);
    try std.testing.expectEqual(expected, neighbors);
}
```

### Key Changes and Explanations

1. Changed `collectNeighbors` to return `[8]u8` instead of `[]u8`. This matches the fixed-size array we're creating.
2. Removed `try` and `catch` from `collectNeighbors`. Instead, we use conditional (ternary) operators to check bounds.
3. Added bounds checking for all array accesses to prevent out-of-bounds errors.
4. In the test, changed `const neighbors: []u8` to `const neighbors` to let Zig infer the correct type.
5. Used `expectEqual` instead of `expectEqualSlices` since we're comparing fixed-size arrays.
6. Made `board` in `collectNeighbors` `const` to ensure we're not modifying it.

This corrected version safely handles out-of-bounds cases and correctly implements the neighbor collection logic.

## Conclusion

This example demonstrates several important concepts in Zig:
- Proper handling of potential out-of-bounds array access
- Working with 2D arrays (slices of slices)
- Correct use of fixed-size arrays vs. slices
- Proper memory management with allocators
- Correct usage of testing functions

Remember to always consider edge cases, especially when working with arrays and indices, and to leverage Zig's type system to catch errors at compile-time when possible.
