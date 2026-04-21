# Abel de Leon 
#Jason Gutierrez

import re
import json

# 1. ANALISIS LÉXICO

token_patron = {
    "KEYWORD": r'\b(inicio|fin|si|entonces|finsi|escribir|func|retornar|sino)\b',
    "IDENTIFIER": r'\b[a-zA-Z_][a-zA-Z0-9_]*\b',
    "NUMBER": r'\b\d+\b',
    "OPERATOR": r'[+\-*/=<>!]',
    "COMPARISON": r'==|!=|<=|>=', 
    "DELIMITER": r'[(),;{}]',
    "WHITESPACE": r'\s+',
}

def identificar_tokens(texto):
    patron_general = "|".join(f"(?P<{token}>{patron})" for token, patron in token_patron.items())
    patron_regex = re.compile(patron_general)
    
    tokens_encontrados = []
    for match in patron_regex.finditer(texto):
        for token, valor in match.groupdict().items():
            if valor is not None and token != "WHITESPACE":
                tokens_encontrados.append((token, valor))
    return tokens_encontrados


# 2. ANALISIS SINTÁCTICO (AST)

# ------------------------------------------------------------- Clases del AST ------------------------------------------------------------------------
class NodoAST:
    def to_dict(self):
        return {}

class NodoPrograma(NodoAST):
    def __init__(self, sentencias):
        self.sentencias = sentencias
    def to_dict(self):
        return {"Tipo": "Programa", "Sentencias": [s.to_dict() for s in self.sentencias]}

class NodoFuncion(NodoAST):
    def __init__(self, nombre, parametros, cuerpo):
        self.nombre = nombre
        self.parametros = parametros 
        self.cuerpo = cuerpo
    def to_dict(self):
        return {"Tipo": "Funcion", "Nombre": self.nombre, "Parametros": self.parametros, "Cuerpo": [s.to_dict() for s in self.cuerpo]}

class NodoAsignacion(NodoAST):
    def __init__(self, nombre, expresion):
        self.nombre = nombre
        self.expresion = expresion
    def to_dict(self):
        return {"Tipo": "Asignacion", "Variable": self.nombre, "Expresion": self.expresion.to_dict()}

class NodoOperacion(NodoAST):
    def __init__(self, izquierda, operador, derecha):
        self.izquierda = izquierda
        self.operador = operador
        self.derecha = derecha
    def to_dict(self):
        return {"Tipo": "Operacion", "Operador": self.operador, "Izquierda": self.izquierda.to_dict(), "Derecha": self.derecha.to_dict()}

class NodoCondicional(NodoAST):
    def __init__(self, condicion, sentencias_verdadero, sentencias_falso=None):
        self.condicion = condicion
        self.sentencias_verdadero = sentencias_verdadero
        self.sentencias_falso = sentencias_falso if sentencias_falso else []
    def to_dict(self):
        return {
            "Tipo": "Condicional", 
            "Condicion": self.condicion.to_dict(), 
            "CuerpoVerdadero": [s.to_dict() for s in self.sentencias_verdadero],
            "CuerpoFalso": [s.to_dict() for s in self.sentencias_falso]
        }

class NodoEscritura(NodoAST):
    def __init__(self, expresion):
        self.expresion = expresion
    def to_dict(self):
        return {"Tipo": "Escritura", "Expresion": self.expresion.to_dict()}

class NodoRetorno(NodoAST):
    def __init__(self, expresion):
        self.expresion = expresion
    def to_dict(self):
        return {"Tipo": "Retorno", "Expresion": self.expresion.to_dict()}

class NodoLlamada(NodoAST):
    def __init__(self, nombre, argumentos):
        self.nombre = nombre
        self.argumentos = argumentos
    def to_dict(self):
        return {"Tipo": "Llamada", "Nombre": self.nombre, "Argumentos": [a.to_dict() for a in self.argumentos]}

class NodoIdentificador(NodoAST):
    def __init__(self, nombre):
        self.nombre = nombre
    def to_dict(self):
        return {"Tipo": "Identificador", "Nombre": self.nombre}

class NodoNumero(NodoAST):
    def __init__(self, valor):
        self.valor = valor
    def to_dict(self):
        return {"Tipo": "Numero", "Valor": self.valor}


# -------------------------------------------------------------- Parser ----------------------------------------------------------

class Parser:
    def __init__(self, tokens):
        self.tokens = tokens
        self.pos = 0
    
    def obtener_token(self):
        return self.tokens[self.pos] if self.pos < len(self.tokens) else None
    
    def coincidir(self, tipo_esperado, valor_esperado=None):
        token = self.obtener_token()
        if token and token[0] == tipo_esperado:
            if valor_esperado is None or token[1] == valor_esperado:
                self.pos += 1
                return token
        return None
    
    def esperar(self, tipo_esperado, valor_esperado=None):
        token = self.coincidir(tipo_esperado, valor_esperado)
        if token is None:
            actual = self.obtener_token()
            raise SyntaxError(f"Error Sintáctico: Se esperaba {tipo_esperado}:{valor_esperado}, se encontró {actual}")
        return token
    
    def parsear(self):
        sentencias = []
        while self.obtener_token():
            if self.obtener_token()[1] == 'func':
                sentencias.append(self.funcion())
            else:
                self.esperar('KEYWORD', 'inicio')
                s = self.lista_sentencias(['fin'])
                self.esperar('KEYWORD', 'fin')
                sentencias.extend(s)
        return NodoPrograma(sentencias)

    def funcion(self):
        self.esperar('KEYWORD', 'func')
        nombre = self.esperar('IDENTIFIER')[1]
        self.esperar('DELIMITER', '(')
        params = self.lista_parametros()
        self.esperar('DELIMITER', ')')
        cuerpo = self.lista_sentencias(['finfunc'])
        self.esperar('KEYWORD', 'finfunc')
        return NodoFuncion(nombre, params, cuerpo)

    def lista_parametros(self):
        params = []
        if self.obtener_token() and self.obtener_token()[0] == 'IDENTIFIER':
            params.append(self.esperar('IDENTIFIER')[1])
            while self.coincidir('DELIMITER', ','):
                params.append(self.esperar('IDENTIFIER')[1])
        return params

    def lista_sentencias(self, terminadores):
        sentencias = []
        while self.obtener_token() and self.obtener_token()[1] not in terminadores:
            sentencias.append(self.sentencia())
        return sentencias

    def sentencia(self):
        token = self.obtener_token()
        if token[1] == 'si':
            return self.condicional()
        elif token[1] == 'escribir':
            return self.escritura()
        elif token[1] == 'retornar':
            return self.retorno()
        elif token[0] == 'IDENTIFIER':
            return self.asignacion()
        else:
            raise SyntaxError(f"Sentencia no reconocida: {token}")

    def asignacion(self):
        nombre = self.esperar('IDENTIFIER')[1]
        self.esperar('OPERATOR', '=')
        expr = self.expresion()
        return NodoAsignacion(nombre, expr)

    def expresion(self):
        izq = self.termino()
        while self.obtener_token() and self.obtener_token()[1] in ['+', '-']:
            op = self.esperar('OPERATOR')[1]
            der = self.termino()
            izq = NodoOperacion(izq, op, der)
        return izq

    def termino(self):
        izq = self.factor()
        while self.obtener_token() and self.obtener_token()[1] in ['*', '/']:
            op = self.esperar('OPERATOR')[1]
            der = self.factor()
            izq = NodoOperacion(izq, op, der)
        return izq

    def factor(self):
        token = self.obtener_token()
        if token[0] == 'NUMBER':
            self.pos += 1
            return NodoNumero(int(token[1]))
        elif token[0] == 'IDENTIFIER':
            self.pos += 1
            if self.obtener_token() and self.obtener_token()[1] == '(':
                self.pos += 1 
                args = self.lista_argumentos()
                self.esperar('DELIMITER', ')')
                return NodoLlamada(token[1], args)
            return NodoIdentificador(token[1])
        elif token[1] == '(':
            self.pos += 1
            expr = self.expresion()
            self.esperar('DELIMITER', ')')
            return expr
        else:
            raise SyntaxError(f"Factor no reconocido: {token}")

    def lista_argumentos(self):
        args = []
        if self.obtener_token() and self.obtener_token()[1] != ')':
            args.append(self.expresion())
            while self.coincidir('DELIMITER', ','):
                args.append(self.expresion())
        return args

    def condicional(self):
        self.esperar('KEYWORD', 'si')
        self.esperar('DELIMITER', '(')
        cond = self.condicion()
        self.esperar('DELIMITER', ')')
        self.esperar('KEYWORD', 'entonces')
        s_verdadero = self.lista_sentencias(['finsi', 'sino'])
        s_falso = []
        if self.coincidir('KEYWORD', 'sino'):
            s_falso = self.lista_sentencias(['finsi'])
        self.esperar('KEYWORD', 'finsi')
        return NodoCondicional(cond, s_verdadero, s_falso)

    def condicion(self):
        izq = self.expresion()
        op_token = self.obtener_token()
        if op_token and (op_token[0] == 'OPERATOR' or op_token[0] == 'COMPARISON'):
            op = op_token[1]
            self.pos += 1
            der = self.expresion()
            return NodoOperacion(izq, op, der)
        raise SyntaxError("Operador de comparación esperado")

    def escritura(self):
        self.esperar('KEYWORD', 'escribir')
        self.esperar('DELIMITER', '(')
        expr = self.expresion()
        self.esperar('DELIMITER', ')')
        return NodoEscritura(expr)

    def retorno(self):
        self.esperar('KEYWORD', 'retornar')
        expr = self.expresion()
        return NodoRetorno(expr)


# 3. ANALISIS SEMÁNTICO

class AnalizadorSemantico:
    def __init__(self):
        self.tabla_simbolos = {} 
        self.errores = []
        self.advertencias = []
        self.ambito_actual = "global"

    def analizar(self, nodo):
        if isinstance(nodo, NodoPrograma):
            for s in nodo.sentencias:
                if isinstance(s, NodoFuncion):
                    self.tabla_simbolos[s.nombre] = {'tipo': 'funcion', 'params': s.parametros, 'ambito': 'global'}
            
            for s in nodo.sentencias:
                self.visitar(s)
                
        elif isinstance(nodo, NodoFuncion):
            self.ambito_actual = nodo.nombre
            for p in nodo.parametros:
                self.tabla_simbolos[f"{self.ambito_actual}:{p}"] = {'tipo': '
