VimScript Studying
==================

# Preliminary
* Echo message
  two methods to show message in vim
```
:echo "hello, world!"
```
  and
```
:echom "hello world!"
```
  if use `:messages`, echom will be shown , but echo not

* Setting options
  + bool options
    `set option`, and use `set option?` check current option value; `set option!` to change option value
  + key options
    `set <option> = <value>` and `set <option>?` to check current option value

# Mapping
## Basic
  use `:map {key} {command}`, {key} maybe anything, <space> , <c-xxxx> means ctrl+xxxx, there is also `unmap` or use `map {key} <nop>` to mapping something to no-operation
## Mode mapping
  use `:[niv]map {key} {comand}`, especially note!: when not normal mapping, please use <esc> to return back to normal Mode
## Precise Mapping
  above mapping may be recursive, consider `:nmap dd o<esc>jddk`, it will dead-loop. we can use `:[niv]noremap` to mapping but not remap the {key} to already definition. ex: `:nmap x dd`, then `:nnoremap \ x`, "\" will be to "x", but "x" not be replaced to "dd".
## Leader
  use `set mapleader = "\"` & `set maplocalleader = "\\"` to set <leader>. <localleader> only work for some special case.

# Abbrevation
  use `[irc]abbrev str replace-str` define abbrevation to work in insert, replace & command mode. if str is followed by not `iskeyword?`, then abbrev works. The difference between abbrev & mapping is that, abbrev will consider context, but mapping not

# Local valid
  `setlocal <option>` or `map <buffer> ....` will work for current buffer, if change to another buffer, these will take no effect. if mapping exist in local & global, local will be taken firstly.

# Variable
use `let var = val` to define variable, for options can add "&option" to make it as variable, use prefix "l:" to define local varialbe
`help internal-variables` to see variable scope

# If
use `==#` to compare, because `==` is due to users configuration, may generate wrong answer for string compare, same as `>/<` etc. if string start with non-number, then treat as 0.

# Function
use `call` to call function, the retval will be discard or invoke function in any expression.
Note: function definition should always be with UPPER-Capital. Function's varialbe is immutable.
```
function Myfunc(foo, ...)
  echo a:foo        " named arg
  echo a:0          " return variable arg #0
  echo a:1          " return varialbe arg #1
  echo a:000        " return variable arguments list
endfunction
```

# String & Number
* string
  use `.` to contact 2 strings or one integer, one string, not allowed float varialbe
  when use `''`, no special sequence works, like "\n"
* number
  when use scientific format, `xx.xxexx` the "." should alway be there
