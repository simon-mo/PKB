# Database

\#\#\# FOR GSoC - AIMA ONLY ###

I built a relational database management system with Domain Specific Language similar to SQL. With a simple REPL client, user can use SQL like language to store, read, and manipulate data. 

The Spec: 

http://datastructur.es/sp17/materials/proj/proj2/proj2.html

## TOC

[TOC]

## Main features including:

- Load, Store, Print tables
- Parse plain-text formatted table into database and save current table into file
- Insert value into table
- Use `select` statement according SQL logic: `select<column exprs0>,<column expr1>,... from <table0>,<table1>,... where <cond0> and <cond1> and ...`
  - Complex conditional and arithmetic allowed
- Allowing unfit whitespace
- Work with special values like NaN and NOVALUE

##  Design Logic:

I used `HashMap` to represent the columned based structure of a table, `ArrayList` to present specific columns. 

By implementing few filter and parsing method, the commands are handled. 

`Scanner` class is used to handle input file. and `Writer` is used to write files. 

I also used extensive junit testing throughout the process. Test-Driven-Development is a bless. 

## Due to the time constraint, I wasn't able to:

- Use more Regex in my code to simplify the parsing process
- Complete JavaDoc description
- Be more careful about error catching. 