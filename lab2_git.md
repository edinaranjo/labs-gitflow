
# Git aplicado a una aplicación Streamlit para análisis de datos

## Objetivo

Desarrollar una aplicación en Streamlit que permita cargar archivos .csv, visualizar gráficos de análisis de datos y trabajar el flujo de control de versiones con Git, incluyendo:

- Creación de rama principal.
- Desarrollo de una funcionalidad en una rama adicional.
- Merge hacia la rama principal.
- Simulación y resolución de conflictos.
- Validación de código con pytest.


# Parte 1: Preparación del entorno de trabajo

a. Crear el directorio del proyecto

```{bash, eval=FALSE}
   mkdir laboratorio-streamlit-git
   cd laboratorio-streamlit-git
   mkdir src tests data
   touch app.py src/data_utils.py tests/test_data_utils.py requirements.txt README.md
   git init

```

Estructura esperada:

laboratorio-streamlit-git/
├── app.py
├── src/
│   └── data_utils.py
├── tests/
│   └── test_data_utils.py
├── data/
├── requirements.txt
└── README.md

---

b. Crear el entorno virtual

```{bash, eval=FALSE}

    python3 -m venv dash
    source dash/bin/activate

```

---

c. Verificar la versión de Python

```{bash, eval=FALSE}

    python3 -V

```

---

d. Editar el archivo requirements.txt

```{bash, eval=FALSE}

   streamlit
   pandas
   plotly
   pytest

```

\sectionline

e. Instalar dependencias

```{bash, eval=FALSE}

    # Actualizar pip:

    pip install --upgrade pip

    # Instalar dependencias:

    pip install -r requirements.txt

```

# Parte 2: Creacion de funciones de validación



```{bash, eval=FALSE}

   nano src/data_utils.py

```


```{python, eval=FALSE,python.reticulate = FALSE}

import pandas as pd


def validar_extension_csv(nombre_archivo: str) -> bool:
    """
    Valida que el archivo cargado tenga extensión .csv.
    """
    if not nombre_archivo:
        return False

    return nombre_archivo.lower().endswith(".csv")


def cargar_csv(archivo):
    """
    Carga un archivo CSV probando diferentes codificaciones comunes.
    """
    codificaciones = ["utf-8", "utf-8-sig", "latin1", "cp1252"]

    for encoding in codificaciones:
        try:
            df = pd.read_csv(archivo, encoding=encoding)

            if df.empty:
                raise ValueError("El archivo CSV está vacío.")

            return df

        except UnicodeDecodeError:
            archivo.seek(0)
            continue

        except pd.errors.EmptyDataError:
            raise ValueError("El archivo CSV no contiene datos.")

        except pd.errors.ParserError:
            raise ValueError("El archivo CSV tiene un formato inválido.")

    raise ValueError(
        "No se pudo leer el archivo. Verifique que sea un CSV válido y que su 
         codificación sea compatible."
    )

def obtener_columnas_numericas(df: pd.DataFrame):
    """
    Retorna las columnas numéricas del DataFrame.
    """
    return df.select_dtypes(include=["number"]).columns.tolist()


def obtener_columnas_categoricas(df: pd.DataFrame):
    """
    Retorna las columnas categóricas del DataFrame.
    """
    return df.select_dtypes(include=["object", "category"]).columns.tolist()


def filtrar_registros(df: pd.DataFrame, cantidad: int, posicion: str):
    """
    Retorna los primeros o últimos registros del DataFrame.
    """
    if cantidad <= 0:
        raise ValueError("La cantidad debe ser mayor que cero.")

    if posicion == "Inicio":
        return df.head(cantidad)

    if posicion == "Final":
        return df.tail(cantidad)

    raise ValueError("La posición seleccionada no es válida.")


```


# Parte 3: Crear la aplicación principal en Streamlit

```{python, eval=FALSE,python.reticulate = FALSE}

import streamlit as st
import plotly.express as px
import pandas as pd

from src.data_utils import (
    validar_extension_csv,
    cargar_csv,
    obtener_columnas_numericas,
    obtener_columnas_categoricas,
    filtrar_registros,
)


# ============================================================
# Configuración general de Streamlit
# ============================================================

st.set_page_config(
    page_title="Analizador Profesional de Datos",
    page_icon="📊",
    layout="wide"
)


# ============================================================
# Estilos visuales personalizados
# ============================================================

st.markdown(
    """
    <style>
    .main-title {
        font-size: 42px;
        font-weight: bold;
        color: #1F4E79;
        text-align: center;
    }
    .subtitle {
        font-size: 20px;
        text-align: center;
        color: #555555;
    }
    .info-box {
        background-color: #F0F6FC;
        padding: 18px;
        border-radius: 12px;
        border-left: 6px solid #1F77B4;
        font-size: 16px;
    }
    </style>
    """,
    unsafe_allow_html=True
)


# ============================================================
# Encabezado principal
# ============================================================

st.markdown('<p class="main-title">📊 Analizador Profesional de Datos</p>', 
unsafe_allow_html=True)

st.markdown(
    '<p class="subtitle">Carga un archivo CSV y genera visualizaciones automáticas 
    para análisis exploratorio.</p>',
    unsafe_allow_html=True
)

st.divider()


# ============================================================
# Carga del archivo CSV
# ============================================================

st.sidebar.header("📁 Carga de archivo")

archivo = st.sidebar.file_uploader(
    "Seleccione un archivo CSV",
    type=["csv"]
)

if archivo is None:
    st.info("Por favor, cargue un archivo con extensión `.csv` para iniciar el análisis.")
    st.stop()

if not validar_extension_csv(archivo.name):
    st.error("Formato inválido. Solo se permiten archivos con extensión `.csv`.")
    st.stop()

try:
    df = cargar_csv(archivo)

except ValueError as error:
    st.error(str(error))
    st.stop()


# ============================================================
# Validaciones generales del DataFrame
# ============================================================

if df.empty:
    st.error("El archivo cargado no contiene datos.")
    st.stop()

if df.shape[1] < 2:
    st.error("El archivo debe contener al menos dos columnas para realizar el análisis.")
    st.stop()

st.success("Archivo CSV cargado correctamente.")


# ============================================================
# Resumen del archivo
# ============================================================

st.markdown(
    f"""
    <div class="info-box">
        <b>Resumen del archivo:</b><br>
        Filas: {df.shape[0]}<br>
        Columnas: {df.shape[1]}
    </div>
    """,
    unsafe_allow_html=True
)

st.divider()


# ============================================================
# Optimización para gráficas
# Evita problemas de rendimiento y errores WebGL en máquinas virtuales
# ============================================================

MAX_REGISTROS_GRAFICA = 1000

if len(df) > MAX_REGISTROS_GRAFICA:
    st.info(
        f"El archivo contiene {len(df)} registros. "
        f"Para optimizar la visualización se usará una muestra aleatoria de "
        f"{MAX_REGISTROS_GRAFICA} registros en algunas gráficas."
    )

    df_grafica = df.sample(MAX_REGISTROS_GRAFICA, random_state=42)

else:
    df_grafica = df.copy()


# ============================================================
# Identificación de columnas
# ============================================================

columnas_numericas = obtener_columnas_numericas(df)
columnas_categoricas = obtener_columnas_categoricas(df)

if len(columnas_numericas) == 0:
    st.warning("El archivo no contiene columnas numéricas suficientes para generar gráficos.")
    st.stop()


# ============================================================
# Visualizaciones
# ============================================================

st.header("📈 Visualizaciones de análisis de datos")

col1, col2 = st.columns(2)


# ============================================================
# Gráfico de barras
# ============================================================

with col1:
    st.subheader("📊 Gráfico de barras")

    if columnas_categoricas:
        columna_categoria = st.selectbox(
            "Seleccione una columna categórica",
            columnas_categoricas,
            key="bar_categoria"
        )

        columna_valor = st.selectbox(
            "Seleccione una columna numérica",
            columnas_numericas,
            key="bar_valor"
        )

        try:
            datos_barra = (
                df.groupby(columna_categoria, as_index=False)[columna_valor]
                .sum()
                .sort_values(by=columna_valor, ascending=False)
            )

            fig_bar = px.bar(
                datos_barra,
                x=columna_categoria,
                y=columna_valor,
                title=f"Suma de {columna_valor} por {columna_categoria}"
            )

            st.plotly_chart(fig_bar, use_container_width=True)

        except Exception as error:
            st.error(f"No se pudo generar el gráfico de barras: {error}")

    else:
        st.warning("No existen columnas categóricas para generar el gráfico de barras.")


# ============================================================
# Gráfico de líneas
# ============================================================

with col2:
    st.subheader("📈 Gráfico de líneas")

    columna_linea = st.selectbox(
        "Seleccione columna numérica para el gráfico de líneas",
        columnas_numericas,
        key="line_valor"
    )

    try:
        if "Order Date" in df.columns:
            df_linea = df.copy()

            df_linea["Order Date"] = pd.to_datetime(
                df_linea["Order Date"],
                errors="coerce"
            )

            df_linea = df_linea.dropna(subset=["Order Date"])

            datos_linea = (
                df_linea
                .groupby(pd.Grouper(key="Order Date", freq="ME"))[columna_linea]
                .sum()
                .reset_index()
            )

            fig_line = px.line(
                datos_linea,
                x="Order Date",
                y=columna_linea,
                title=f"Evolución mensual de {columna_linea}"
            )

        else:
            fig_line = px.line(
                df_grafica,
                y=columna_linea,
                title=f"Evolución de {columna_linea}"
            )

        st.plotly_chart(fig_line, use_container_width=True)

    except Exception as error:
        st.error(f"No se pudo generar el gráfico de líneas: {error}")


# ============================================================
# Gráfico de dispersión
# ============================================================

st.subheader("🔎 Gráfico de dispersión")

if len(columnas_numericas) >= 2:
    eje_x = st.selectbox(
        "Seleccione eje X",
        columnas_numericas,
        key="scatter_x"
    )

    eje_y = st.selectbox(
        "Seleccione eje Y",
        columnas_numericas,
        key="scatter_y"
    )

    try:
        fig_scatter = px.scatter(
            df_grafica,
            x=eje_x,
            y=eje_y,
            title=f"Relación entre {eje_x} y {eje_y}",
            render_mode="svg"
        )

        st.plotly_chart(fig_scatter, use_container_width=True)

    except Exception as error:
        st.error(f"No se pudo generar el gráfico de dispersión: {error}")

else:
    st.warning("Se requieren al menos dos columnas numéricas para el gráfico de dispersión.")


```


# Parte 4: Primer commit en la rama principal

a. Crear el archivo .gitignore

```{bash, eval=FALSE}

   dash/
   __pycache__/
   .pytest_cache/
   *.pyc
   .env
   .DS_Store
```

\sectionline

b. Relizar el staging y commit

```{bash, eval=FALSE}

    git status
    git add .
    git commit -m "Crear aplicacion Streamlit con carga CSV y graficas"
```

\sectionline

c. Ejecutar la aplicación

```{bash, eval=FALSE}

    streamlit run app.py
```



# Parte 5: Crear rama para nueva funcionalidad

La nueva funcionalidad será visualizar el archivo cargado en forma de tabla y filtrar registros desde el inicio o desde el final. 


a. Crear la rama

```{bash, eval=FALSE}

    git checkout -b feature/tabla-datos
```

b. Editar app.py y agregar la nueva funcionalidad al final

```{python, eval=FALSE,python.reticulate = FALSE}

# ============================================================
# Visualización tabular
# ============================================================

st.divider()

st.header("🧾 Visualización tabular de datos")

cantidad = st.number_input(
    "Seleccione la cantidad de registros a visualizar",
    min_value=1,
    max_value=len(df),
    value=min(10, len(df))
)

posicion = st.radio(
    "Seleccione desde dónde desea visualizar los datos",
    ["Inicio", "Final"],
    horizontal=True
)

try:
    df_filtrado = filtrar_registros(df, cantidad, posicion)
    st.dataframe(df_filtrado, use_container_width=True)

except ValueError as error:
    st.error(str(error))

```

c. Commit de la nueva funcionalidad

```{bash, eval=FALSE}

   git status
   git add .
   git commit -m "Agregar visualizacion tabular con filtro de registros"
```

d. Merge hacia la rama principal

```{bash, eval=FALSE}

  git checkout master
  git merge feature/tabla-datos
```

e. Verificar 

```{bash, eval=FALSE}

  git log --oneline --graph --all
```


# Parte 6: Simulación de conflicto

a. Crear una rama nueva

```{bash, eval=FALSE}

  git checkout -b feature/cambio-titulo
```

b. Editar en app.py esta línea


```{bash, eval=FALSE}

  st.markdown('<p class="main-title">📊 Analizador Profesional de Datos</p>', 
  unsafe_allow_html=True)
```

  
      Cambiarla por:
  
  
```{bash, eval=FALSE}

  st.markdown('<p class="main-title">📊 Plataforma Profesional de Analítica CSV</p>',
  unsafe_allow_html=True)
```

c. Guardar y hacer commit


  
```{bash, eval=FALSE}

    git add app.py
    git commit -m "Modificar titulo principal de la aplicacion"
```


d. Volver a main/master

```{bash, eval=FALSE}

   git checkout main
```

e. Editar la misma línea en app.py, pero con otro texto

```{bash, eval=FALSE}

   st.markdown('<p class="main-title">📊 Dashboard Profesional de Datos CSV</p>', 
   unsafe_allow_html=True)
```

f. Guardar y hacer commit

```{bash, eval=FALSE}

    git add app.py
    git commit -m "Actualizar titulo del dashboard principal"
```

g. Intentar merge

```{bash, eval=FALSE}

   git merge feature/cambio-titulo
```

  Aparecerá un conflicto similar a:
  
```{bash, eval=FALSE}

    CONFLICT (content): Merge conflict in app.py
    Automatic merge failed; fix conflicts and then commit the result.
```


# Parte 7: Resolver el conflicto

a. Abrir app.py. Git mostrará algo como:

```{bash, eval=FALSE}

  <<<<<<< HEAD
st.markdown('<p class="main-title">📊 Dashboard Profesional de Datos CSV</p>', 
unsafe_allow_html=True)
=======
st.markdown('<p class="main-title">📊 Plataforma Profesional de Analítica CSV</p>', 
unsafe_allow_html=True)
>>>>>>> feature/cambio-titulo
```

b. Dejar una versión final

```{bash, eval=FALSE}

  st.markdown('<p class="main-title">📊 Plataforma Profesional de Análisis de Datos CSV</p>', 
  unsafe_allow_html=True)
```

c. Finalizar resolución

```{bash, eval=FALSE}

    git add app.py
    git commit -m "Resolver conflicto en titulo principal"
```


d. Verificar historial

```{bash, eval=FALSE}

    git log --oneline --graph --all
```

# Parte 8: Crear pruebas con pytest

a. Crear archivo

```{bash, eval=FALSE}

    nano tests/test_data_utils.py
```

```{python, eval=FALSE,python.reticulate = FALSE}

import pandas as pd
import pytest

from src.data_utils import (
    validar_extension_csv,
    obtener_columnas_numericas,
    obtener_columnas_categoricas,
    filtrar_registros,
)


@pytest.fixture
def penguins_df():
    """
    DataFrame de prueba basado en la estructura del dataset Palmer Penguins.
    """
    return pd.DataFrame({
        "species": ["Adelie", "Adelie", "Gentoo", "Chinstrap", "Gentoo"],
        "island": ["Torgersen", "Torgersen", "Biscoe", "Dream", "Biscoe"],
        "bill_length_mm": [39.1, 39.5, 46.1, 50.0, 45.2],
        "bill_depth_mm": [18.7, 17.4, 13.2, 19.5, 14.8],
        "flipper_length_mm": [181, 186, 211, 196, 212],
        "body_mass_g": [3750, 3800, 4500, 3900, 5200],
        "sex": ["male", "female", "female", "male", "female"],
        "year": [2007, 2007, 2008, 2009, 2009],
    })


def test_validar_extension_csv_correcta():
    assert validar_extension_csv("penguins.csv") is True


def test_validar_extension_csv_mayuscula():
    assert validar_extension_csv("penguins.CSV") is True


def test_validar_extension_csv_incorrecta():
    assert validar_extension_csv("penguins.xlsx") is False


def test_columnas_numericas_penguins(penguins_df):
    columnas = obtener_columnas_numericas(penguins_df)

    assert "bill_length_mm" in columnas
    assert "bill_depth_mm" in columnas
    assert "flipper_length_mm" in columnas
    assert "body_mass_g" in columnas
    assert "year" in columnas

    assert "species" not in columnas
    assert "island" not in columnas
    assert "sex" not in columnas


def test_columnas_categoricas_penguins(penguins_df):
    columnas = obtener_columnas_categoricas(penguins_df)

    assert "species" in columnas
    assert "island" in columnas
    assert "sex" in columnas

    assert "bill_length_mm" not in columnas
    assert "body_mass_g" not in columnas
    assert "year" not in columnas


def test_filtrar_registros_inicio_penguins(penguins_df):
    resultado = filtrar_registros(penguins_df, 2, "Inicio")

    assert len(resultado) == 2
    assert resultado.iloc[0]["species"] == "Adelie"
    assert resultado.iloc[1]["species"] == "Adelie"


def test_filtrar_registros_final_penguins(penguins_df):
    resultado = filtrar_registros(penguins_df, 2, "Final")

    assert len(resultado) == 2
    assert resultado.iloc[0]["species"] == "Chinstrap"
    assert resultado.iloc[1]["species"] == "Gentoo"


def test_filtrar_registros_cantidad_invalida_penguins(penguins_df):
    with pytest.raises(ValueError):
        filtrar_registros(penguins_df, 0, "Inicio")


def test_filtrar_registros_posicion_invalida_penguins(penguins_df):
    with pytest.raises(ValueError):
        filtrar_registros(penguins_df, 3, "Centro")

```

b. Ejecutar desde la raiz del proyecto

```{bash, eval=FALSE}
    touch src/__init__.py
    python -m pytest -v
```

c. Registrar el cambio en Git

```{bash, eval=FALSE}
    git add src/__init__.py tests/test_data_utils.py
    git commit -m "Corregir importacion del paquete src en pruebas"
```
