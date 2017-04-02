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

I also used extensive junit testing throughout the process. Test-Driven-Development is a blessing. 

## Due to the time constraint, I wasn't able to:

- Use more Regex in my code to simplify the parsing process
- Complete JavaDoc description
- Be more careful about error catching. 

## Code:

#### `Database.java`

```java
package db;

import edu.princeton.cs.algs4.In;

import java.io.IOException;
import java.util.regex.Pattern;
import java.util.regex.Matcher;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.Arrays;
import java.util.NoSuchElementException;
import java.util.Iterator;


public class Database {
    ExchangeSpace db;
    //TODO: delete this after
    public Database() {
        db = new ExchangeSpace();
    }

    public String transact(String query) {

        query = operatorHelper(query); //Handle no space around operator special case.
        //Elimate all unnecssary white space
        query = query.replaceAll("\\s+", " ");
        if (query.indexOf(' ') == 0) {
            query = query.substring(1);
        }
        return eval(query);
    }

    private String eval(String query) {
        Matcher m;
        if ((m = CREATE_CMD.matcher(query)).matches()) {
            return createTable(m.group(1));
        } else if ((m = LOAD_CMD.matcher(query)).matches()) {
            return loadTable(m.group(1));
        } else if ((m = STORE_CMD.matcher(query)).matches()) {
            return storeTable(m.group(1));
        } else if ((m = DROP_CMD.matcher(query)).matches()) {
            return dropTable(m.group(1));
        } else if ((m = INSERT_CMD.matcher(query)).matches()) {
            return insertRow(m.group(1));
        } else if ((m = PRINT_CMD.matcher(query)).matches()) {
            return printTable(m.group(1));
        } else if ((m = SELECT_CMD.matcher(query)).matches()) {
            return select(m.group(1));
        } else {
            return String.format("ERROR: Malformed query: %s\n", query);
        }
    }

    private String createTable(String expr) {
        Matcher m;
        if ((m = CREATE_NEW.matcher(expr)).matches()) {
            return createNewTable(m.group(1), m.group(2).split(COMMA));
        } else if ((m = CREATE_SEL.matcher(expr)).matches()) {
            return createSelectedTable(m.group(1), m.group(2), m.group(3), m.group(4));
        } else {
            return String.format("ERROR: Malformed create: %s\n", expr);
        }
    }

    private String createNewTable(String name, String[] cols) {
        if (db.allTables().contains(name)) {
            return "ERROR: table " + name + "already exist";
        } else if (cols.length == 0) {
            return "ERROR: you can't create table with no columns";
        }
        ArrayList<String> headers = new ArrayList<>(Arrays.asList(cols));
        //Handle Invalid Type
        for (String header : headers) {
            switch (parseTBL.parseHeader(header).get(1)) {
                case "int":
                    break;
                case "string":
                    break;
                case "float":
                    break;
                default:
                    return "ERROR: Invalid type: " + header;
            }
        }
        db.createEmptyColumns(name, headers);

        return "";
    }

    private String createSelectedTable(String name, String exprs, String tables, String conds) {
        if (db.allTables().contains(name)) {
            return "ERROR: table " + name + "already exist";
        } else if (exprs.length() == 0) {
            return "ERROR: you can't create table with no columns";
        }
        String created = select(exprs, tables, conds);
        if (created.contains("ERROR:")) {
            return created;
        }
        Table create = parseTBL.parseString(created);
        db.createFullTable(name, create);
        return "";
    }

    private String loadTable(String name) {
        try {
            db.load(name);
            return "";
        } catch (IOException | NoSuchElementException e) {
            return "ERROR: No such table!" + name;
        } catch (RuntimeException r) {
            return "ERROR: " + r;
        }
    }

    private String storeTable(String name) {
        try {
            db.store(name);
        } catch (NullPointerException e) {
            return "ERROR: table not found!!! " + name;
        }
        return "";
    }

    private String dropTable(String name) {
        try {
            db.drop(name);
            return "";
        } catch (NullPointerException e) {
            return "ERROR: Can't Find Table " + name;
        }
    }

    private String insertRow(String expr) {
        String returned = "";

        Matcher m = INSERT_CLS.matcher(expr);
        if (!m.matches()) {
            return String.format("ERROR: Malformed insert: %s\n", expr);
        }

        String table = m.group(1);
        String literals = m.group(2);
        try {
            ArrayList<String> values =
                    new ArrayList<>(Arrays.asList(literals.split(COMMA)));
            ArrayList<String> headers = db.getTable(table).getHeader();
            ArrayList<String> types = db.getTable(table).getTypes();
            ArrayList<Object> cleanedValues = new ArrayList<>();

            if (headers.size() != values.size()) {
                return "ERROR: Row does not match table";
            }

            Iterator contentIterate = values.iterator();
            while (contentIterate.hasNext()) {
                for (int i = 0; i < headers.size(); i += 1) {
                    String value = (String) contentIterate.next();
                    parseTBL.parseAddToColumn(types.get(i), value, cleanedValues);
                }
            }

            db.insertInto(table, cleanedValues);
        } catch (NumberFormatException n) {
            returned = "ERROR: Malformed data entry: " + literals;
        } catch (NullPointerException e) {
            returned = "ERROR: No such table: " + table;
        } catch (RuntimeException e) {
            returned = "ERROR: Malformed data entry: " + literals;
        }

        return returned;
    }

    private String printTable(String name) {
        try {
            return db.getTable(name).toString();
        } catch (NullPointerException e) {
            return "ERROR: can't find table " + name;
        }
    }

    private String select(String expr) {
        Matcher m = SELECT_CLS.matcher(expr);
        if (!m.matches()) {
            return String.format("ERROR: Malformed select: %s\n", expr);
        }
        return select(m.group(1), m.group(2), m.group(3));
    }

    //TODO: ReWrite This
    private String select(String exprs, String tables, String conds) {
        ArrayList<String> Columns = parseTBL.parseComma(exprs);
        ArrayList<String> Tables = parseTBL.parseComma(tables);
        ArrayList<String> Conds = parseTBL.parseAnd(conds);
        ArrayList<String> finalColumns = new ArrayList<>();

        //Joining Stage
        Table joined = new Table();

        //joined = db.getTable(Tables.get(0));
        for (int i = 0; i < Tables.size(); i += 1) {
            try {
                Table tojoin = parseTBL.parseString(db.getTable(Tables.get(i)).toString()); //Make a copy
                Table newJoin = joined.join(tojoin);
                joined = newJoin;
            } catch (NullPointerException e) {
                return "ERROR: No such table: " + Tables.get(i);
            }
        }

        for (String operations : Columns) {
            if ((!exprs.equals("*")) &&
                    !((operations.contains("+")
                            || operations.contains("-")
                            || operations.contains("*")
                            || operations.contains("/")))) {
                finalColumns.add(operations);
            } else if ((!exprs.equals("*")) && ((operations.contains("+")
                    || operations.contains("-")
                    || operations.contains("*")
                    || operations.contains("/")))) {
                //Arithmatic Select
                try {

                    ArrayList<Column> columns = new ArrayList<>();
                    ArrayList<String> headers = new ArrayList<>();

                    HashMap<String, String> operation = parseTBL.parseConditional(operations);
                    //Use parseCondition for first three elements.
                    if (joined.getNames().contains(operation.get("operand1"))) {
                        //Binary
                        String arith = binaryArithmatic(operations, joined, columns, headers);
                        if (!arith.equals("")) {
                            return arith;
                        }
                    } else {
                        //Unary
                        String arith = unaryArithmatic(operations, joined, columns, headers);
                        if (!arith.equals("")) {
                            return arith;
                        }
                    }

                    for (int i = 0; i < headers.size(); i += 1) {
                        joined.addColumn(headers.get(i), columns.get(i));
                        finalColumns.add(parseTBL.parseHeader(headers.get(i)).get(0));
                    }
                } catch (NullPointerException | NoSuchElementException e) {
                    return "ERROR: Malformed column expression: " + operations;
                }
            }
        }
        //Conditional
        if (conds != null) {
            for (String cond : Conds) {
                HashMap<String, String> conditional = parseTBL.parseConditional(cond);
                String operand1 = conditional.get("operand1");
                if (joined.getNames().contains(operand1)) {
                    //Binary
                    String returnedB = binaryConditional(cond, joined);
                    if (!returnedB.equals("")) {
                        return returnedB;
                    }
                } else {
                    //Unary
                    String returnedU = unaryConditional(cond, joined);
                    if (!returnedU.equals("")) {
                        return returnedU;
                    }
                }

            }

        }

        //Selecting State
        db.loadFullTable("joined", joined);
        if (exprs.equals("*")) {
            return joined.toString();
        } else {
            try {
                joined = db.selectColumnsTable("joined", finalColumns);
                db.drop("joined");
            } catch (RuntimeException e) {
                return "ERROR:  there's no " + exprs + " in " + tables;
            }
        }

        //Return Stage
        return joined.toString();
    }

    private String unaryConditional(String cond, Table joined) {

        ArrayList<Integer> index = new ArrayList<>();
        HashMap<String, String> conditional = parseTBL.parseConditional(cond);

        String head0;
        try {
            head0 = joined.getHeader().get(joined.getNameIndex(conditional.get("operand0")));
        } catch (RuntimeException e) {
            return "ERROR: Column " + conditional.get("operand0") + " dodesn't exist. " + e;
        }

        Column operand0 = joined.getColumn(head0);
        String operator = conditional.get("operator");
        String type0 = parseTBL.parseHeader(head0).get(1);

        String literal = conditional.get("operand1");
        String type1 = parseTBL.unaryTypeConditional(literal);

        if (!type0.equals("string") && !type1.equals("string")) {
            if (type0.equals("int") && type1.equals("int")) {
                for (int i = 0; i < operand0.size(); i += 1) {
                    if (operand0.get(i).toString().equals("NOVALUE")) {

                    } else {
                        Integer val0 = (Integer) operand0.get(i);
                        Integer val1 = Integer.parseInt(literal);

                        switch (operator) {
                            case ">":
                                if (val0.compareTo(val1) > 0) {
                                    index.add(i);
                                }
                                break;
                            case "<":
                                if (val0.compareTo(val1) < 0) {
                                    index.add(i);
                                }
                                break;
                            case "<=":
                                if (val0.compareTo(val1) <= 0) {
                                    index.add(i);
                                }
                                break;
                            case ">=":
                                if (val0.compareTo(val1) >= 0) {
                                    index.add(i);
                                }
                                break;
                            case "!=":
                                if (!val0.equals(val1)) {
                                    index.add(i);
                                }
                                break;
                            case "==":
                                if (val0.equals(val1)) {
                                    index.add(i);
                                }
                                break;
                            default:
                                return "ERROR: Malformed conditional: " + cond;
                        }
                    }
                }
            } else { //int float mixed or all float case
                for (int i = 0; i < operand0.size(); i += 1) {
                    if (operand0.get(i).toString().equals("NOVALUE")) {

                    } else {

                        Number val0N = (Number) operand0.get(i);
                        Float val0 = val0N.floatValue();
                        Float val1 = Float.parseFloat(literal);
                        switch (operator) {
                            case ">":
                                if (val0.compareTo(val1) > 0) {
                                    index.add(i);
                                }
                                break;
                            case "<":
                                if (val0.compareTo(val1) < 0) {
                                    index.add(i);
                                }
                                break;
                            case "<=":
                                if (val0.compareTo(val1) <= 0) {
                                    index.add(i);
                                }
                                break;
                            case ">=":
                                if (val0.compareTo(val1) >= 0) {
                                    index.add(i);
                                }
                                break;
                            case "!=":
                                if (val0.compareTo(val1) != 0) {
                                    index.add(i);
                                }
                                break;
                            case "==":
                                if (val0.compareTo(val1) == 0) {
                                    index.add(i);
                                }
                                break;
                            default:
                                return "ERROR: Malformed conditional: " + cond;
                        }
                    }
                }

            }
        } else if (type0.equals("string") && type1.equals("string")) {
            literal = parseTBL.stripString(literal);
            for (int i = 0; i < operand0.size(); i += 1) {
                if (operand0.get(i).toString().equals("NOVALUE")) {

                } else {


                    switch (operator) {
                        case ">":
                            if (((String) operand0.get(i)).compareTo(literal) > 0) {
                                index.add(i);
                            }
                            break;
                        case "<":
                            if (((String) operand0.get(i)).compareTo(literal) < 0) {
                                index.add(i);
                            }
                            break;
                        case "<=":
                            if (((String) operand0.get(i)).compareTo(literal) <= 0) {
                                index.add(i);
                            }
                            break;
                        case ">=":
                            if (((String) operand0.get(i)).compareTo(literal) >= 0) {
                                index.add(i);
                            }
                            break;
                        case "!=":
                            if (((String) operand0.get(i)).compareTo(literal) != 0) {
                                index.add(i);
                            }
                            break;
                        case "==":
                            if (((String) operand0.get(i)).compareTo(literal) == 0) {
                                index.add(i);
                            }
                            break;
                        default:
                            return "ERROR: Malformed conditional: " + cond;
                    }
                }
            }
        } else {
            return "ERROR: Incompatible types: " + type0 + ", " + type1;
        }
        joined.filterRow(index);
        return "";
    }

    private String binaryConditional(String cond, Table joined) {
        ArrayList<Integer> index = new ArrayList<>();
        HashMap<String, String> conditional = parseTBL.parseConditional(cond);

        String head0, head1;

        try {
            head0 = joined.getHeader().get(joined.getNameIndex(conditional.get("operand0")));
            head1 = joined.getHeader().get(joined.getNameIndex(conditional.get("operand1")));
        } catch (RuntimeException e) {
            return "ERROR: Column " + conditional.get("operand0") + " dodesn't exist. " + e;
        }

        Column operand0 = joined.getColumn(head0);
        String operator = conditional.get("operator");
        String type0 = parseTBL.parseHeader(head0).get(1);


        Column operand1 = joined.getColumn(head1);
        String type1 = parseTBL.parseHeader(head1).get(1);

        if (!type0.equals("string") && !type1.equals("string")) {
            if (type0.equals("int") && type1.equals("int")) {
                for (int i = 0; i < operand0.size(); i += 1) {
                    if (operand0.get(i).toString().equals("NOVALUE")
                            || operand1.get(i).toString().equals("NOVALUE")) {

                    } else {

                        Integer val0 = (Integer) operand0.get(i);
                        Integer val1 = (Integer) operand1.get(i);

                        switch (operator) {
                            case ">":
                                if (val0.compareTo(val1) > 0) {
                                    index.add(i);
                                }
                                break;
                            case "<":
                                if (val0.compareTo(val1) < 0) {
                                    index.add(i);
                                }
                                break;
                            case "<=":
                                if (val0.compareTo(val1) <= 0) {
                                    index.add(i);
                                }
                                break;
                            case ">=":
                                if (val0.compareTo(val1) >= 0) {
                                    index.add(i);
                                }
                                break;
                            case "!=":
                                if (val0.compareTo(val1) != 0) {
                                    index.add(i);
                                }
                                break;
                            case "==":
                                if (val0.compareTo(val1) == 0) {
                                    index.add(i);
                                }
                                break;
                            default:
                                return "ERROR: Malformed conditional: " + cond;
                        }
                    }
                }
            } else { //int float mixed or all float case
                for (int i = 0; i < operand0.size(); i += 1) {
                    if (operand0.get(i).toString().equals("NOVALUE")
                            || operand1.get(i).toString().equals("NOVALUE")) {

                    } else {

                        Number val0N = (Number) operand0.get(i);
                        Number val1N = (Number) operand1.get(i);
                        Float val0 = val0N.floatValue();
                        Float val1 = val1N.floatValue();

                        switch (operator) {
                            case ">":
                                if (val0.compareTo(val1) > 0) {
                                    index.add(i);
                                }
                                break;
                            case "<":
                                if (val0.compareTo(val1) < 0) {
                                    index.add(i);
                                }
                                break;
                            case "<=":
                                if (val0.compareTo(val1) <= 0) {
                                    index.add(i);
                                }
                                break;
                            case ">=":
                                if (val0.compareTo(val1) >= 0) {
                                    index.add(i);
                                }
                                break;
                            case "!=":
                                if (val0.compareTo(val1) != 0) {
                                    index.add(i);
                                }
                                break;
                            case "==":
                                if (val0.compareTo(val1) == 0) {
                                    index.add(i);
                                }
                                break;
                            default:
                                return "ERROR: Malformed conditional: " + cond;
                        }
                    }
                }

            }
        } else if (type0.equals("string") && type1.equals("string")) {
            for (int i = 0; i < operand0.size(); i += 1) {
                if (operand0.get(i).toString().equals("NOVALUE")
                        || operand1.get(i).toString().equals("NOVALUE")) {

                } else {

                    switch (operator) {
                        case ">":
                            if (((String) operand0.get(i)).compareTo((String) operand1.get(i)) > 0) {
                                index.add(i);
                            }
                            break;
                        case "<":
                            if (((String) operand0.get(i)).compareTo((String) operand1.get(i)) < 0) {
                                index.add(i);
                            }
                            break;
                        case "<=":
                            if (((String) operand0.get(i)).compareTo((String) operand1.get(i)) <= 0) {
                                index.add(i);
                            }
                            break;
                        case ">=":
                            if (((String) operand0.get(i)).compareTo((String) operand1.get(i)) >= 0) {
                                index.add(i);
                            }
                            break;
                        case "!=":
                            if (((String) operand0.get(i)).compareTo((String) operand1.get(i)) != 0) {
                                index.add(i);
                            }
                            break;
                        case "==":
                            if (((String) operand0.get(i)).compareTo((String) operand1.get(i)) == 0) {
                                index.add(i);
                            }
                            break;
                        default:
                            return "ERROR: Malformed conditional: " + cond;
                    }
                }
            }
        } else {
            return "ERROR: Incompatible types: " + type0 + ", " + type1;
        }
        joined.filterRow(index);
        return "";
    }


    private String binaryArithmatic(String operations,
                                    Table joined,
                                    ArrayList<Column> columns, ArrayList<String> headers) {


        String asColumnType = "";
        HashMap<String, String> operation = parseTBL.parseOperations(operations);

        Column resultColumn = new Column();

        //Building Table: New Column Name
        String asColumnName = operation.get("asColumnName");

        //Check Types
        ArrayList<String> types = joined.getTypes();
        ArrayList<String> header = joined.getHeader();

        String header0 = header.get(joined.getNameIndex(operation.get("operand0")));
        String header1 = header.get(joined.getNameIndex(operation.get("operand1")));

        String type0 = types.get(joined.getNameIndex(operation.get("operand0")));
        String type1 = types.get(joined.getNameIndex(operation.get("operand1")));
        Column operand0 = joined.getColumn(header0);
        Column operand1 = joined.getColumn(header1);

        for (int i = 0; i < operand0.size(); i += 1) {
            if (operand0.get(i).toString().equals("NaN")) {
                resultColumn.add(new NaN<>());
                asColumnType = columnTypeDecideHelper(type0, type1);
                if (asColumnType.contains("ERROR")) {
                    return asColumnType;
                }
            } else if (!type0.equals("string") && !type1.equals("string")) { //Int and Float cases, OK to use Number.
                if (type0.equals("int") && type1.equals("int")) {

                    Integer val0 = null;
                    Integer val1 = null;

                    if (operand0.get(i).toString().equals("NOVALUE")
                            && operand1.get(i).toString().equals("NOVALUE")) {
                        resultColumn.add(new NOVALUE());
                        asColumnType = "int";
                    } else {

                        if (operand0.get(i).toString().equals("NOVALUE")) {
                            val0 = Integer.parseInt("0");
                            val1 = (Integer) operand1.get(i);
                        } else if (operand1.get(i).toString().equals("NOVALUE")) {
                            val1 = Integer.parseInt("0");
                            val0 = (Integer) operand0.get(i);
                        } else {
                            val1 = (Integer) operand1.get(i);
                            val0 = (Integer) operand0.get(i);
                        }

                        switch (operation.get("operator")) {
                            //We don't need to worry about unmatched row numbers
                            case "+":
                                resultColumn.add(val0 + val1);
                                asColumnType = "int";
                                break;
                            case "-":
                                resultColumn.add(val0 - val1);
                                asColumnType = "int";
                                break;
                            case "*":
                                resultColumn.add(val0 * val1);
                                asColumnType = "int";
                                break;
                            case "/":
                                if ((val1).equals(0)) {
                                    resultColumn.add(new NaN());
                                } else {
                                    resultColumn.add(val0 / val1);
                                }
                                asColumnType = "int";
                                break;
                            default:
                                return "ERROR: Malformed column expression: " + operations;
                        }
                    }
                } else {

                    if (operand0.get(i).toString().equals("NOVALUE")
                            && operand1.get(i).toString().equals("NOVALUE")) {
                        resultColumn.add(new NOVALUE());
                        asColumnType = "int";
                    } else {

                        Float val0 = null;
                        Float val1 = null;

                        if (operand0.get(i).toString().equals("NOVALUE")) {
                            val0 = Float.parseFloat("0.0f");
                            Number v1 = (Number) operand1.get(i);
                            val1 = v1.floatValue();
                        } else if (operand1.get(i).toString().equals("NOVALUE")) {
                            val1 = Float.parseFloat("0.0f");
                            Number v0 = (Number) operand0.get(i);
                            val0 = v0.floatValue();
                        } else {
                            Number v0 = (Number) operand0.get(i);
                            val0 = v0.floatValue();
                            Number v1 = (Number) operand1.get(i);
                            val1 = v1.floatValue();
                        }


                        switch (operation.get("operator")) {
                            //We don't need to worry about unmatched row numbers
                            case "+":
                                Number result = val0 + val1;
                                resultColumn.add(result.floatValue());
                                asColumnType = "float";
                                break;
                            case "-":
                                Number resultM = val0 - val1;
                                resultColumn.add(resultM.floatValue());
                                asColumnType = "float";
                                break;
                            case "*":
                                Number resultT = val0 * val1;
                                resultColumn.add(resultT.floatValue());
                                asColumnType = "float";
                                break;
                            case "/":
                                if (val1 == 0.0f) {
                                    resultColumn.add(new NaN());
                                } else {
                                    Number resultD = val0 / val1;
                                    resultColumn.add(resultD.floatValue());
                                }
                                asColumnType = "float";
                                break;
                            default:
                                return "ERROR: Malformed column expression: " + operations;
                        }
                    }
                }
            } else if (type0.equals("string") && type1.equals("string")) {

                String val0 = null;
                String val1 = null;

                if (operand0.get(i).toString().equals("NOVALUE")
                        && operand1.get(i).toString().equals("NOVALUE")) {
                    resultColumn.add(new NOVALUE());
                    asColumnType = "int";
                } else {
                    if (operand0.get(i).toString().equals("NOVALUE")) {
                        val0 = "";
                        val1 = (String) operand1.get(i);
                    } else if (operand1.get(i).toString().equals("NOVALUE")) {
                        val1 = "";
                        val0 = (String) operand0.get(i);
                    } else {
                        val1 = (String) operand1.get(i);
                        val0 = (String) operand0.get(i);
                    }


                    switch (operation.get("operator")) {
                        //We don't need to worry about unmatched row numbers
                        case "+":
                            resultColumn.add(val0 + val1);
                            asColumnType = "string";
                            break;
                        case "-":
                            return "ERROR: Strings do not support subtraction";
                        case "*":
                            return "ERROR: Strings do not support multiplication";
                        case "/":
                            return "ERROR: Strings do not support division";
                        default:
                            return "ERROR: Malformed column expression: " + operations;
                    }
                }
            } else {
                return "ERROR: Incompatible types: " + type0 + ", " + type1;
            }
        }
        columns.add(resultColumn);
        headers.add(asColumnName + " " + asColumnType);

        return "";
    }

    private String unaryArithmatic(String operations,
                                   Table joined,
                                   ArrayList<Column> columns, ArrayList<String> headers) {

        String asColumnType = "";
        HashMap<String, String> operation = parseTBL.parseConditional(operations);
        //Use parseConditional Here because we just need three elements.

        Column resultColumn = new Column();

        //Building Table: New Column Name
        String asColumnName = operation.get("operand0");

        //Check Types
        ArrayList<String> types = joined.getTypes();
        ArrayList<String> header = joined.getHeader();

        String header0 = header.get(joined.getNameIndex(operation.get("operand0")));
        String type0 = types.get(joined.getNameIndex(operation.get("operand0")));
        Column operand0 = joined.getColumn(header0);

        String type1 = parseTBL.unaryTypeConditional(operation.get("operand1"));
        String string1 = operation.get("operand1");
        for (int i = 0; i < operand0.size(); i += 1) {
            if (operand0.get(i).toString().equals("NaN")) {
                resultColumn.add(new NaN<>());
                asColumnType = columnTypeDecideHelper(type0, type1);
                if (asColumnType.contains("ERROR")) {
                    return asColumnType;
                }
            } else if (!type0.equals("string") && !type1.equals("string")) {
                if (type0.equals("int") && type1.equals("int")) {

                    Integer val0 = null;


                    if (operand0.get(i).toString().equals("NOVALUE")) {
                        val0 = Integer.parseInt("0");
                    } else {
                        val0 = (Integer) operand0.get(i);
                    }

                    Integer int1 = Integer.parseInt(string1);
                    resultColumn.add((Integer) int1);
                    asColumnType = "int";
                    switch (operation.get("operator")) {
                        //We don't need to worry about unmatched row numbers
                        case "+":
                            resultColumn.add(val0 + int1);
                            break;
                        case "-":
                            resultColumn.add(val0 - int1);
                            asColumnType = "int";
                            break;
                        case "*":
                            resultColumn.add(val0 * int1);
                            asColumnType = "int";
                            break;
                        case "/":
                            if ((int1).equals(0)) {
                                resultColumn.add(new NaN());
                            } else {
                                resultColumn.add(val0 / int1);
                            }
                            asColumnType = "int";
                            break;
                        default:
                            return "ERROR: Malformed column expression: " + operations;
                    }
                } else {
                    Float val0 = null;

                    if (operand0.get(i).toString().equals("NOVALUE")) {
                        val0 = Float.parseFloat("0.0f");
                    } else {
                        Number v0 = (Number) operand0.get(i);
                        val0 = v0.floatValue();
                    }


                    Float float1 = Float.parseFloat(string1);

                    Number result;
                    switch (operation.get("operator")) {
                        //We don't need to worry about unmatched row numbers
                        case "+":
                            result = val0 + float1;
                            resultColumn.add(result.floatValue());
                            asColumnType = "float";
                            break;
                        case "-":
                            result = val0 - float1;
                            resultColumn.add(result.floatValue());
                            asColumnType = "float";
                            break;
                        case "*":
                            result = val0 * float1;
                            resultColumn.add(result.floatValue());
                            asColumnType = "float";
                            break;
                        case "/":
                            if (float1 == 0.0f) {
                                resultColumn.add(new NaN());
                            } else {
                                result = val0 / float1;
                                resultColumn.add(result.floatValue());
                            }
                            asColumnType = "float";
                            break;
                        default:
                            return "ERROR: Malformed column expression: " + operations;
                    }
                }
            } else if (type0.equals("string") && type1.equals("string")) {

                String val0 = null;

                if (operand0.get(i).toString().equals("NOVALUE")) {
                    val0 = "";
                } else {
                    val0 = (String) operand0.get(i);
                }

                switch (operation.get("operator")) {
                    //We don't need to worry about unmatched row numbers
                    case "+":
                        resultColumn.add(val0 + parseTBL.stripString(string1));
                        asColumnType = "string";
                        break;
                    case "-":
                        return "ERROR: Strings do not support subtraction";
                    case "*":
                        return "ERROR: Strings do not support multiplication";
                    case "/":
                        return "ERROR: Strings do not support division";
                    default:
                        return "ERROR: Malformed column expression: " + operations;
                }
            } else {
                return "ERROR: Incompatible types: " + type0 + ", " + type1;
            }
        }

        columns.add(resultColumn);
        headers.add(asColumnName + " " + asColumnType);

        return "";
    }

    private String operatorHelper(String query) {
        if (query.contains("insert into")) {
            return query;
        }
        if (query.contains("+")) {
            query = query.replaceAll("\\+", " + ");
        }
        if (query.contains("-")) {
            query = query.replaceAll("-", " - ");
        }
        if (query.contains("*")) {
            query = query.replaceAll("\\*", " * ");
        }
        if (query.contains("/")) {
            query = query.replaceAll("/", " / ");
        }
        if (query.contains(">")) {
            query = query.replaceAll(">(?!=)", " > ");
        }
        if (query.contains(">=")) {
            query = query.replaceAll(">=", " >= ");
        }
        if (query.contains("!=")) {
            query = query.replaceAll("!=", " != ");
        }
        if (query.contains("<")) {
            query = query.replaceAll("<(?!=)", " < ");
        }
        if (query.contains("<=")) {
            query = query.replaceAll("<=", " <= ");
        }
        if (query.contains("==")) {
            query = query.replaceAll("==", " == ");
        }

        return query;
    }

    private String columnTypeDecideHelper(String type0, String type1) {
        if (!type0.equals("string") && !type1.equals("string")) {
            if (type0.equals("int") && type1.equals("int")) {
                return "int";
            } else {
                return "float";
            }
        } else if (type0.equals("string") && type1.equals("string")) {
            return "string";
        } else {
            return "ERROR: Incompatible types: " + type0 + ", " + type1;
        }
    }

    /**
     * Constant
     */
    // Various common constructs, simplifies parsing.
    private final String REST = "\\s*(.*)\\s*",
            COMMA = "\\s*,\\s*",
            AND = "\\s+and\\s+";

    // Stage 1 syntax, contains the command name.
    private final Pattern CREATE_CMD = Pattern.compile("create table " + REST),
            LOAD_CMD = Pattern.compile("load " + REST),
            STORE_CMD = Pattern.compile("store " + REST),
            DROP_CMD = Pattern.compile("drop table " + REST),
            INSERT_CMD = Pattern.compile("insert into " + REST),
            PRINT_CMD = Pattern.compile("print " + REST),
            SELECT_CMD = Pattern.compile("select " + REST);

    // Stage 2 syntax, contains the clauses of commands.
    private final Pattern CREATE_NEW = Pattern.compile("(\\S+)\\s+\\((\\S+\\s+\\S+\\s*"
            + "(?:,\\s*\\S+\\s+\\S+\\s*)*)\\)"),
            SELECT_CLS = Pattern.compile("([^,]+?(?:,[^,]+?)*)\\s+from\\s+"
                    + "(\\S+\\s*(?:,\\s*\\S+\\s*)*)(?:\\s+where\\s+"
                    + "([\\w\\s+\\-*/'<>=!]+?(?:\\s+and\\s+"
                    + "[\\w\\s+\\-*/'<>=!]+?)*))?"),
            CREATE_SEL = Pattern.compile("(\\S+)\\s+as select\\s+"
                    + SELECT_CLS.pattern()),
            INSERT_CLS = Pattern.compile("(\\S+)\\s+values\\s+(.+?"
                    + "\\s*(?:,\\s*.+?\\s*)*)");
}

```

#### `ExchangeSpace.java`

```java
package db;

import java.io.IOException;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.HashMap;
import java.io.File;
import java.util.Set;

/**
 * Try to build an exchange space,
 * Pretty much a intermediate for operation.
 * Created by simonmo on 2/21/17.
 */
public class ExchangeSpace {
    private HashMap<String, Table> database;

    public ExchangeSpace() {
        database = new HashMap<>();
    }

    public Set<String> allTables() {
        return database.keySet();
    }

    /**
     * Create method can only be called within an instance of ExchangeSpace
     */
    public void createBlankTable(String name) {
        database.put(name, new Table());
    }

    public void createEmptyColumns(String name, ArrayList<String> header) {
        createBlankTable(name);
        Table curr = database.get(name);
        for (String head: header) {
            curr.addColumn(head, new Column());
        }
    }

    public void createFullTable(String name, Table table) {
        database.put(name, table);
    }

    /**
     *Create a new Table in the database with header and columns.
     * Suitable for "select * ( or columns names) from multiple tables.
     * you need a new same for it, therefore: "as ..."
     * @param name
     * @param header
     * @param columns
     */
    public void createFullTable(String name, ArrayList<String> header, ArrayList<Column> columns) {
        if (header.size() != columns.size()) {
            throw new RuntimeException("header and columns are not the same size!");
        }
        createBlankTable(name);
        for (int i = 0; i < header.size(); i += 1) {
            database.get(name).addColumn(header.get(i), columns.get(i));
        }
    }

    /**
     * Input need to be an ArrayList of literals.
     * @param tableName
     * @param literals
     */
    public void insertInto(String tableName, ArrayList<Object> literals) {
        Table current = database.get(tableName);
        ArrayList<String> types = current.getHeader();

        for (int i = 0; i < types.size(); i += 1) {
            current.getColumn(types.get(i)).add(literals.get(i));
        }
    }


    /**
     * Input should be tbl name
     * Default "exchange" folder
     * @param name without tbl
     */
    public void load(String name) throws IOException {
        database.put(name, parseTBL.parseTBL("", name + ".tbl"));
    }

    /**
     * Input should be tbl name
     * @param path name of the directory
     * @param name without tbl
     */
    public void loadWithPath(String path, String name) {
        try {
            database.put(name, parseTBL.parseTBL(path, name + ".tbl"));
        } catch (IOException e) {
            System.out.println("Load failed, such table doesn't exist " + name);
        }
    }

    /**
     * Special Case of inner operation
     */
    public void loadFullTable(String name, Table t) {
        database.put(name, t);
    }

    /**
     * Retrieve reference to a particular table
     * @param name of the table.
     * @return Table
     */
    public Table getTable(String name) {
        return database.get(name);
    }

    /**
     * Select a particular column from the table.
     * @param name Table name
     * @param header the Column header
     * @return Column data type.
     */
     public Column selectColumn(String name, String header) {
         return getTable(name).getColumnByName(header);
     }


    /**
     * Create a new table from several columns of a table.
     * @param name The name of the Table.
     * @param headers ArrayList of Tables.
     * @return Table
     */
     public Table selectColumnsTable(String name, ArrayList<String> headers) {
         Table selected = new Table();

         for (String head: headers) {
             String headFull = getTable(name).getHeader().get(getTable(name).getNames().indexOf(head));
             selected.addColumn(headFull, selectColumn(name, head));
         }

         return selected;
     }

    /**
     * Return an arraylist of selected colummns
     * @param name
     * @param headers
     * @return
     */
    public ArrayList<Column> selectColumnsList(String name, ArrayList<String> headers) {
        ArrayList<Column> selected = new ArrayList();

        for (String header: headers) {

            for (String tableHeader: getTable(name).getHeader()) {
                if (tableHeader.contains(header)) {
                    header = tableHeader;
                }
            }

            selected.add(selectColumn(name, header));
        }

        return selected;
    }

    /**
     * without tbl, expect to add.
     * @param name a key from database.
     */
    public void store(String name) {
        try {
            Table temp = database.get(name);
            name += ".tbl";
            new saveTBL().save(name, temp.toString());
        } catch (NullPointerException e) {
           throw new NullPointerException("Table not found" + e);
        }
    }

    /**
     * drop table from the database
     * Detached.
     * @param name the key.
     */
    public void drop(String name) throws NullPointerException {
        if (allTables().contains(name)) {
            database.remove(name);
        } else {
            throw new NullPointerException("ERROR: No such table: " + name);
        }
    }

    /**
     * Return string representation of a selected table.
     * @param table
     * @return
     */
    public String getString(String table) {
        try {
            return database.get(table).toString();
        } catch (NullPointerException e) {
            System.out.print("Print Error: table not found");
        }
        return "";
    }

    /**Delete all files in exhcange folder.
     *
     */
    public void endSession() {
        File exchangePath = new File("exchange/");
        File rootPath = new File(".");
        try {
            for(File file: exchangePath.listFiles()) {
                if (!file.isDirectory())
                    file.delete();
            }

            for (File file: rootPath.listFiles()) {
                if (file.getName().contains(".tbl")) {
                    file.delete();
                }
            }

        } catch (NullPointerException e) {
        }
    }

}

```

#### `parseTBL.java`

```java
package db;

import java.io.IOException;
import java.util.*;
import java.nio.file.*;

/**
 * Created by simonmo on 2/18/17.
 * Use it by new parseTBL().parTBL(first, morefilname)
 * Return a Table.
 */

public class parseTBL {
    private Table parsed = new Table();

    public static ArrayList<String> parseHeader(String header) {
        Scanner scanner = new Scanner(header);
        ArrayList<String> head = new ArrayList<>();
        head.add(scanner.next());
        head.add(scanner.next());
        return head;
    }


    public static HashMap<String, String> parseConditional(String conditionals) {
        Scanner scanner = new Scanner(conditionals);
        HashMap<String, String> result = new HashMap<>();

        result.put("operand0", scanner.next());
        result.put("operator", scanner.next());
        result.put("operand1", scanner.next());

        return result;
    }

    public static String unaryTypeConditional(String input) {
        if (input.contains("\'")) {
            return "string";
        } else if (input.contains(".")) {
            return "float";
        } else {
            return "int";
        }
    }


    public static HashMap<String, String> parseOperations(String operations) {
        Scanner scanner = new Scanner(operations);
        HashMap<String, String> result = new HashMap<>();

        result.put("operand0", scanner.next());
        result.put("operator", scanner.next());
        result.put("operand1", scanner.next());
        scanner.next();
        result.put("asColumnName", scanner.next());

        return result;
    }


    public static boolean checkType(String header, String expectedType) {
        switch (expectedType) {
            case "int":
                try {
                    Integer.parseInt(header);
                } catch (NumberFormatException e) {
                    return false;
                }
                break;
            case "float":
                try {
                    Float.parseFloat(header);
                } catch (NumberFormatException e) {
                    return false;
                }
                break;
            case "string":
                try {
                    if (header.indexOf('\'') == -1) {
                        //throw new RuntimeException("String be formatted as \'string\' ");
                        return false;
                    }
                } catch (NumberFormatException e) {
                    return false;
                }
                break;
            default:
                return true;
        }
        return true;
    }

    public static Table parseTBL(String first, String morefilename) throws IOException {
        Path path = FileSystems.getDefault().getPath(first, morefilename);
        Scanner scanner = new Scanner(path);
        return parseScanner(scanner);
    }


    public static Table parseString(String table) {
        Scanner scanner = new Scanner(table);
        return parseScanner(scanner);
    }

    private static Table parseScanner(Scanner scanner) {
        ArrayList<String> header = new ArrayList<>();
        ArrayList<String> types = new ArrayList<>();
        ArrayList<String> contents = new ArrayList<>();
        ArrayList<Column> columns = new ArrayList<>();
        int columnCounts = 0;

        //Find Header
        String next = scanner.nextLine();
        Scanner nextScan = new Scanner(next).useDelimiter("\\s*,\\s*");
        while (nextScan.hasNext()){
            header.add(nextScan.next());
            columnCounts += 1;
        }

        //Find Type
        Scanner typeScanner = new Scanner(next).useDelimiter("\\s*,\\s*");
        for (int i = 0; i < columnCounts; i += 1) {
            String element = typeScanner.next();
            Scanner searchType = new Scanner(element);
            String name = searchType.next();
            String type = searchType.next();
            types.add(type);
        }


        //Fill in the rest of elements in contents
        // These are still string.
        while (scanner.hasNextLine()) {
            int counter = 0;
            String nextL = scanner.nextLine();
            Scanner nextScanL = new Scanner(nextL).useDelimiter("\\s*,\\s*");
            while (nextScanL.hasNext()){
                contents.add(nextScanL.next());
            }
        }

        // Now we create new Coloums.
        for (int i = 0; i < columnCounts; i += 1) {
            switch (types.get(i)) {
                case "int":
                    columns.add(new Column<Integer>("int"));
                    break;
                case "float":
                    columns.add(new Column<Float>("float"));
                    break;
                case "string":
                    columns.add(new Column<String>("string"));
                    break;
                default:
                    throw new RuntimeException("Incompatible type: must be int, float, string");
            }
        }

        //Now we synchronize columns and content
        Iterator contentIterate = contents.iterator();
        while (contentIterate.hasNext()) {
            for (int i = 0; i < columnCounts; i += 1) {
                String type = types.get(i);
                String value = (String) contentIterate.next();
                parseAddToColumn(type, value, columns.get(i));

            }
        }

        Table parsed = new Table();

        //Finally add them to the table
        for (int i = 0; i < columnCounts; i += 1) {
            parsed.addColumn(header.get(i), columns.get(i));
        }

        return parsed;
    }

    public static void parseAddToColumn(String type, String content, ArrayList col) {
        switch (type) {
            case "int":
                if (content.toString().equals("NOVALUE")) {
                    col.add(new NOVALUE());
                } else if (checkType(content, "int")){
                    col.add(Integer.parseInt(content));
                } else {
                    throw new RuntimeException("Incompatible type:must be int,float,string");
                }
                break;
            case "float":
                if (content.toString().equals("NOVALUE")) {
                    col.add(new NOVALUE());
                } else if (checkType(content, "float")) {
                    col.add(Float.parseFloat(content));
                } else {
                    throw new RuntimeException("Incompatible type:must be int,float,string");
                }
                break;
            case "string":
                if (content.toString().equals("NOVALUE")) {
                    col.add(new NOVALUE());
                } else if (content.equals("NaN")){
                    col.add(new NaN<>());
                } else if (checkType(content, "string")) {
                    content = content.substring(1, content.length() - 1); //Strip away the single ' mark.
                    col.add(content);
                } else {
                    throw new RuntimeException("Incompatible type:must be int,float,string");
                }
                break;
            default:
                throw new RuntimeException("Incompatible type:must be int,float,string");
        }
    }

    /**
     * Parse a comma seperated string to Array List of string.
     * @param exprs
     * @return
     */
    public static ArrayList<String> parseComma(String exprs) {
        if (exprs == null) {
            return new ArrayList<>();
        }

        String COMMA = "\\s*,\\s*";
        Scanner temp = new Scanner(exprs);
        temp.useDelimiter(COMMA);
        ArrayList<String> list = new ArrayList<>();
        while (temp.hasNext()) {
            list.add(temp.next());
        }
        return list;
    }



    public static ArrayList<String> parseAnd(String exprs) {
        if (exprs == null) {
            return new ArrayList<>();
        }

        String AND = "\\s*and\\s*";
        Scanner temp = new Scanner(exprs);
        temp.useDelimiter(AND);
        ArrayList<String> list = new ArrayList<>();
        while (temp.hasNext()) {
            list.add(temp.next());
        }
        return list;
    }


    public static String stripString(String quoted) {
        return quoted.substring(1, quoted.length()-1);
    }
}

```



#### `Table.java`

```java
package db;

import java.util.*;


/**
 * Table class extends HashMap with type String, Column: header and column.
 * Its key is the type/header of each column,
 * and the value is the content of the column.
 * <p>
 * This is not a generic class because Table class always consist of String header and Hashmap
 */
public class Table extends HashMap<String, Column> {
    private HashMap<String, Column> table;
    private ArrayList<String> header = new ArrayList<>(); //x int
    private ArrayList<String> names = new ArrayList<>();  //x
    private ArrayList<String> types = new ArrayList<>();  //int

    /**
     * Default Column Constructor.
     * Initialize a new HashMap.
     */
    public Table() {
        table = new HashMap<>();
    }

    public boolean equals(Table t) {
        if (header.equals(t.getHeader())) {
            for (String header : t.getHeader()) {
                if (!getColumn(header).equals(t.getColumn(header))) {
                    return false;
                }
            }
        } else {
            return false;
        }
        return true;
    }

    @Override
    public boolean isEmpty() {
        return table.isEmpty();
    }

    /**
     * addColumn
     *
     * @param Header the "Header of the column", like "x int".
     * @param Column the content,
     */
    public void addColumn(String Header, Column Column) {
        table.put(Header, Column);
        header.add(Header);
        ArrayList<String> nameAndType = parseTBL.parseHeader(Header);
        names.add(nameAndType.get(0));
        types.add(nameAndType.get(1));
    }

    /**
     * Extract a single column with type Header from the table
     *
     * @param Header in String, like "x int"
     * @return a column.
     */
    public Column getColumn(String Header) {
        int index = header.indexOf(Header);
        if (index == -1) {
            throw new RuntimeException("Can't find the type: " + Header);
        }
        return table.get(header.get(index));
    }

    /**
     * Use name like x, instead "x int"
     *
     * @param name "x"
     * @return
     */
    public Column getColumnByName(String name) {
        int index = names.indexOf(name);
        if (index == -1) {
            throw new RuntimeException("Can't find Column: " + name);
        }
        return table.get(header.get(index));
    }


    /**
     * make a table with array of headers and columns
     *
     * @param header  "x int"
     * @param columns Column type data.
     * @return "this"
     */
    public Table createFullTable(ArrayList<String> header, ArrayList<Column> columns) {
        Table t = new Table();
        if (header.size() != columns.size()) {
            throw new RuntimeException("header and columns are not the same size!");
        }
        for (int i = 0; i < header.size(); i += 1) {
            t.addColumn(header.get(i), columns.get(i));
        }
        return t;
    }


    /**
     * Useful method to testing and presenting.
     * Formatted according to project 2 spec.
     */
    @Override
    public String toString() {
        StringBuilder outputBuff = new StringBuilder();
        //String output = "";

        //Printing Header
        for (String type : header) {
            outputBuff.append(type);
            //output += type;
            if (header.indexOf(type) != header.size() - 1) {
                outputBuff.append(",");
                //output += ',';
            } else if (table.get(header.get(0)).size() != 0) {
                //The messy conditional above handle the edge case
                //of an empty table.

                outputBuff.append("\n");
                //output += '\n';
            }
        }

        //Printing Rows
        Column firstColumn = table.get(header.get(0));
        int rowNumbers = firstColumn.size();
        for (int i = 0; i < rowNumbers; i += 1) {
            for (String head : header) {
                Column col = (Column) table.get(head);
                if (head.contains("float")) {
                    if (col.get(i).toString().equals("NOVALUE")) {
                        outputBuff.append("NOVALUE");
                    } else if (col.get(i).toString().equals("NaN")) {
                        outputBuff.append("NaN");
                        //output += "NaN";
                    } else {
                        //Use the custom rounding method.
                        Float f = (Float) col.get(i);

                        outputBuff.append(roundHelper(f));
                        //output += roundHelper(f);
                    }
                } else if (head.contains("string")) {
                    if (col.get(i).toString().equals("NOVALUE")) {
                        outputBuff.append("NOVALUE");
                    } else {
                        outputBuff.append("\'" + col.get(i).toString() + "\'");
                    }
                    //output += "\'" + col.get(i).toString() + "\'";
                } else {
                    outputBuff.append(col.get(i).toString());
                    //output += col.get(i).toString();
                }

                if (header.indexOf(head) != header.size() - 1) {
                    outputBuff.append(",");
                    //output += ',';
                } else if (i != rowNumbers - 1) {
                    outputBuff.append("\n");
                    //output += '\n';
                }
            }
        }
        return outputBuff.toString();
    }

    /**
     * get the headers role in ArrayList
     *
     * @return Arralist of header
     */
    public ArrayList<String> getHeader() {
        return header;
    }

    public ArrayList<String> getTypes() {
        return types;
    }

    public ArrayList<String> getNames() {
        return names;
    }

    public int getNameIndex(String name) {
        if (names.indexOf(name) == -1) {
            throw new RuntimeException("Column " + name + " doesn't exist");
        }
        return names.indexOf(name);
    }

    /**
     * Join method, by the spec.
     *
     * @param toJoin must be Table type.
     * @return a new Table
     */
    public Table join(Table toJoin) {
        if (this.isEmpty()) {
            return toJoin;
        } else if (toJoin.isEmpty()) {
            return this;
        } else if (this.equals(toJoin)) {
            return this;
        }

        ArrayList<String> sameColumn = new ArrayList<>();
        ArrayList<Integer> sameRow = new ArrayList();
        ArrayList<Integer> sameRowB = new ArrayList();
        Table joined = new Table();

        //Find the sameColumn,
        //append the type to sameColumn
        ArrayList<String> toJoinTableHeaders = toJoin.getHeader();
        for (String joinTableHeader : header) {
            for (String toJoinTableHeader : toJoinTableHeaders) {
                if (joinTableHeader.equals(toJoinTableHeader)) {
                    sameColumn.add(joinTableHeader);
                }
            }
        }

        //Handle Special Case: Cartesian Join when two tables have no common column
        if (sameColumn.size() == 0) {
            //get the row size of both table
            int rowA = getColumn(header.get(0)).size();
            ArrayList<String> headerB = toJoin.getHeader();
            int rowB = toJoin.getColumn((String) headerB.get(0)).size();

            //number of columns
            int numOfColumns = header.size() + headerB.size();

            //make the columns
            ArrayList<Column> columnsA = new ArrayList<>();
            for (String aHeader : header) {
                if (parseTBL.parseHeader(aHeader).get(1).equals("int")) {
                    columnsA.add(new Column<Integer>());
                } else if (parseTBL.parseHeader(aHeader).get(1).equals("float")) {
                    columnsA.add(new Column<Float>());
                } else {
                    columnsA.add(new Column<String>());
                }
            }

            ArrayList<Column> columnsB = new ArrayList<>();
            for (int b = 0; b < headerB.size(); b += 1) {
                if (parseTBL.parseHeader(headerB.get(b)).get(1).equals("int")) {
                    columnsB.add(new Column<Integer>());
                } else if (parseTBL.parseHeader(headerB.get(b)).get(1).equals("float")) {
                    columnsB.add(new Column<Float>());
                } else {
                    columnsB.add(new Column<String>());
                }
            }

            //fill in
            for (int x = 0; x < columnsA.size(); x += 1) {
                for (int i = 0; i < rowA; i += 1) {
                    for (int j = 0; j < rowB; j += 1) {
                        columnsA.get(x).add(getColumn(header.get(x)).get(i));
                    }
                }
            }

            for (int y = 0; y < columnsB.size(); y += 1) {
                for (int p = 0; p < rowA; p += 1) {
                    for (int q = 0; q < rowB; q += 1) {
                        columnsB.get(y).add(toJoin.getColumn(headerB.get(y)).get(q));
                    }
                }
            }

            //create table and return
            ArrayList<String> temp = new ArrayList();
            temp.addAll(header);
            temp.addAll(headerB);
            ArrayList<Column> temp2 = new ArrayList();
            temp2.addAll(columnsA);
            temp2.addAll(columnsB);
            for (int nums = 0; nums < numOfColumns; nums++) {
                joined.addColumn(temp.get(nums), temp2.get(nums));
            }

            return joined;
        }

        //Are they all the same????
        boolean sameSpecialCase = true;
        for (String col : sameColumn) {
            if (!this.getColumn(col).equals(toJoin.getColumn(col))) {
                sameSpecialCase = false;
            }
        }

        //Deal with special case!!!
        if (sameSpecialCase) {
            for (String col : sameColumn) {
                joined.addColumn(col, this.getColumn(col));
            }
            //then add the rest of first table
            for (String joinTableHeader : header) {
                if (!sameColumn.contains(joinTableHeader)) {
                    Column sameCol = (Column) getColumn(joinTableHeader);
                    joined.addColumn(joinTableHeader, sameCol);
                }
            }
            //then the rest of the second table
            for (String joinTableHeader : toJoin.header) {
                if (!sameColumn.contains(joinTableHeader)) {
                    Column sameCol = (Column) toJoin.getColumn(joinTableHeader);
                    joined.addColumn(joinTableHeader, sameCol);
                }
            }
            return joined;
        }
        //Iterating through the Column(s), get sameRow indices.
        for (String col : sameColumn) {
            Column targetColumnA = (Column) this.getColumn(col);
            Column targetColumnB = (Column) toJoin.getColumn(col);

            for (int i = 0; i < targetColumnA.size(); i += 1) {
                for (int j = 0; j < targetColumnB.size(); j += 1) {
                    if (targetColumnA.get(i).equals(targetColumnB.get(j))) {
                        sameRow.add(i);
                        sameRowB.add(j);
                    }
                }
            }
        }

        //add sameColumn first
        for (String colHeader : sameColumn) {
            Column sameCol = (Column) getColumn(colHeader);
            joined.addColumn(colHeader, sameCol.filter(sameRow));
        }

        //then add the rest of first table
        for (String joinTableHeader : header) {
            if (!sameColumn.contains(joinTableHeader)) {
                Column sameCol = (Column) getColumn(joinTableHeader);
                joined.addColumn(joinTableHeader, sameCol.filter(sameRow));
            }
        }
        //then the rest of the second table
        for (String joinTableHeader : toJoin.header) {
            if (!sameColumn.contains(joinTableHeader)) {
                Column sameCol = (Column) toJoin.getColumn(joinTableHeader);
                joined.addColumn(joinTableHeader, sameCol.filter(sameRowB));
            }
        }

        return joined;
    }

    public void filterRow(ArrayList<Integer> filter) {
        for (String head : header) {
            table.put(head, getColumn(head).filter(filter));
        }
    }

    private String roundHelper(Float f) {
        return String.format("%.3f", f);
    }

}

```



#### `Column.java`

```java
package db;
import java.util.ArrayList;
import java.lang.Integer;

/**
 * Created by simonmo on 2/18/17.
 * @author Simon Mo
 * Column class extends ArrayList data structure.
 * Use Generics.
 */
public class Column<V> extends ArrayList<V>{

    private ArrayList<V> column;
    private String type;

    /**
     * Empty constructor for Column, initialize an empty column.
     */
    public Column() {
        column = new ArrayList<>();
    }

    public boolean equalsColumn(Column c) {
       if (this.size() != c.size()) {
           return false;
       } else {
           for (int i = 0; i < this.size(); i++) {
               if (!this.get(i).equals(c.get(i))) {
                   return false;
               }
           }
       }
       return true;
    }

    /**
     * Constructor Recommendo
     */
    public Column(String typeOf) {
        type = typeOf;
        column = new ArrayList<>();
    }

    /**
     * Fill Column with...
     * @param size
     * @param item
     */
    public Column(int size, V item) {
        column = new ArrayList<>();
        for (int i = 0; i < size; i += 1) {
            column.add(item);
        }
    }
  
    public Column filter(ArrayList<Integer> indexs) {
        Column filtered = new Column();
        for (int index: indexs) {
            filtered.add(get(index));
        }
        return filtered;
    }

}
```

