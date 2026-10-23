_This project has been created as part of the 42 curriculum by gvan-box._
# Push_swap

## Description

**Push_swap** is an algorithm project that challenges students to sort data on a stack, using a limited set of instructions, and with the smallest number of moves possible.

The goal is to sort a random list of integers using two stacks (**Stack A** and **Stack B**) and a set of specific operations. To succeed, the program must calculate the most "cost-effective" moves to move elements between stacks until Stack A is perfectly sorted. This implementation uses a **Chunking Strategy** combined with a **Best Cost Reinsertion Strategy** to minimize the total number of operations.

### Key Features

* **Rank Mapping:** Pre-processes input values by mapping them to their relative ranks ( to ) to simplify comparison logic.
* **Chunking:** Divides the data into logical blocks based on total input size (e.g., 5 chunks for 500 numbers) to maintain control over the stacks.
* **Cost Optimization:** For every element in Stack B, the algorithm calculates the number of rotations required for both stacks to place it in its correct position in Stack A, picking the cheapest option.

---

## Instructions

### Compilation

The project includes a `Makefile` that compiles the source files into an executable named `push_swap`.

```bash
# To compile the project
make

```

### Execution

Run the program by providing a list of integers as arguments. The integers can be passed as separate arguments or as a single quoted string.

```bash
# Example with individual arguments
./push_swap 42 67 12 0

# Example with a string
./push_swap "3 1 2"

# Example with N random numbers
N=100; ARG=$(shuf -i 0-$N -n $N | tr '\n' ' '); ./push_swap $ARG

```

The program will output a list of instructions (e.g., `pb`, `ra`, `rrr`, `pa`) that sort the stack.
If the user would like to know the operation count:

```bash

./push_swap "3 1 2" | wc -l

```

```bash

# Check for correct result
N=100; ARG=$(shuf -i 0-$N -n $N | tr '\n' ' '); ./push_swap $ARG | ./checker_linux $ARG

```


### Operations

* `sa`, `sb`, `ss`: Swap the first two elements of a stack.
* `pa`, `pb`: Push the top element from one stack to another.
* `ra`, `rb`, `rr`: Rotate the stack (shift all elements up by one).
* `rra`, `rrb`, `rrr`: Reverse rotate the stack (shift all elements down by one).

---

## Technical Choices

* **Doubly Linked Lists:** Used for the stack implementation to allow efficient movement and access to both the `next` and `prev` nodes.
* **Map to Rank:** First, the numbers given are pre-sorted and given a rank. In this manner it is already known where the numbers need to end up.
* **Chunking Logic:** Second, based on the amount of numbers entered to be push_swapped, the numbers are sorted into their respective chunks based on their rank, and pushed to B in that order until 3 elements remain in stack A. These elements are sorted with `sort_3`
* **Targeting Logic:** Thid, to calculate the cheapest element to be pushed backed into stack a, the function `get_target_in_a` identifies the smallest number in Stack A that is still larger than the element being pushed from Stack B, ensuring the ascending order is maintained.
* **Final:** Finally, the best cost is executed and push to stack A. This is done for every element in B, until it is empty.

---

### Bonus

For the bonus the user is required to write their own checker program. This is done by piping the operations to the checker program. It is done as follows:

```bash

# Check for correct result
./push_swap "3 1 2" | ./checker "3 1 2"

```

If the result is correct, the checker will return `OK`, otherwise `KO`.


## Resources

### Documentation & Articles

* **42 Project Instructions:** The primary source for logic constraints.
* **The "Mechanical Turk" Algorithm:** Inspired by common 42 community approaches for optimizing push_swap cost calculations.
* **Sorting Algorithms:** General research on QuickSort and MergeSort logic to understand partitioning.

### AI Usage Disclosure

In compliance with the 42 curriculum requirements, AI (Gemini 3 Flash) was used in the following capacity:

* **Task:** Help with thinking about the general logic of the code.
* **Specifics:** AI was used to generate this `README.md` based on the existing source code.