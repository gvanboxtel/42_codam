*This project has been created as part of the 42 curriculum by gvan-box.*

# ft_printf

## Description

ft_printf is a re-implementation of the libc `printf` function in C, packaged as a static library (`libftprintf.a`). The main lesson is variadic functions: reading an unknown number of arguments of different types with `va_start`, `va_arg` and `va_end`.

`ft_printf` writes to standard output and returns the number of characters printed, like the original.

### Supported conversions

| Conversion | Output |
|---|---|
| `%c` | A single character |
| `%s` | A string |
| `%p` | A pointer address in hexadecimal, prefixed with `0x` |
| `%d`, `%i` | A signed decimal integer |
| `%u` | An unsigned decimal integer |
| `%x`, `%X` | A hexadecimal number in lowercase / uppercase |
| `%%` | A literal percent sign |

## Instructions

```bash
make          # build libftprintf.a (builds the bundled libft first)
make clean    # remove object files
make fclean   # remove object files and the library
```

Use it in a program:

```c
#include "ft_printf.h"

int main(void)
{
    int n = ft_printf("%s is %d years old, at %p\n", "Libft", 1, &n);
    ft_printf("printed %u characters (%x in hex)\n", n, n);
    return (0);
}
```

```bash
cc main.c -L. -lftprintf
```

## Technical choices

- **Dispatch on the specifier:** the format string is walked character by character; on a `%` the next character selects the conversion and the matching `va_arg` type is read.
- **Reuse of Libft:** characters, strings and signed numbers go through the `ft_put*_fd` functions from Libft, which return how many bytes they wrote so the total count can be kept.
- **Recursive hex conversion:** hexadecimal output (`%x`, `%X`, `%p`) is produced by a recursive function over a digit table, so no buffer is needed. Pointers are handled as `uintptr_t` so the full address width is printed.

## Resources

- `man 3 printf` and `man 3 stdarg`.
- The 42 ft_printf subject.

### AI usage

This README was written with Claude (Anthropic), based on the project's source code.
