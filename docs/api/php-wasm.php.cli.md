<!-- Do not edit this file. It is automatically generated by API Documenter. -->

## PHP.cli() method
<b>Signature:</b>

```typescript
cli(argv: string[]): Promise<number>;
```

* `argv` – The arguments to pass to the CLI.
* Returns: The exit code of the CLI session.


Starts a PHP CLI session with given arguments.

Can only be used when PHP was compiled with the CLI SAPI.
Cannot be used in conjunction with `run()`<!-- -->.

