# Laboratorio GitFlow + MLOps: Predicción de precios de viviendas con regresión lineal.

## 1. Escenario

Una empresa desea construir un modelo de regresión lineal para predecir precios de viviendas usando un dataset con variables como:

* `price`
* `area`
* `bedrooms`
* `bathrooms`
* `stories`
* `mainroad`
* `guestroom`
* `basement`
* `hotwaterheating`
* `airconditioning`
* `parking`
* `prefarea`
* `furnishingstatus`

El equipo usará GitFlow para organizar el desarrollo del modelo.

---

# 2. Crear el proyecto

```bash
mkdir housing-gitflow-ml
cd housing-gitflow-ml
git init
git flow init
```

Aceptar valores por defecto:

```text
master
develop
feature/
release/
hotfix/
```

---

# 3. Estructura inicial

```bash
mkdir data models tests src
touch README.md requirements.txt .gitignore
```

Copiar el dataset:

```bash
cp ~/Descargas/Housing.csv data/Housing.csv
```

Estructura:

```text
housing-gitflow-ml/
├── data/
│   └── Housing.csv
├── models/
├── src/
├── tests/
├── README.md
├── requirements.txt
└── .gitignore
```

---

# 4. Dependencias

Archivo `requirements.txt`:

```text
pandas
scikit-learn
joblib
pytest
```

Instalar:

```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

Archivo `.gitignore`:

```gitignore
venv/
__pycache__/
*.pyc
models/*.pkl
.env
```

En este laboratorio sí se versiona `Housing.csv` por fines académicos, pero en producción los datasets grandes normalmente se gestionan con herramientas como DVC o repositorios de artefactos.

---

# 5. Código base en master

Crear archivo:

```bash
nano src/train_model.py
```

Contenido inicial:

```python
import pandas as pd
import joblib
from pathlib import Path
from sklearn.model_selection import train_test_split
from sklearn.linear_model import LinearRegression
from sklearn.metrics import r2_score, mean_absolute_error

DATA_PATH = Path("data/Housing.csv")
MODEL_PATH = Path("models/housing_linear_model.pkl")


def load_data():
    return pd.read_csv(DATA_PATH)


def preprocess_data(df):
    df = df.copy()

    categorical_columns = [
        "mainroad",
        "guestroom",
        "basement",
        "hotwaterheating",
        "airconditioning",
        "prefarea",
        "furnishingstatus",
    ]

    df = pd.get_dummies(
        df,
        columns=categorical_columns,
        drop_first=True
    )

    X = df.drop("price", axis=1)
    y = df["price"]

    return X, y


def train_model():
    df = load_data()
    X, y = preprocess_data(df)

    X_train, X_test, y_train, y_test = train_test_split(
        X,
        y,
        test_size=0.2,
        random_state=42
    )

    model = LinearRegression()
    model.fit(X_train, y_train)

    predictions = model.predict(X_test)

    r2 = r2_score(y_test, predictions)
    mae = mean_absolute_error(y_test, predictions)

    MODEL_PATH.parent.mkdir(exist_ok=True)
    joblib.dump(model, MODEL_PATH)

    print("Modelo entrenado correctamente")
    print(f"R2: {r2:.4f}")
    print(f"MAE: {mae:.2f}")
    print(f"Modelo guardado en: {MODEL_PATH}")


if __name__ == "__main__":
    train_model()
```

Ejecutar:

```bash
python3 src/train_model.py
```

Primer commit:

```bash
git add .
git commit -m "Crear modelo base de regresion lineal para precios de viviendas"
```

---

# 6. Rama develop

Cambiar a `develop`:

```bash
git checkout develop
```

Si no existe:

```bash
git checkout -b develop
```

La rama `develop` será la rama de integración. Aquí llegarán las funcionalidades antes de preparar una versión estable.

---

# 7. Feature 1: validación del dataset

Crear rama:

```bash
git flow feature start data-validation
```

Git crea:

```text
feature/data-validation
```

Modificar `src/train_model.py` agregando:

```python
def validate_data(df):
    required_columns = [
        "price",
        "area",
        "bedrooms",
        "bathrooms",
        "stories",
        "mainroad",
        "guestroom",
        "basement",
        "hotwaterheating",
        "airconditioning",
        "parking",
        "prefarea",
        "furnishingstatus",
    ]

    for column in required_columns:
        if column not in df.columns:
            raise ValueError(f"Falta la columna requerida: {column}")

    if df.isnull().sum().sum() > 0:
        raise ValueError("El dataset contiene valores nulos.")

    if (df["price"] <= 0).any():
        raise ValueError("Existen precios inválidos.")

    if (df["area"] <= 0).any():
        raise ValueError("Existen áreas inválidas.")
```

Actualizar `train_model()`:

```python
def train_model():
    df = load_data()
    validate_data(df)

    X, y = preprocess_data(df)
```

Probar:

```bash
python3 src/train_model.py
```

Commit:

```bash
git add src/train_model.py
git commit -m "Agregar validacion del dataset Housing"
```

Finalizar feature:

```bash
git flow feature finish data-validation
```

Esto fusiona `feature/data-validation` hacia `develop`.

---

# 8. Feature 2: script de predicción

Crear nueva rama:

```bash
git flow feature start predict-script
```

Crear archivo:

```bash
nano src/predict.py
```

Contenido:

```python
import joblib
import pandas as pd
from pathlib import Path

MODEL_PATH = Path("models/housing_linear_model.pkl")


def predict_price(input_data):
    if not MODEL_PATH.exists():
        raise FileNotFoundError(
            "El modelo no existe. Ejecute primero src/train_model.py"
        )

    model = joblib.load(MODEL_PATH)

    df = pd.DataFrame([input_data])
    prediction = model.predict(df)

    return round(prediction[0], 2)


if __name__ == "__main__":
    sample_house = {
        "area": 7420,
        "bedrooms": 4,
        "bathrooms": 2,
        "stories": 3,
        "parking": 2,
        "mainroad_yes": True,
        "guestroom_yes": False,
        "basement_yes": False,
        "hotwaterheating_yes": False,
        "airconditioning_yes": True,
        "prefarea_yes": True,
        "furnishingstatus_semi-furnished": False,
        "furnishingstatus_unfurnished": False,
    }

    price = predict_price(sample_house)
    print(f"Precio estimado: {price}")
```

Probar:

```bash
python3 src/train_model.py
python3 src/predict.py
```

Commit:

```bash
git add src/predict.py
git commit -m "Agregar script de prediccion de precios"
```

Finalizar feature:

```bash
git flow feature finish predict-script
```

---

# 9. Feature 3: pruebas automatizadas

Crear rama:

```bash
git flow feature start tests
```

Crear archivo:

```bash
nano tests/test_model.py
```

Contenido:

```python
from src.train_model import load_data, validate_data, preprocess_data


def test_dataset_loads_correctly():
    df = load_data()
    assert not df.empty
    assert "price" in df.columns


def test_dataset_validation():
    df = load_data()
    validate_data(df)


def test_preprocess_data():
    df = load_data()
    X, y = preprocess_data(df)

    assert "price" not in X.columns
    assert len(X) == len(y)
    assert X.shape[1] > 0
```

Ejecutar:

```bash
pytest -v
```

Commit:

```bash
git add tests/test_model.py
git commit -m "Agregar pruebas automatizadas del modelo"
```

Finalizar feature:

```bash
git flow feature finish tests
```

---

# 10. Release 1.0.0

Crear release:

```bash
git flow release start 1.0.0
```

En esta rama no se agregan funcionalidades nuevas. Solo se estabiliza la versión.

Crear archivo `VERSION`:

```bash
echo "1.0.0" > VERSION
```

Actualizar `README.md`:

````markdown
# Housing Price Prediction

Proyecto MLOps básico para predecir precios de viviendas mediante regresión lineal.

## Dataset

Se utiliza `Housing.csv`, que contiene variables estructurales y categóricas de viviendas.

## Entrenamiento

```bash
python3 src/train_model.py
````

## Predicción

```bash
python3 src/predict.py
```

## Pruebas

```bash
pytest -v
```

## Flujo GitFlow

* master: producción
* develop: integración
* feature/*: funcionalidades
* release/*: estabilización
* hotfix/*: correcciones críticas

````

Validar:

```bash
python3 src/train_model.py
pytest -v
````

Commit:

```bash
git add README.md VERSION
git commit -m "Preparar release 1.0.0 del modelo Housing"
```

Finalizar release:

```bash
git flow release finish 1.0.0
```

GitFlow hará:

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

# 11. Hotfix 1.0.1: corrección crítica

Supongamos que en producción se detecta un problema: el script `predict.py` falla si el orden o número de columnas no coincide exactamente con el modelo entrenado.

Esto es crítico porque puede generar errores en producción.

Crear hotfix:

```bash
git flow hotfix start 1.0.1
```

Modificar `src/train_model.py` para guardar también las columnas:

```python
FEATURES_PATH = Path("models/features.pkl")
```

Dentro de `train_model()` después de guardar el modelo:

```python
joblib.dump(list(X.columns), FEATURES_PATH)
```

Modificar `src/predict.py`:

```python
import joblib
import pandas as pd
from pathlib import Path

MODEL_PATH = Path("models/housing_linear_model.pkl")
FEATURES_PATH = Path("models/features.pkl")


def predict_price(input_data):
    if not MODEL_PATH.exists():
        raise FileNotFoundError(
            "El modelo no existe. Ejecute primero src/train_model.py"
        )

    if not FEATURES_PATH.exists():
        raise FileNotFoundError(
            "No existe el archivo de columnas del modelo."
        )

    model = joblib.load(MODEL_PATH)
    feature_columns = joblib.load(FEATURES_PATH)

    df = pd.DataFrame([input_data])
    df = df.reindex(columns=feature_columns, fill_value=0)

    prediction = model.predict(df)

    return round(prediction[0], 2)


if __name__ == "__main__":
    sample_house = {
        "area": 7420,
        "bedrooms": 4,
        "bathrooms": 2,
        "stories": 3,
        "parking": 2,
        "mainroad_yes": True,
        "airconditioning_yes": True,
        "prefarea_yes": True,
    }

    price = predict_price(sample_house)
    print(f"Precio estimado: {price}")
```

Actualizar versión:

```bash
echo "1.0.1" > VERSION
```

Validar:

```bash
python3 src/train_model.py
python3 src/predict.py
pytest -v
```

Commit:

```bash
git add src/train_model.py src/predict.py VERSION
git commit -m "Corregir alineacion de columnas en prediccion"
```

Finalizar hotfix:

```bash
git flow hotfix finish 1.0.1
```

GitFlow fusiona el arreglo en:

```text
master
develop
```

y crea el tag:

```text
1.0.1
```

---

# 12. Resumen de ramas usadas

## master

Contiene versiones estables listas para producción:

```text
1.0.0
1.0.1
```

## develop

Integra funcionalidades antes de producción:

```text
data-validation
predict-script
tests
```

## feature/data-validation

Se agregó validación de columnas, valores nulos, precios y áreas inválidas.

## feature/predict-script

Se creó un script para usar el modelo entrenado y generar predicciones.

## feature/tests

Se añadieron pruebas automatizadas para validar carga, estructura y preprocesamiento.

## release/1.0.0

Se preparó la primera versión estable del modelo.

## hotfix/1.0.1

Se corrigió un error crítico relacionado con la alineación de columnas al predecir.

---

# 13. Comandos GitFlow principales

```bash
git flow init

git flow feature start data-validation
git flow feature finish data-validation

git flow feature start predict-script
git flow feature finish predict-script

git flow feature start tests
git flow feature finish tests

git flow release start 1.0.0
git flow release finish 1.0.0

git flow hotfix start 1.0.1
git flow hotfix finish 1.0.1
```

---

# 14. Interpretación DevOps/MLOps

Este laboratorio representa un flujo MLOps básico porque combina:

* Git como sistema de control de versiones.
* GitFlow como estrategia de ramas.
* Regresión lineal como modelo predictivo.
* Dataset versionado para práctica académica.
* Pruebas automatizadas con pytest.
* Release para estabilizar una versión.
* Hotfix para corregir fallos críticos en producción.

