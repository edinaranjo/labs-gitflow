# Laboratorio GitFlow: Despliegue y actualización de una base de datos SQLite3

## Escenario

Una empresa desarrolla una pequeña aplicación para registrar dispositivos tecnológicos. La primera versión permite consultar dispositivos desde una base SQLite3. Luego se implementará una nueva funcionalidad para insertar dispositivos, se preparará una versión estable y finalmente se corregirá un error crítico mediante una rama `hotfix`.

---

# 1. Preparación del proyecto

```bash
mkdir sqlite-gitflow-lab
cd sqlite-gitflow-lab
git init
```

Configurar GitFlow:

```bash
git flow init
```

Aceptar los valores por defecto:

```text
Branch name for production releases: master
Branch name for "next release" development: develop
Feature branches: feature/
Release branches: release/
Hotfix branches: hotfix/
```

---

# 2. Crear aplicación inicial

Crear archivo:

```bash
nano app.py
```

Contenido:

```python
import sqlite3

DB_NAME = "devices.db"

def init_db():
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()

    cursor.execute("""
    CREATE TABLE IF NOT EXISTS devices (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        hostname TEXT NOT NULL,
        ip_address TEXT NOT NULL
    )
    """)

    conn.commit()
    conn.close()

def list_devices():
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()

    cursor.execute("SELECT id, hostname, ip_address FROM devices")
    devices = cursor.fetchall()

    conn.close()
    return devices

if __name__ == "__main__":
    init_db()
    print("Dispositivos registrados:")
    for device in list_devices():
        print(device)
```

Crear `.gitignore`:

```bash
nano .gitignore
```

Contenido:

```gitignore
__pycache__/
*.pyc
devices.db
devops/
.env
```

Primer commit:

```bash
git add .
git commit -m "Crear aplicacion inicial con SQLite"
```

---

# 3. Crear rama develop

Si GitFlow no la creó automáticamente:

```bash
git checkout -b develop
```

O verificar ramas:

```bash
git branch
```

La rama `develop` representa el trabajo en integración. No es producción; es donde se acumulan las funcionalidades antes de preparar una versión.

---

# 4. Crear una funcionalidad con feature

Objetivo: agregar inserción de dispositivos.

```bash
git flow feature start add-device
```

Esto crea y cambia automáticamente a:

```text
feature/add-device
```

Modificar `app.py`:

```python
import sqlite3

DB_NAME = "devices.db"

def init_db():
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()

    cursor.execute("""
    CREATE TABLE IF NOT EXISTS devices (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        hostname TEXT NOT NULL,
        ip_address TEXT NOT NULL
    )
    """)

    conn.commit()
    conn.close()

def add_device(hostname, ip_address):
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()

    cursor.execute(
        "INSERT INTO devices (hostname, ip_address) VALUES (?, ?)",
        (hostname, ip_address)
    )

    conn.commit()
    conn.close()

def list_devices():
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()

    cursor.execute("SELECT id, hostname, ip_address FROM devices")
    devices = cursor.fetchall()

    conn.close()
    return devices

if __name__ == "__main__":
    init_db()

    add_device("R1", "192.168.10.1")
    add_device("SW1", "192.168.10.2")

    print("Dispositivos registrados:")
    for device in list_devices():
        print(device)
```

Probar:

```bash
python3 app.py
```

Confirmar cambios:

```bash
git add app.py
git commit -m "Agregar funcion para insertar dispositivos"
```

Finalizar feature:

```bash
git flow feature finish add-device
```

Esto fusiona `feature/add-device` hacia `develop` y elimina la rama feature.

Verificar:

```bash
git branch
git log --oneline --graph --all
```

---

# 5. Crear release

Cuando `develop` ya contiene funcionalidades listas para estabilizar, se crea una rama `release`.

```bash
git flow release start 1.0.0
```

Esto crea:

```text
release/1.0.0
```

En esta rama no se deberían agregar grandes funcionalidades. Solo ajustes menores, documentación, versión y preparación para producción.

Crear archivo `VERSION`:

```bash
echo "1.0.0" > VERSION
```

Crear archivo `README.md`:

```bash
nano README.md
```

Contenido:

````markdown
# SQLite GitFlow Lab

Aplicación básica en Python para gestionar dispositivos usando SQLite3.

## Ejecución

```bash
python3 app.py
```
````

## Versión

1.0.0

Confirmar cambios:

```bash
git add VERSION README.md
git commit -m "Preparar release 1.0.0"
```


Finalizar release:

```bash
git flow release finish 1.0.0
```

GitFlow hará lo siguiente:

```text
release/1.0.0 -> master
release/1.0.0 -> develop
crea tag 1.0.0
elimina release/1.0.0
```

Verificar:

```bash
git tag
git log --oneline --graph --all
```

---

# 6. Simular error en producción

Supongamos que la versión `1.0.0` ya está en producción, pero se detecta un error: la aplicación inserta dispositivos duplicados cada vez que se ejecuta.

Este problema no debe corregirse en `develop`, porque afecta directamente a producción. Se debe crear una rama `hotfix`.

```bash
git flow hotfix start 1.0.1
```

Esto crea:

```text
hotfix/1.0.1
```

Modificar `app.py` para evitar duplicados.

Cambiar la tabla:

```python
cursor.execute("""
CREATE TABLE IF NOT EXISTS devices (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    hostname TEXT NOT NULL UNIQUE,
    ip_address TEXT NOT NULL
)
""")
```

Modificar `add_device`:

```python
def add_device(hostname, ip_address):
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()

    cursor.execute(
        "INSERT OR IGNORE INTO devices (hostname, ip_address) VALUES (?, ?)",
        (hostname, ip_address)
    )

    conn.commit()
    conn.close()
```

Actualizar `VERSION`:

```bash
echo "1.0.1" > VERSION
```

Probar:

```bash
rm -f devices.db
python3 app.py
python3 app.py
```

Confirmar que no se insertan duplicados.

Commit:

```bash
git add app.py VERSION
git commit -m "Corregir insercion duplicada de dispositivos"
```

Finalizar hotfix:

```bash
git flow hotfix finish 1.0.1
```

GitFlow hará lo siguiente:

```text
hotfix/1.0.1 -> master
hotfix/1.0.1 -> develop
crea tag 1.0.1
elimina hotfix/1.0.1
```

Verificar:

```bash
git tag
git log --oneline --graph --all
```

---

# 7. Flujo completo de ramas

```text
master
  └── Código estable en producción

develop
  └── Integración de funcionalidades

feature/add-device
  └── Desarrollo de nueva funcionalidad

release/1.0.0
  └── Preparación de versión estable

hotfix/1.0.1
  └── Corrección urgente sobre producción
```

---

# 8. Comandos principales usados

```bash
git flow init
git flow feature start add-device
git flow feature finish add-device
git flow release start 1.0.0
git flow release finish 1.0.0
git flow hotfix start 1.0.1
git flow hotfix finish 1.0.1
```

---

# 9. Interpretación DevOps

Este laboratorio representa un flujo profesional porque separa claramente:

* Desarrollo activo: `develop`
* Nuevas funcionalidades: `feature/*`
* Preparación de versión: `release/*`
* Producción estable: `master`
* Correcciones críticas: `hotfix/*`

En un entorno DevOps real, este flujo se complementaría con:

* Pruebas automáticas con `pytest`
* Validación de migraciones de base de datos
* Pipeline CI/CD
* Pull Requests
* Revisión de código
* Versionamiento semántico
* Despliegue automatizado

---

# 10. Buenas prácticas

* No desarrollar directamente sobre `master`.
* No agregar funcionalidades nuevas en una rama `release`.
* Usar `hotfix` solo para errores críticos en producción.
* Ejecutar pruebas antes de finalizar cada rama.
* Versionar scripts de base de datos, no la base `.db`.
* Excluir archivos generados mediante `.gitignore`.
* Usar tags para marcar versiones productivas.

---

# Resultado esperado

Al finalizar, el participante habrá usado GitFlow de forma completa para:

1. Crear una aplicación inicial.
2. Desarrollar una nueva funcionalidad.
3. Integrar cambios en `develop`.
4. Preparar una versión estable.
5. Publicar una release.
6. Corregir un error crítico con hotfix.
7. Comprender la relación entre GitFlow, control de versiones y DevOps.

