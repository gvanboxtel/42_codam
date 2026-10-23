*This project has been created as part of the 42 curriculum by gvan-box.*

# get_next_line

## Description

get_next_line is a C function that returns one line at a time from a file descriptor:

```c
char *get_next_line(int fd);
```

Each call returns the next line, including its `\n` (except at the end of a file without a trailing newline), or `NULL` when there is nothing left to read or an error occurs. The caller frees the returned line.

The function reads with `read()` in chunks of `BUFFER_SIZE` bytes, so a chunk can end in the middle of a line or contain several lines. The core lesson of the project is **static variables**: whatever was read past the end of the current line has to survive until the next call.

The bonus version reads from **several file descriptors at the same time** without mixing up their content, using only one static variable.

## Instructions

Compile `get_next_line` together with your own `main`, choosing any buffer size:

```bash
cc -Wall -Wextra -Werror -D BUFFER_SIZE=42 \
    get_next_line.c get_next_line_utils.c main.c

# Bonus: multiple file descriptors
cc -Wall -Wextra -Werror -D BUFFER_SIZE=42 \
    get_next_line_bonus.c get_next_line_utils_bonus.c main.c
```

Example usage:

```c
int   fd = open("file.txt", O_RDONLY);
char *line;

while ((line = get_next_line(fd)) != NULL)
{
    printf("%s", line);
    free(line);
}
close(fd);
```

`BUFFER_SIZE` defaults to 100 when it is not given at compile time.

## Algorithm

1. Start the new line from the **remainder** left over from the previous call, if there is one.
2. Keep calling `read()` and appending the buffer to the line until it contains a `\n` or `read()` returns 0 (end of file).
3. Split at the first `\n`: everything up to and including it is returned, everything after it is stored as the new remainder.
4. On a read error, free both the line and the remainder and return `NULL`, so no memory leaks and the next call starts clean.

For the bonus, the remainder is a static array indexed by file descriptor (`static char *remainder[1024]`), so every open file keeps its own leftover text.

## Testing

The project was tested against a set of edge-case files: an empty file, a file without a final newline, many empty lines, one-character and one-word lines, a single very long line (with and without a final newline), lines around the buffer size, and a read error.

## Resources

- `man 2 read`, `man 2 open`.
- The 42 get_next_line subject.

### AI usage

This README was written with Claude (Anthropic), based on the project's source code.
