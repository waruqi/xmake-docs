
```lua
target("calc")
     set_kind("binary")
     add_rules("lex", "yacc")
     add_files("src/*.l", "src/*.y")
```
