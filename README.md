# SimpleShell
A shell capable of important UNIX sys calls. Written in C.

## Disclaimer: 
This code was written for ECS 150: Operating Systems. As such, I cannot
freely post the source. If you would like to see the actual implementation,
please contact me or (@cfahrney)[https://github.com/cfahrney]. 

## List of functionalities:
1. Execute user-supplied commands with optional arguments
2. Offer a selection of builtin commands
3. Redirect the standard input or standard output of commands to files
4. Pipe the output of commands to other commands
5. Put commands in the background

### 1. User-supplied commands (Phase 2 & 3)
In this section, we decided on the main data structure: `struct Command`.
It originally contained 2 properties:

1. `char* unparsed` to hold the original, untouched command.
2. `char** commandLine` to hold the inputted command in pieces

To parse the original command, we used C's `strtok()` within a custom
`read_command()` function to separate the string by spaces and newline
characters, then placed the resulting pieces into `commandLine`.

Feeding commandLine into `execve` then executed the user-supplied commands.

### 2. Offer a selection of builtin commands (Phase 4)
To check against `cd`, `pwd`, and `exit`, we implemented the following:

`bool check_builtin()`
  - Called immediately in a conditional after `read_command()`
  - If there is a match, it executes the respective custom function
  - If no match, it proceeds with the program's original functionality

Since there is no fork for custom functions, the completion text is printed
to stderr from inside the custom function.

### 3. Redirect the standard input or standard output to files (Phase 5 & 6)
To allow for redirection, we added an additional function within
`read_command()` that checks for any redirection symbol:

`parseCommand()`
  - Searches current command for '<' or '>' using `strchr()`
  - If found, it calls `decipherInput()` or `decipherOutput()` accordingly.

While similar, the two aforementioned functions have some key differences:
- `decipherInput()`
  - Moves character by character to extract file name from the entire unparsed
line
  - Attempts to open existing file for reading from the extracted file name
- `decipherOutput()`
  - Moves character by character to extract file name from the unparsed line
  - Attempts to open a new or existing file for writing from given file name.
File descriptors from these functions were added to `struct Command` in the
form of `int input` and `int output`. Failed attempts were split into two
possibilities: -1 for failed to open the provided file, and -2 for an input or
output symbol found with no following or preceding file name.

In order to minimize special cases, we simply set `input` and `output` on
command initialization to `STDIN_FILENO` and `STDOUT_FILENO` respectively.
They are changed according to `decipherInput()` and/or `decipherOutput()` if
redirection exists in the command.

### 4. Pipe the output of commands to other commands (Phase 7)
Implementing pipes required an addition to our program to link multiple
commands together. We made a doubly linked list containing our existing data
structure, a prev pointer, and a next pointer.
This required a replace-all of existing direct accesses to `struct Command`
with accesses through the linked list.

Within `read_command()`, we checked for the pipe symbol, '|', using `strchr()`. If a single pipe symbol is found, we call the following, implemented function:

`parsePipe()`
   - Breaks the command down with `strtok()` using pipe as a delimiter
   - Uses each spliced segment to create a new node for our linked list
containing an instance of `struct Command`

From here, the head of the linked list is sent to `parseCommand()` where each
node's individual command is correctly parsed.

Once past the call to `fork()`, there were 3 cases to handle for piping:
- General case: Current node was both preceded and succeeded by
another node
- Head case: Current node is the head and has no preceding node
- Tail case: Current node is the tail and has no succeeding node

For the general case, we pull in the previous output to use as our input,
then pipe the output in preparation for the next command. Head and Tail cases
are used to support the general case for input and output redirection.

### 5. Put commands in background (Phase 8)
Accounting for background commands required an additional struct as well as
additional modification to our `read_command()`. The additional struct added is
`struct mapNode`, which contains the following important components:
1. `char* unparsed` to hold the original, untouched command
2. `int pid` to hold the process that is running the command's process ID
3. `int status[64]` to hold the statuses returned from the backgrounded process
4. `int bgStatusSize` to keep track of the size of used ints in status[]
5. `mapNode *prev/*next` to be pointers to the previous and next mapNode within
the linked list, respectively

The changes within `read_command()` for background commands mirrored the change
required for piping. A `strchr()` check was added to look for the background
symbol '&' and, if found, initialized the background or inserted a new command
to the background if there already existed a background process.

Upon entering the parent process, there are two options to process backgrounded
commands:
1. The ***current*** process is to be backgrounded and `waitpid()` is called
with `WNOHANG` to return control back to the shell immediately
2. The current process isn't a background process but we check the background
processes that do exist for any that have completed

For the latter of the two options, we implemented the following:
`checkBGProcs()`
- Loops through linked list of backgrounded commands to check for any that have
completed. If they have, their Process IDs are set to zero for record-keeping
and the exit statuses are recorded into `int status[]`.

Printing out the results for background commands is a simple loop through the
current backgrounded commands and printing those whose Process IDs are zero
(set when `checkBGProcs()` is called), along with their exit statuses.
