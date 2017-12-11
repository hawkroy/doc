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


