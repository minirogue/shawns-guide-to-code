# Shawn's Opinionated Guide to Code

This is the repository where I document my opinionated approach to coding best practices with a focus on Android and Kotlin.

## The Site
This documentation is published as a GitHub Pages site that can be found here: https://minirogue.github.io/shawns-guide-to-code.

## Working with this repo
This is mostly notes for myself:

- Use a python virtual environment to install the required packages.
  - `source .venv/bin/activate` to enter the virtual environment stored in `.venv` (not checked into git).
- Use `pip install -r requirements.txt` to install all required packages.
- Use `pur -r requirements.txt` to automatically update `requirements.txt` to the latest versions.
- Use `mkdocs serve` to run a local server.

## Upcoming Topics to Cover
Just a TODO list for future pages.
- State management for MVVM/MVI/UDF.
    - I will cover 3 main approaches to representing the state (monolith `data class`, mutually exclusive`sealed inteface` subclasses, microstates).
    - [Article with reference tweet](https://gpeal.medium.com/modeling-android-screens-as-state-97aa5511657d).
- Discuss keeping Data Transfer Objects (DTOs) and Domain Models separate.
    - I will also mention how gradle modularization can help encapsulate the DTO details.
- Discuss separated interface pattern, modularization strategy, benefits, drawbacks, the whole shebang.
- Standard general coroutines practices
    - injectable dispatchers
    - Responsibility of thread switching at source
    - Collecting `Flow` in an `Activity` or `Fragment` in a lifecycle safe manner.
    - Using MutableStateFlow for VM state
    - Using channels + `receiveAsFlow()` for one-shot events.
- Expand on Kotlin idioms and anti-idioms
  - if != null vs ?.let (multiline/singleline it) vs ?.also, var vs val, etc.
- Unit testing best practices

## Possible Site Improvements

- Need to check if RSS feed is working correctly
- Revisit page organization/navigation once there's a few more pages
- Maybe some images would help break things up within pages?
- Should revise RSS metadata:
  - Image
  - Page descriptions
