# Chapter 7: Building a Calculator - Foundation for Database Development

## Introduction: From Arithmetic to SQL

Before diving into the complexities of database engines, we begin with a seemingly simple project: building a calculator. This foundational exercise introduces crucial concepts that directly translate to database query processing. When a database executes `SELECT price * quantity FROM orders WHERE total > 100`, it performs the same fundamental operations as our calculator: tokenizing input, parsing expressions, and evaluating results according to precedence rules.

The recursive descent parser we'll implement here forms the backbone of SQL query parsers in production databases. The expression evaluation engine becomes the foundation for computing derived columns, WHERE clause conditions, and aggregate functions. By mastering these concepts in a calculator context, we prepare ourselves for the more complex challenges of database development.

## Understanding the Problem Space

A calculator must handle mathematical expressions with proper operator precedence, parentheses, and error handling. Consider the expression `3 + 4 * 2`. A naive left-to-right evaluation would yield 14, but mathematical convention demands we get 11. This precedence handling is identical to how databases must parse `SELECT * FROM users WHERE age > 18 AND status = 'active'`, ensuring logical operators are evaluated in the correct order.

Our calculator will support:
- Basic arithmetic operators (+, -, *, /)
- Parentheses for grouping
- Floating-point numbers
- Proper error handling for division by zero and syntax errors
- Extensibility for future operations (essential for database function expansion)

## The Recursive Descent Parser Algorithm

The recursive descent parser is a top-down parsing technique that mirrors the grammatical structure of mathematical expressions. Each grammar rule becomes a function, and the parser recursively calls these functions to build an expression tree.

### Grammar Definition

Our calculator follows this grammar hierarchy:
```
expression → term (('+' | '-') term)*
term       → factor (('*' | '/') factor)*
factor     → number | '(' expression ')'
number     → digit+ ('.' digit+)?
```

This grammar naturally encodes operator precedence: expressions handle addition/subtraction (lowest precedence), terms handle multiplication/division (higher precedence), and factors handle numbers and parenthetical expressions (highest precedence).

### Core Parser Implementation

```python
class Token:
    def __init__(self, type, value, position):
        self.type = type      # NUMBER, PLUS, MINUS, MULTIPLY, DIVIDE, LPAREN, RPAREN, EOF
        self.value = value    # The actual value (number or operator)
        self.position = position

class Lexer:
    def __init__(self, text):
        self.text = text
        self.position = 0
        self.current_char = self.text[0] if text else None
    
    def advance(self):
        self.position += 1
        self.current_char = self.text[self.position] if self.position < len(self.text) else None
    
    def skip_whitespace(self):
        while self.current_char is not None and self.current_char.isspace():
            self.advance()
    
    def read_number(self):
        result = ''
        while self.current_char is not None and (self.current_char.isdigit() or self.current_char == '.'):
            result += self.current_char
            self.advance()
        return float(result)
    
    def get_next_token(self):
        while self.current_char is not None:
            if self.current_char.isspace():
                self.skip_whitespace()
                continue
            
            if self.current_char.isdigit():
                return Token('NUMBER', self.read_number(), self.position)
            
            if self.current_char == '+':
                self.advance()
                return Token('PLUS', '+', self.position)
            
            if self.current_char == '-':
                self.advance()
                return Token('MINUS', '-', self.position)
            
            if self.current_char == '*':
                self.advance()
                return Token('MULTIPLY', '*', self.position)
            
            if self.current_char == '/':
                self.advance()
                return Token('DIVIDE', '/', self.position)
            
            if self.current_char == '(':
                self.advance()
                return Token('LPAREN', '(', self.position)
            
            if self.current_char == ')':
                self.advance()
                return Token('RPAREN', ')', self.position)
            
            raise SyntaxError(f"Invalid character '{self.current_char}' at position {self.position}")
        
        return Token('EOF', None, self.position)

class Parser:
    def __init__(self, lexer):
        self.lexer = lexer
        self.current_token = self.lexer.get_next_token()
    
    def error(self, message):
        raise SyntaxError(f"{message} at position {self.current_token.position}")
    
    def eat(self, token_type):
        if self.current_token.type == token_type:
            self.current_token = self.lexer.get_next_token()
        else:
            self.error(f"Expected {token_type}, got {self.current_token.type}")
    
    def factor(self):
        """Parse a factor: number or parenthesized expression"""
        token = self.current_token
        
        if token.type == 'NUMBER':
            self.eat('NUMBER')
            return token.value
        
        elif token.type == 'LPAREN':
            self.eat('LPAREN')
            result = self.expression()
            self.eat('RPAREN')
            return result
        
        else:
            self.error("Expected number or opening parenthesis")
    
    def term(self):
        """Parse a term: factor followed by multiply/divide operations"""
        result = self.factor()
        
        while self.current_token.type in ('MULTIPLY', 'DIVIDE'):
            op = self.current_token.type
            self.eat(op)
            
            if op == 'MULTIPLY':
                result *= self.factor()
            elif op == 'DIVIDE':
                divisor = self.factor()
                if divisor == 0:
                    raise ValueError("Division by zero")
                result /= divisor
        
        return result
    
    def expression(self):
        """Parse an expression: term followed by add/subtract operations"""
        result = self.term()
        
        while self.current_token.type in ('PLUS', 'MINUS'):
            op = self.current_token.type
            self.eat(op)
            
            if op == 'PLUS':
                result += self.term()
            elif op == 'MINUS':
                result -= self.term()
        
        return result
    
    def parse(self):
        result = self.expression()
        if self.current_token.type != 'EOF':
            self.error("Unexpected token after expression")
        return result

class Calculator:
    def evaluate(self, expression):
        try:
            lexer = Lexer(expression)
            parser = Parser(lexer)
            return parser.parse()
        except (SyntaxError, ValueError) as e:
            return f"Error: {e}"

# Usage example
calc = Calculator()
print(calc.evaluate("3 + 4 * 2"))        # 11
print(calc.evaluate("(3 + 4) * 2"))      # 14
print(calc.evaluate("10 / 2 + 3"))       # 8.0
print(calc.evaluate("1 + 2 * 3 + 4"))    # 11
```

## The Bridge to Database Concepts

### Tokenization and SQL Parsing

The lexer's tokenization process directly parallels SQL parsing. Just as we identify NUMBER and PLUS tokens, SQL parsers identify SELECT, FROM, WHERE, and literal values. The position tracking we implement becomes crucial for reporting syntax errors in complex SQL statements.

Consider how our calculator handles `(3 + 4) * 2` versus how a database parses `SELECT (price + tax) * quantity FROM products`. The parenthetical grouping, operator precedence, and expression evaluation follow identical patterns.

### Abstract Syntax Trees

Our recursive descent parser implicitly builds an Abstract Syntax Tree (AST) through its recursive function calls. In a production database, this AST becomes the query execution plan. The precedence rules that ensure `3 + 4 * 2` evaluates correctly become the logical rules that ensure `WHERE age > 18 AND status = 'active'` applies conditions in the proper order.

### Expression Evaluation Engine

The evaluation logic in our `term()` and `expression()` methods forms the foundation of database expression evaluation. When a database processes `SELECT price * 1.08 AS total_with_tax`, it uses the same multiplication logic we implement here, extended to handle column references and type conversions.

## Error Handling and Recovery

Robust error handling in our calculator teaches patterns essential for database development:

```python
class EnhancedCalculator(Calculator):
    def evaluate_with_context(self, expression):
        try:
            lexer = Lexer(expression)
            parser = Parser(lexer)
            result = parser.parse()
            return {
                'success': True,
                'result': result,
                'error': None
            }
        except SyntaxError as e:
            return {
                'success': False,
                'result': None,
                'error': f"Syntax error: {e}",
                'suggestion': self.suggest_fix(expression, e)
            }
        except ValueError as e:
            return {
                'success': False,
                'result': None,
                'error': f"Runtime error: {e}"
            }
    
    def suggest_fix(self, expression, error):
        # Basic error recovery suggestions
        if "Expected RPAREN" in str(error):
            return "Check for missing closing parenthesis"
        elif "Division by zero" in str(error):
            return "Ensure divisor is not zero"
        return "Check expression syntax"
```

This error handling approach scales to database query validation, where we need to catch syntax errors, type mismatches, and constraint violations while providing meaningful feedback to users.

## Extending for Database Functionality

Our calculator's architecture naturally extends to support database-like operations:

```python
class DatabaseCalculator(Calculator):
    def __init__(self):
        super().__init__()
        self.variables = {}  # Symbol table for column references
        self.functions = {   # Built-in functions
            'ABS': abs,
            'ROUND': round,
            'MAX': max,
            'MIN': min
        }
    
    def set_variable(self, name, value):
        """Simulate column values in a database row"""
        self.variables[name] = value
    
    def evaluate_with_context(self, expression, row_data=None):
        """Evaluate expression with row context (like SQL WHERE clause)"""
        if row_data:
            self.variables.update(row_data)
        return self.evaluate(expression)

# This foundation supports expressions like:
# "price * quantity" where price and quantity are column values
# "ABS(balance - 100)" where ABS is a built-in function
```

## Performance Considerations

The recursive descent parser's performance characteristics directly impact database query performance. Understanding these patterns helps us optimize database expression evaluation:

### Time Complexity
- Tokenization: O(n) where n is expression length
- Parsing: O(n) for well-formed expressions
- Evaluation: O(n) for arithmetic expressions

### Space Complexity
- Call stack depth: O(d) where d is maximum nesting level
- Token storage: O(n) for the entire expression

### Optimization Strategies
```python
class OptimizedParser(Parser):
    def __init__(self, lexer):
        super().__init__(lexer)
        self.expression_cache = {}  # Memoization for repeated subexpressions
    
    def evaluate_cached(self, expression):
        if expression in self.expression_cache:
            return self.expression_cache[expression]
        
        result = self.evaluate(expression)
        self.expression_cache[expression] = result
        return result
```

## Testing and Validation

Comprehensive testing patterns learned here apply directly to database development:

```python
import unittest

class TestCalculator(unittest.TestCase):
    def setUp(self):
        self.calc = Calculator()
    
    def test_basic_arithmetic(self):
        self.assertEqual(self.calc.evaluate("2 + 3"), 5)
        self.assertEqual(self.calc.evaluate("10 - 4"), 6)
        self.assertEqual(self.calc.evaluate("3 * 4"), 12)
        self.assertEqual(self.calc.evaluate("15 / 3"), 5)
    
    def test_operator_precedence(self):
        self.assertEqual(self.calc.evaluate("2 + 3 * 4"), 14)
        self.assertEqual(self.calc.evaluate("2 * 3 + 4"), 10)
        self.assertEqual(self.calc.evaluate("10 - 6 / 2"), 7)
    
    def test_parentheses(self):
        self.assertEqual(self.calc.evaluate("(2 + 3) * 4"), 20)
        self.assertEqual(self.calc.evaluate("2 * (3 + 4)"), 14)
        self.assertEqual(self.calc.evaluate("((2 + 3) * 4)"), 20)
    
    def test_error_handling(self):
        with self.assertRaises(ValueError):
            self.calc.evaluate("10 / 0")
        
        result = self.calc.evaluate("2 +")
        self.assertTrue(result.startswith("Error:"))
    
    def test_complex_expressions(self):
        self.assertAlmostEqual(
            self.calc.evaluate("(10 + 5) * 2 - 8 / 4"), 
            28.0
        )
```

## Looking Ahead: Database Applications

The calculator we've built establishes patterns that directly translate to database development:

1. **Query Parsing**: SQL SELECT statements use identical parsing techniques
2. **Expression Evaluation**: Column computations and WHERE clauses use the same evaluation engine
3. **Error Handling**: SQL syntax errors and runtime errors follow similar patterns
4. **Optimization**: Expression caching and AST optimization apply to query plans
5. **Extensibility**: Adding functions mirrors implementing SQL built-in functions

In the next chapter, we'll extend these concepts to handle table schemas, row filtering, and join operations, building toward a complete database engine. The recursive descent parser becomes our SQL parser, the expression evaluator becomes our query executor, and the error handling becomes our constraint validation system.

The foundation is set. Every database query is, at its core, an expression to be parsed, validated, and evaluated—exactly what our calculator accomplishes.

## Summary

Building a calculator with a recursive descent parser provides essential foundations for database development. The tokenization, parsing, and evaluation patterns we've implemented here scale directly to SQL query processing. The error handling and optimization strategies prepare us for the complexities of database constraint validation and query optimization.

Most importantly, we've established a robust, extensible architecture that we'll build upon as we develop our database engine. The recursive descent parser isn't just an academic exercise—it's the production-ready foundation that powers database query parsing in systems used by millions of applications worldwide.
