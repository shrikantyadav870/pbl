import re

# Token types
INTEGER, PLUS, MINUS, MUL, DIV, LPAREN, RPAREN, EOF = (
    'INTEGER', 'PLUS', 'MINUS', 'MUL', 'DIV', 'LPAREN', 'RPAREN', 'EOF'
)


class Token:
    def __init__(self, type_, value):
        self.type = type_
        self.value = value

    def __repr__(self):
        return f"Token({self.type}, {repr(self.value)})"


class Lexer:
    def __init__(self, text):
        self.text = text
        self.pos = 0
        self.current_char = self.text[self.pos] if self.text else None

    def advance(self):
        self.pos += 1
        if self.pos >= len(self.text):
            self.current_char = None
        else:
            self.current_char = self.text[self.pos]

    def skip_whitespace(self):
        while self.current_char and self.current_char.isspace():
            self.advance()

    def integer(self):
        result = ''
        while self.current_char and self.current_char.isdigit():
            result += self.current_char
            self.advance()
        return int(result)

    def get_next_token(self):
        while self.current_char:
            if self.current_char.isspace():
                self.skip_whitespace()
                continue
            if self.current_char.isdigit():
                return Token(INTEGER, self.integer())
            if self.current_char == '+':
                self.advance()
                return Token(PLUS, '+')
            if self.current_char == '-':
                self.advance()
                return Token(MINUS, '-')
            if self.current_char == '*':
                self.advance()
                return Token(MUL, '*')
            if self.current_char == '/':
                self.advance()
                return Token(DIV, '/')
            if self.current_char == '(':
                self.advance()
                return Token(LPAREN, '(')
            if self.current_char == ')':
                self.advance()
                return Token(RPAREN, ')')
            raise Exception(f"Invalid character: {self.current_char}")
        return Token(EOF, None)


# AST node types
class AST:
    pass


class BinOp(AST):
    def __init__(self, left, op, right):
        self.left = left
        self.op = op
        self.right = right


class Num(AST):
    def __init__(self, token):
        self.token = token
        self.value = token.value


# Recursive descent parser
class Parser:
    def __init__(self, lexer):
        self.lexer = lexer
        self.current_token = self.lexer.get_next_token()

    def error(self):
        raise Exception('Invalid syntax')

    def eat(self, token_type):
        if self.current_token.type == token_type:
            self.current_token = self.lexer.get_next_token()
        else:
            self.error()

    def factor(self):
        token = self.current_token
        if token.type == INTEGER:
            self.eat(INTEGER)
            return Num(token)
        elif token.type == LPAREN:
            self.eat(LPAREN)
            node = self.expr()
            self.eat(RPAREN)
            return node
        elif token.type == PLUS:
            self.eat(PLUS)
            node = self.factor()
            return node  # unary plus just returns the factor
        elif token.type == MINUS:
            self.eat(MINUS)
            node = self.factor()
            return BinOp(Num(Token(INTEGER, 0)), Token(MINUS, '-'), node)  # unary minus as 0 - expr
        else:
            self.error()

    def term(self):
        node = self.factor()
        while self.current_token.type in (MUL, DIV):
            token = self.current_token
            if token.type == MUL:
                self.eat(MUL)
            else:
                self.eat(DIV)
            node = BinOp(node, token, self.factor())
        return node

    def expr(self):
        node = self.term()
        while self.current_token.type in (PLUS, MINUS):
            token = self.current_token
            if token.type == PLUS:
                self.eat(PLUS)
            else:
                self.eat(MINUS)
            node = BinOp(node, token, self.term())
        return node

    def parse(self):
        return self.expr()


# Code Generator for stack machine
class CodeGenerator:
    def __init__(self):
        self.instructions = []

    def visit(self, node):
        if isinstance(node, Num):
            self.instructions.append(f"PUSH {node.value}")
        elif isinstance(node, BinOp):
            self.visit(node.left)
            self.visit(node.right)
            if node.op.type == PLUS:
                self.instructions.append('ADD')
            elif node.op.type == MINUS:
                self.instructions.append('SUB')
            elif node.op.type == MUL:
                self.instructions.append('MUL')
            elif node.op.type == DIV:
                self.instructions.append('DIV')
            else:
                raise Exception(f"Unknown operator: {node.op.type}")
        else:
            raise Exception(f"Unknown node type: {type(node)}")

    def generate(self, node):
        self.visit(node)
        return self.instructions


# Simple stack machine to execute the code
class VM:
    def __init__(self, instructions):
        self.instructions = instructions
        self.stack = []

    def run(self):
        for inst in self.instructions:
            parts = inst.split()
            op = parts[0]
            if op == 'PUSH':
                self.stack.append(int(parts[1]))
            elif op == 'ADD':
                b = self.stack.pop()
                a = self.stack.pop()
                self.stack.append(a + b)
            elif op == 'SUB':
                b = self.stack.pop()
                a = self.stack.pop()
                self.stack.append(a - b)
            elif op == 'MUL':
                b = self.stack.pop()
                a = self.stack.pop()
                self.stack.append(a * b)
            elif op == 'DIV':
                b = self.stack.pop()
                a = self.stack.pop()
                self.stack.append(a // b)  # integer division
            else:
                raise Exception(f"Unknown instruction: {op}")
        return self.stack.pop() if self.stack else None


def compile_expression(text):
    lexer = Lexer(text)
    parser = Parser(lexer)
    ast = parser.parse()
    codegen = CodeGenerator()
    instructions = codegen.generate(ast)
    return instructions


def main():
    print("Mini compiler for arithmetic expressions")
    expr = input("Enter expression: ")
    try:
        instructions = compile_expression(expr)
        print("Generated instructions:")
        for inst in instructions:
            print(inst)
        vm = VM(instructions)
        result = vm.run()
        print("Evaluation result:", result)
    except Exception as e:
        print("Error:", e)


if __name__ == '__main__':
    main()

