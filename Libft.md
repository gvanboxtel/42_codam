*This project has been created as part of the 42 curriculum by gvan-box.*

# Libft

## Description

Libft is the first project of the 42 Common Core: a static C library (`libft.a`) that re-implements a selection of standard C library functions, plus extra string, memory and linked-list utilities. It is reused in later projects (ft_printf, get_next_line, push_swap), so it has to be correct, leak-free and robust against edge cases.

Everything is written in C under the 42 Norm, compiled with `-Wall -Wextra -Werror`, without global variables, and only `malloc`, `free` and `write` from the system.

### Part 1: libc functions

| Category | Functions |
|---|---|
| Character checks | `ft_isalpha`, `ft_isdigit`, `ft_isalnum`, `ft_isascii`, `ft_isprint`, `ft_toupper`, `ft_tolower` |
| Strings | `ft_strlen`, `ft_strlcpy`, `ft_strlcat`, `ft_strchr`, `ft_strrchr`, `ft_strncmp`, `ft_strnstr`, `ft_strdup`, `ft_atoi` |
| Memory | `ft_memset`, `ft_bzero`, `ft_memcpy`, `ft_memmove`, `ft_memchr`, `ft_memcmp`, `ft_calloc` |

### Part 2: additional functions

| Function | Purpose |
|---|---|
| `ft_substr`, `ft_strjoin`, `ft_strtrim` | Build new strings from existing ones |
| `ft_split` | Split a string on a delimiter into a `NULL`-terminated array |
| `ft_itoa` | Convert an `int` to a string (including `INT_MIN`) |
| `ft_strmapi`, `ft_striteri` | Apply a function to every character |
| `ft_putchar_fd`, `ft_putstr_fd`, `ft_putendl_fd`, `ft_putnbr_fd` | Write to a file descriptor |

### Bonus: linked lists

A singly linked list (`t_list`) with `ft_lstnew`, `ft_lstadd_front`, `ft_lstadd_back`, `ft_lstsize`, `ft_lstlast`, `ft_lstdelone`, `ft_lstclear`, `ft_lstiter` and `ft_lstmap`.

## Instructions

```bash
make          # build libft.a
make bonus    # add the linked-list functions
make clean    # remove object files
make fclean   # remove object files and libft.a
```

Link against it from another project:

```bash
cc main.c -L. -lft -I.
```

## Technical notes

- `ft_memmove` copies backwards when the destination overlaps the end of the source, so overlapping copies are safe.
- `ft_split` frees everything it allocated if any allocation fails, so a failed split never leaks.
- `ft_calloc` checks for `nmemb * size` overflow before allocating.

## Resources

- `man 3` pages of the original libc functions.
- The 42 Libft subject and the 42 Norm.

### AI usage

This README was written with Claude (Anthropic), based on the project's source code.
