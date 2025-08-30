# Range

Ranges can take in up to 3 arguments.

A single argument will be interpreted as the end of the range which is non-inclusive.

```loom
range(10) # iterates from 0 to 9
```

Two arguments will be interpreted as the start and end of the range.

```loom
range(1, 10) # iterates from 1 to 9
```

A third argument can be used to specify the step size.

```loom
range(1, 10, 2) # iterates from 1 to 9 with a step of 2
```