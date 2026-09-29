# Call Me Maybe — Project Documentation

## 1. Project Overview

Call Me Maybe converts a natural-language request into a structured function call. For example, when asked to add two numbers, it identifies `fn_add_numbers` and produces its arguments. It returns the call description; it does **not** run the function or calculate the sum.

The project uses the supplied `llm_sdk` and its default Qwen3-0.6B model. Function names, descriptions, and parameter types come from `data/input/functions_definition.json`. Requests come from `data/input/function_calling_tests.json`. Generated calls are written to a JSON file.

The main goal is to use constrained decoding so the model can select only known function names and generate parameter values according to their declared types.

## 2. Project Structure

```text
.
├── data/
│   ├── input/
│   │   ├── function_calling_tests.json
│   │   └── functions_definition.json
│   └── output/                         created when the program runs
├── llm_sdk/                            supplied local model SDK
├── moulinette/                         supplementary exercise/evaluation tool
├── sources/                            original assignment PDF and archives
├── inspiration/                        reference project; not imported at runtime
├── src/
│   ├── __init__.py
│   ├── __main__.py                     command-line entry point
│   ├── decoder.py                      model prompt and constrained generation
│   ├── models.py                       Pydantic input models
│   ├── parser.py                       JSON loading and validation
│   ├── prompt_builder.py                model prompt construction
│   └── runner.py                       processes prompts one by one
├── Makefile                            install, run, debug, lint, and clean commands
├── pyproject.toml                      project and dependency configuration
├── uv.lock                             locked dependency versions
├── README.md                           project summary and quick-start guide
└── DOCUMENTATION.md                    this detailed explanation
```

The `moulinette/` directory contains a separate tool for preparing exercises and evaluating answers. The `src/` application does not import or run it. The `inspiration/` directory contains the reference project; it is also not imported by the application.

The two input JSON files contain five function definitions and eleven natural-language prompts. The supplied functions are:

| Function | Parameters |
| --- | --- |
| `fn_add_numbers` | `a`, `b` (`number`) |
| `fn_greet` | `name` (`string`) |
| `fn_reverse_string` | `s` (`string`) |
| `fn_get_square_root` | `a` (`number`) |
| `fn_substitute_string_with_regex` | `source_string`, `regex`, `replacement` (`string`) |

## 3. Execution Flow

When run as `python -m src` or with `make run`, the program proceeds as follows:

1. **Read command-line options.** `src/__main__.py` accepts paths for function definitions, prompts, and output. If omitted, it uses the default paths under `data/`.
2. **Load and validate both input files.** `parser.py` reads UTF-8 JSON and validates each list entry with the Pydantic models. Unreadable, empty, malformed, incorrectly shaped, or empty-list input is reported to standard error and ends the run with status 1.
3. **Initialize the model.** The program constructs `Small_LLM_Model()` from the supplied SDK. Its default model is Qwen3-0.6B. The SDK chooses an available device and downloads model/tokenizer files as needed. Initialization errors are reported and end the run with status 1.
4. **Create the function-call generator.** `FunctionCallGenerator` reads the model vocabulary and prepares it for token filtering.
5. **Process each prompt in order.** The prompt builder lists available functions and their parameter types. The decoder selects a valid function name and generates values for its parameters.
6. **Handle per-prompt errors.** If generation for one prompt raises an exception, the runner reports it and records that prompt with an empty name and parameter object. It continues with the remaining prompts.
7. **Write the result list.** The entry point creates the output directory if necessary and serializes all result dictionaries as indented JSON. A file-writing error is reported and ends the run with status 1; otherwise, the program prints the output path and exits successfully.

## 4. File-by-File Explanation

### `src/__main__.py`

This is the application entry point. `main()` defines the command-line options, calls the input loaders, initializes the SDK model and `FunctionCallGenerator`, delegates prompt processing to the runner, and writes the results. `argparse` supplies help text and defaults. `Path` creates the destination directory, while `json.dumps()` creates the output JSON.

The options are:

- `--functions_definition`: defaults to `data/input/functions_definition.json`.
- `--input`: defaults to `data/input/function_calling_tests.json`.
- `--output`: defaults to `data/output/function_calling_results.json`.

### `src/models.py`

These Pydantic models describe the accepted input shapes:

- `ParameterDefinition` restricts parameter types to `string`, `number`, `integer`, or `boolean`.
- `ReturnDefinition` holds the declared return type. It is loaded from the definitions but is not used to generate or execute a function result.
- `FunctionDefinition` holds a function's name, description, parameter map, and return definition.
- `Prompt` holds one input prompt string.

Each model forbids extra fields so unexpected keys are rejected during validation.

### `src/parser.py`

- `_read_contents()` reads a UTF-8 file, converts file-system errors to a clear `ValueError`, and rejects a blank file.
- `_parse_list()` uses a `TypeAdapter` to validate a JSON array and rejects an array with no entries.
- `load_function_definitions()` and `load_prompts()` apply these shared steps with the appropriate model type.

### `src/prompt_builder.py`

`build_prompt()` formats the user request with a textual catalogue of function names, descriptions, and parameter types. It tells the model to choose a suitable function and identify its values. The decoder adds the output-structure prefix after this text.

### `src/decoder.py`

`FunctionCallGenerator` owns the model, function definitions, and vocabulary map. During initialization, it reads the vocabulary file path supplied by the SDK and normalizes the `Ġ` space and `Ċ` newline markers.

- `_encode()` uses the SDK's tokenizer to turn text into token IDs.
- `_choose_token()` reads next-token logits, masks out IDs that are not allowed, and selects the remaining token with the highest score (greedy decoding).
- `_generate_function_name()` allows only tokens that keep the current text a prefix of a supplied function name. It stops once the generated text matches a known name.
- `_generate_number()` permits digits, a leading minus sign, and—only for `number`—at most one decimal point. It parses the completed text as an `int` or `float`. Generation is limited to 20 token steps.
- `_generate_boolean()` keeps tokens that can form `true` or `false`, and is limited to 5 steps.
- `_generate_string()` takes the model's highest-scoring token while it is printable, stopping when the token contains a double quote. It is limited to 50 steps and trims whitespace around the result.
- `generate()` builds the prompt context, selects the function, generates parameter values in the definition's parameter order, and returns a dictionary containing `prompt`, `name`, and `parameters`.

### `src/runner.py`

`process_prompts()` calls `generate()` once for each validated `Prompt`. It collects successful results and catches generation errors per prompt so later prompts still run.

### Project configuration and supplied SDK

- `pyproject.toml` declares runtime and development dependencies and points `llm-sdk` to the local `llm_sdk/` directory.
- `uv.lock` records resolved dependency versions for `uv`.
- `Makefile` provides the project's common commands.
- `llm_sdk/llm_sdk/__init__.py` defines `Small_LLM_Model`, which loads the tokenizer and language model, encodes text, returns next-token logits, and provides the vocabulary file path.

## 5. Data Flow

```text
function_calling_tests.json ──> load_prompts() ──> Prompt objects ──┐
                                                                    ├─> process_prompts()
functions_definition.json ───> load_function_definitions() ────────┘         │
                                      │                                      │
                                      └─> prompt catalogue ─> model prompt   │
                                                       │                     │
                                                       v                     v
                                            constrained decoding ──> result dictionaries
                                                                         │
                                                                         v
                                                    function_calling_results.json
```

The SDK encodes the text prompt as token IDs. At each generation step, the decoder obtains logits for the next token and restricts the choices where a type or function-name constraint applies. The chosen token is appended to the context for the next step. The program assembles generated values into Python dictionaries; `json.dumps()` then serializes those dictionaries to the output file.

## 6. Algorithms and Logic

### Function selection

The decoder checks each vocabulary token against the set of known names. A token is eligible only if adding its text leaves a prefix of at least one known function name. Among eligible tokens, the decoder chooses the one with the highest model logit. This keeps the name within the supplied list. The current function names do not require the decoder to resolve a shared complete-name prefix.

### Parameter generation

The chosen function definition determines the generator for each parameter:

- `number`: digits, one optional leading minus, and at most one decimal point; parsed as a float.
- `integer`: digits and one optional leading minus, with no decimal point; parsed as an integer.
- `boolean`: tokens forming exactly `true` or `false`.
- `string`: printable tokens until a token contains a closing double quote; leading and trailing whitespace is stripped.

The program inserts JSON keys and separators into the model context itself. It does not ask the model to create the entire JSON structure. The final output is serialized by Python, which handles JSON string escaping.

## 7. Sources Provided by the Subject

The original files under `sources/` are kept as provided:

- `sources/call_me_maybe_subject.pdf` contains the assignment requirements.
- `sources/data.zip` contains the function definitions and prompt examples. Their extracted working copies are under `data/input/`.
- `sources/llm_sdk.zip` contains the SDK. Its extracted working copy is under `llm_sdk/` and is installed as a local project dependency.

The application reads the extracted copies. It does not read from, rewrite, or replace the archives or assignment PDF. The `inspiration/` project is a reference only; the application imports its own code from `src/` and the supplied SDK from `llm_sdk/`.

## 8. Dependencies

### Runtime

- **Python 3.10 or later:** runs the application.
- **uv:** installs the dependencies from `pyproject.toml` and `uv.lock`.
- **NumPy:** stores and masks the model's next-token logits before selecting a token.
- **Pydantic:** validates the function definitions and prompt JSON against the input models.
- **Local `llm_sdk`:** wraps model loading, tokenization, vocabulary-file lookup, and next-token inference.
- **PyTorch:** runs the neural model; it is a dependency of the supplied SDK.
- **Transformers:** loads the tokenizer and language model; it is a dependency of the supplied SDK.
- **Hugging Face Hub:** downloads model-related files; it is a dependency of the supplied SDK.

### Development checks

The `dev` dependency group includes **Flake8** for style checks and **mypy** for static type checking. They are run by `make lint`.

## 9. How to Install and Run

Run these commands from the project root.

Install the project and its dependencies:

```sh
make install
```

Run the default prompt set:

```sh
make run
```

The default result file is `data/output/function_calling_results.json`. The first model run may need an internet connection to download Qwen3-0.6B and its tokenizer files.

To select custom input and output files:

```sh
uv run python -m src \
  --functions_definition path/to/functions.json \
  --input path/to/prompts.json \
  --output path/to/results.json
```

Other available commands:

```sh
make debug  # Run the application under Python's debugger.
make lint   # Run Flake8 and mypy.
make clean  # Remove caches and generated data/output results.
```

## 10. Example Execution

An input prompt entry has this shape:

```json
{
  "prompt": "What is the sum of 40 and 2?"
}
```

The corresponding result has this shape:

```json
{
  "prompt": "What is the sum of 40 and 2?",
  "name": "fn_add_numbers",
  "parameters": {
    "a": 40.0,
    "b": 2.0
  }
}
```

The actual output file contains an array of such result objects, one per input prompt. The program reports the destination after writing it:

```text
Results written to data/output/function_calling_results.json
```

## 11. Important Design Decisions

- **Constrain variable tokens:** limit the function name and parameter token choices where the implementation has a supported type rule, instead of trusting the model to produce an entire well-formed call.
- **Insert structure in code:** provide the JSON-like prefix, parameter names, and separators deterministically; build a Python dictionary for the output.
- **Choose greedily:** use the highest-scoring allowed token each time. No sampling or temperature is applied in this decoder.
- **Use the provided data and SDK:** preserve the source material and keep function metadata in JSON, independent from the generation logic.
- **Continue after a prompt error:** preserve one output entry for each prompt and allow later prompts to be processed.

## 12. Limitations

- The program returns a function name and arguments; it does not execute the named function or return its computed result.
- Constrained decoding enforces known function names and basic value types, but does not guarantee that the selected function or extracted values are semantically correct.
- String generation is limited to printable tokens and quote-based stopping; it has no schema-level constraint over the string's meaning.
- A string that exceeds the generation limit may be returned without the model producing a closing quote token.
- A per-prompt generation error produces an empty function name and parameter object. This is recorded in the output, while processing continues.
- The model is fixed to the supplied SDK's default Qwen3-0.6B; the command-line interface does not select other models.
- Inference makes a model call for each generated token, so execution time depends on the device and generated output length.
