*This project has been created as part of the 42 curriculum by gvan-box.*

# Call Me Maybe

## Description

Call Me Maybe turns natural-language requests into **structured function calls** using a small language model (Qwen3-0.6B). Given a list of available functions and a prompt such as *"What is the product of 3 and 5?"*, the program picks the right function and fills in its arguments with the correct types:

```json
{
  "prompt": "What is the product of 3 and 5?",
  "name": "fn_multiply_numbers",
  "parameters": { "a": 3.0, "b": 5.0 }
}
```

A 0.6B-parameter model is far too small to reliably produce valid JSON on its own. Instead of hoping the model formats its answer correctly, this project works at the **token level**: it reads the model's raw logits and restricts what the model is allowed to generate. The output is therefore always a valid function name with arguments of the right type.

## Instructions

### Requirements

- Python 3.10+
- [uv](https://docs.astral.sh/uv/)

### Run

```bash
make install   # uv sync
make run       # process data/input/function_calling_tests.json
make lint      # flake8 + mypy
```

Or directly, with custom input and output files:

```bash
uv run python3 -m src \
    --functions_definition data/input/functions_definition.json \
    --input data/input/function_calling_tests.json \
    --output data/output/function_calls.json
```

### Input format

Function definitions describe the name, a description, and typed parameters and return value:

```json
{
  "name": "fn_is_even",
  "description": "Check if an integer is even, returns True if even, False if odd.",
  "parameters": { "n": { "type": "integer" } },
  "returns": { "type": "boolean" }
}
```

Supported types are `number`, `integer`, `boolean` and `string`. Prompts are a JSON list of `{"prompt": "..."}` objects.

## How it works

### 1. Function selection by likelihood scoring

The model is shown the list of functions and the user prompt, followed by `Function:`. For each candidate, the program computes the log-probability that the model would produce exactly that function's name:

```
log P(name | prompt) = Σ log P(token_i | prompt, token_1 … token_i-1)
```

Log-probabilities come from the logits with a numerically stable log-softmax (log-sum-exp). The function with the highest score is chosen. Because only real function names are scored, the model can never invent a function that does not exist.

### 2. Constrained decoding of the arguments

A partial JSON object is built up to `"parameters": {`, and each argument is generated token by token:

- the parameter key (`"a": `) is written by the program, not the model;
- at each step, all tokens that do not fit the parameter's type are masked to `-inf`, and the most likely remaining token is taken (greedy decoding);
  - `number` / `integer`: only digit and `.` tokens;
  - `boolean`: only the tokens of `true` / `false`;
  - `string`: any token, with the program adding the opening and closing quotes;
- generation of a value stops as soon as the model's unconstrained choice would be a `,`, `}` or closing quote, which means the model "wants" to end the value.

The raw text is then cast to the right Python type and validated.

### 3. Validation

Inputs and outputs are Pydantic models (`FunctionDefinition`, `FunctionCall`). Function names must start with `fn_`, and parameter types are restricted to the supported set, so malformed definition files are rejected with a clear error before the model is loaded.

## Architecture

| Module | Responsibility |
|---|---|
| `__main__.py` | CLI arguments and error reporting |
| `io.py` | Loading and validating the function definitions and prompts |
| `models.py` | Pydantic schemas |
| `decoder.py` | Function scoring and constrained decoding on top of the model |
| `coordinator.py` | Runs the decoder over all prompts |
| `writer.py` | Writes the results to JSON |

The model itself is accessed through the provided `llm_sdk` package, which exposes tokenisation and raw logits.

## Resources

- [Qwen3-0.6B model card](https://huggingface.co/Qwen/Qwen3-0.6B)
- [Pydantic documentation](https://docs.pydantic.dev/)
- Willard & Louf (2023), [Efficient Guided Generation for Large Language Models](https://arxiv.org/abs/2307.09702): the idea behind constrained decoding with token masks.
- [The log-sum-exp trick](https://gregorygundersen.com/blog/2020/02/09/log-sum-exp/)

### AI usage

This README was written with Claude (Anthropic), based on the project's source code.
