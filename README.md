<!--
CÓMO USAR ESTE ARCHIVO
1. Reemplazá el README.md del repo por el contenido de abajo.
2. Completá [URL_STREAMLIT] en "Demo en vivo" una vez que despliegues el
   dashboard en Streamlit Community Cloud. [URL_RENDER] ya se completa solo
   apenas termine el deploy de la API en Render (avisame la URL y lo actualizo).
3. Borrá este comentario antes de commitear.
-->

<div align="center">

# 💳 CreditFlow

**¿Este cliente va a pagar a tiempo?** Pipeline de ML para anticipar riesgo de no pago crediticio, de punta a punta: features → modelo → API → monitoreo de drift.

![Python](https://img.shields.io/badge/Python-3776AB?style=flat-square&logo=python&logoColor=white)
![scikit-learn](https://img.shields.io/badge/scikit--learn-F7931E?style=flat-square&logo=scikitlearn&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-009688?style=flat-square&logo=fastapi&logoColor=white)
![Streamlit](https://img.shields.io/badge/Streamlit-FF4B4B?style=flat-square&logo=streamlit&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat-square&logo=docker&logoColor=white)
![CI/CD](https://img.shields.io/badge/CI%2FCD-2088FF?style=flat-square&logo=githubactions&logoColor=white)

</div>

<br>

<div align="center">
  <img src="src/curvas_evaluacion.png" alt="Curvas ROC y Precision-Recall del modelo final" width="700">
  <br><em>Curvas ROC / PR del modelo final. Reemplazar por una captura del dashboard de Streamlit si querés mostrar la UI (ver README de MovieTime como referencia de formato).</em>
</div>

<br>

## 📑 Contenido

- [El problema, en una escena](#-el-problema-en-una-escena)
- [La solución](#-la-solución)
- [Demo en vivo](#-demo-en-vivo)
- [Qué hace](#-qué-hace)
- [Resultados, sin maquillaje](#-resultados-sin-maquillaje)
- [Monitoreo de drift](#-monitoreo-de-drift)
- [Instalación y ejecución](#-instalación-y-ejecución)
- [Cómo está construido](#-cómo-está-construido)
- [Recursos útiles](#-recursos-útiles)
- [Licencia](#-licencia)
- [Autor](#-autor)

---

## 🎬 El problema, en una escena

Una entidad financiera aprueba un crédito. Seis meses después, ese cliente deja de pagar. La pérdida ya está hecha — la pregunta que importaba, *¿este cliente iba a pagar a tiempo?*, tenía que responderse **antes** de prestarle, no después.

Con una cartera donde solo ~5% de los créditos termina en mora, el reto no es solo predecir: es hacerlo sobre un desbalance fuerte, con datos que solo existen *antes* de la aprobación (no información que se genera durante la vida del crédito, que sería trampa).

## 💡 La solución

**CreditFlow** estima la probabilidad de que un cliente no pague a tiempo, usando únicamente información disponible al momento de la solicitud (historial, buró de crédito, ingresos declarados), y expone esa probabilidad — con una banda de riesgo y una razón explícita — vía API y un dashboard de monitoreo.

Lo construí como proyecto integrador de Machine Learning aplicado, y la parte de la que más orgulloso estoy no es el modelo: es haber **detectado y corregido un data leakage severo** antes de entregarlo. Con las variables filtradas (saldo de mora, saldo total — información que solo existe *durante* la vida del crédito), el modelo daba un PR-AUC perfecto de 1.0000. Sacándolas, cayó a 0.1367 — el número real, incómodo, pero honesto. Ese es el diferencial: no reportar la métrica que se ve mejor, sino la que sirve para decidir.

Lo que más aprendí: un modelo con recall de 25% sigue siendo útil — como **señal de apoyo dentro de un proceso más amplio**, nunca como aprobador automático — y decirlo explícitamente en la documentación es parte del trabajo, no un detalle a esconder.

---

## 🚀 Demo en vivo

- **API (FastAPI):** [URL_RENDER]/docs
- **Dashboard (Streamlit):** [URL_STREAMLIT]

[![Deploy to Render](https://render.com/images/deploy-to-render-button.svg)](https://render.com/deploy?repo=https://github.com/Juanma-Alvarado/CreditFlow)

> Ambos corren en planes gratuitos (Render + Streamlit Community Cloud) y pueden tardar 30-60s en "despertar" tras inactividad. El repo incluye `render.yaml`, así que el botón de arriba despliega la API con un solo click.

---

## ✨ Qué hace

| | |
|---|---|
| 🎯 | Predicción individual de probabilidad de no pago + banda de riesgo (`POST /predecir`) |
| 📦 | Scoring en lote desde un CSV (`POST /predecir-lote`) |
| 📊 | Dashboard con dos vistas: scoring interactivo y monitoreo de drift |
| 🔍 | Cada predicción devuelve el motivo, no solo el número — pensado para auditoría |
| 🩺 | Health check del modelo cargado (`GET /salud`) |
| 🔄 | CI que valida en cada push que el contenedor levanta y la API responde el valor exacto esperado |

## 📉 Resultados, sin maquillaje

Sobre 2.153 registros de test (102 incumplimientos reales), con **Random Forest + class weights balanceados**:

| Métrica | Valor | Lectura |
|---|---:|---|
| PR-AUC | 0,137 | 3,1× por encima del baseline aleatorio |
| ROC-AUC | 0,676 | — |
| Recall | 25,5% | Detecta ~1 de cada 4 incumplimientos |
| Precision | 10,3% | De cada 10 clientes marcados como riesgo, 1 lo es realmente |

**Variables más influyentes:** `puntaje_datacredito`, `huella_consulta`, `edad_cliente`, `ratio_ingresos_burodeclarado`, `capital_prestado`.

Es un modelo de **apoyo a la decisión**, no un aprobador automático — y decirlo así, en vez de vender un número inflado, es a propósito.

## 🩺 Monitoreo de drift

El dashboard compara un período de referencia (nov 2024–jun 2025) contra uno actual (jul 2025–abr 2026) con PSI por variable. Hallazgo real detectado: la tasa de mora bajó de 5,19% a 3,19%, pero el modelo seguía prediciendo 10-14% mensual — señal clara de que el modelo fue entrenado sobre una población más riesgosa que la actual, y necesita reentrenarse.

---

## 🔧 Instalación y ejecución

**Con Docker (recomendado):**

```bash
git clone https://github.com/Juanma-Alvarado/CreditFlow.git
cd CreditFlow
docker compose up --build
```

- API + docs: `http://localhost:8000/docs`
- Dashboard: `http://localhost:8501`

**Local:**

```bash
python -m venv venv && source venv/bin/activate   # Windows: venv\Scripts\activate
pip install -r requirements.txt
cd src
python ft_engineering.py
python model_training_evaluation.py
uvicorn model_deploy:app --reload      # otra terminal:
streamlit run app_streamlit.py
```

## 🏗️ Cómo está construido

| Capa | Tecnologías |
|---|---|
| Modelado | scikit-learn, XGBoost, Random Forest, imbalanced classes |
| Backend | FastAPI, joblib |
| Frontend | Streamlit |
| Infraestructura | Docker + docker-compose (una imagen, dos servicios), usuario sin privilegios |
| Deploy | Render (`render.yaml`, blueprint de un click) para la API, Streamlit Community Cloud para el dashboard |
| CI/CD | GitHub Actions — build, health check, valor exacto de predicción, validación de inputs |

**Endpoints principales:**

| Método | Ruta | Función |
|---|---|---|
| GET | `/salud` | Health check del modelo |
| GET | `/modelo` | Umbral, métricas de test, variables excluidas por leakage |
| POST | `/predecir` | Predicción individual |
| POST | `/predecir-lote` | Scoring en lote (CSV) |

## 📚 Recursos útiles

- [Documentación de scikit-learn — class_weight y desbalance](https://scikit-learn.org/stable/modules/generated/sklearn.ensemble.RandomForestClassifier.html)
- [Population Stability Index (PSI) para monitoreo de drift](https://scikit-learn.org/stable/modules/generated/sklearn.metrics.PrecisionRecallDisplay.html)
- [FastAPI — documentación oficial](https://fastapi.tiangolo.com/)

## 📄 Licencia

MIT — ver [LICENSE](LICENSE). Desarrollado como proyecto integrador de Machine Learning aplicado; el dataset y el caso de negocio son de uso académico.

## 👤 Autor

**Juan Manuel Alvarado** — Data Analyst Jr. & Data Scientist Jr.
[GitHub](https://github.com/Juanma-Alvarado) · [Portfolio](https://juanma-alvarado.github.io/PortafolioWeb)
