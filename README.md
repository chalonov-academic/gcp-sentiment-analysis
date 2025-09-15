# 🔍 Análisis de Sentimientos con Google Cloud Platform

## 📋 Descripción del Proyecto

Implementación completa de un sistema de análisis de sentimientos usando servicios gestionados de Google Cloud Platform. Este proyecto demuestra conceptos fundamentales de cloud computing incluyendo serverless computing, machine learning como servicio, y análisis de big data.

## 🎯 Objetivos Académicos

- Demostrar arquitectura **serverless** con Cloud Functions
- Implementar **Machine Learning como Servicio** con Natural Language AI
- Utilizar **Big Data warehouse** con BigQuery
- Mostrar **integración de APIs** entre servicios cloud
- Aplicar conceptos de **elasticidad** y **pay-per-use**

## 🏗️ Arquitectura del Sistema

```
┌─────────────┐    ┌──────────────┐    ┌─────────────────┐    ┌───────────┐
│   Cliente   │──▶│ Cloud        │───▶│ Natural         │──▶│ BigQuery  │
│ (Postman/   │    │ Function     │    │ Language AI     │    │ Dataset   │
│  Script)    │    │ (Python)     │    │ API             │    │           │
└─────────────┘    └──────────────┘    └─────────────────┘    └───────────┘
       │                   │                      │                   │
       │                   │                      │                   ▼
       │                   │                      │            ┌──────────────┐
       │                   │                      │            │ Data Studio  │
       │                   │                      │            │ Dashboard    │
       └───────────────────┼──────────────────────┼────────────┴──────────────┘
                           │                      │
                    ┌──────▼──────┐        ┌──────▼──────┐
                    │ Cloud       │        │ ML Model    │
                    │ Monitoring  │        │ Analytics   │
                    └─────────────┘        └─────────────┘
```

## 🛠️ Tecnologías Utilizadas

| Servicio | Propósito | Costo |
|----------|-----------|-------|
| **Cloud Functions** | Procesamiento serverless | Gratis (2M requests/mes) |
| **Natural Language AI** | Análisis ML de sentimientos | $1.00/1K requests |
| **BigQuery** | Data warehouse y analytics | Gratis (1TB queries/mes) |
| **Cloud Storage** | Almacenamiento de archivos | Gratis (5GB) |
| **Cloud IAM** | Gestión de permisos | Gratis |

## 🚀 Configuración e Instalación

### Prerrequisitos

- Cuenta de Google Cloud Platform
- Proyecto GCP creado
- Cloud Shell o gcloud CLI instalado
- Créditos de facturación habilitados

### 1. Configuración Inicial

```bash
# Configurar proyecto
export PROJECT_ID="sentiment-analysis-demo-2024"
gcloud config set project $PROJECT_ID

# Habilitar APIs necesarias
gcloud services enable cloudfunctions.googleapis.com
gcloud services enable language.googleapis.com
gcloud services enable bigquery.googleapis.com
gcloud services enable storage.googleapis.com
```

### 2. Crear Infraestructura BigQuery

```sql
-- Crear dataset
CREATE SCHEMA `sentiment-analysis-demo-2024.sentiment_data`
OPTIONS(
  description="Dataset para análisis de sentimientos",
  location="US"
);

-- Crear tabla principal
CREATE TABLE `sentiment-analysis-demo-2024.sentiment_data.reviews` (
  product_id STRING,
  review_text STRING,
  sentiment_score FLOAT64,
  sentiment_magnitude FLOAT64,
  sentiment_classification STRING,
  processed_timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP()
);
```

### 3. Desplegar Cloud Function

```bash
# Crear directorio del proyecto
mkdir sentiment-function && cd sentiment-function

# Crear requirements.txt
cat > requirements.txt << 'EOF'
google-cloud-language==2.9.1
google-cloud-bigquery==3.11.4
functions-framework==3.4.0
EOF

# Crear main.py (ver código completo abajo)
# ...

# Desplegar función
gcloud functions deploy analyze-sentiment \
    --runtime python39 \
    --trigger-http \
    --allow-unauthenticated \
    --entry-point analyze_sentiment \
    --memory 256MB \
    --timeout 60s \
    --region us-central1
```

## 📝 Código de la Cloud Function

**Crear archivo `main.py`:**

```python
import functions_framework
from google.cloud import language_v1
from google.cloud import bigquery
import json
import logging
from datetime import datetime

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

@functions_framework.http
def analyze_sentiment(request):
    """
    Cloud Function que analiza el sentimiento de una review
    y almacena el resultado en BigQuery
    """
    
    # Configurar CORS
    if request.method == 'OPTIONS':
        headers = {
            'Access-Control-Allow-Origin': '*',
            'Access-Control-Allow-Methods': 'POST',
            'Access-Control-Allow-Headers': 'Content-Type',
        }
        return ('', 204, headers)
    
    headers = {'Access-Control-Allow-Origin': '*'}
    
    try:
        # Obtener datos del request
        request_json = request.get_json(silent=True)
        
        if not request_json:
            return json.dumps({'error': 'No JSON data provided'}), 400, headers
        
        review_text = request_json.get('review_text', '')
        product_id = request_json.get('product_id', 'UNKNOWN')
        
        if not review_text:
            return json.dumps({'error': 'review_text is required'}), 400, headers
        
        logger.info(f"Procesando review para producto {product_id}")
        
        # Analizar sentimiento con Natural Language AI
        client = language_v1.LanguageServiceClient()
        document = language_v1.Document(
            content=review_text,
            type_=language_v1.Document.Type.PLAIN_TEXT
        )
        
        response = client.analyze_sentiment(request={'document': document})
        sentiment = response.document_sentiment
        
        # Clasificar sentimiento
        if sentiment.score > 0.1:
            classification = 'Positivo'
        elif sentiment.score < -0.1:
            classification = 'Negativo'
        else:
            classification = 'Neutral'
        
        # Guardar en BigQuery
        bigquery_client = bigquery.Client()
        table_id = f"{bigquery_client.project}.sentiment_data.reviews"
        
        row = {
            "product_id": product_id,
            "review_text": review_text,
            "sentiment_score": sentiment.score,
            "sentiment_magnitude": sentiment.magnitude,
            "sentiment_classification": classification,
            "processed_timestamp": datetime.utcnow().isoformat()
        }
        
        errors = bigquery_client.insert_rows_json(table_id, [row])
        
        if errors:
            logger.error(f"Error en BigQuery: {errors}")
            return json.dumps({'error': 'Database error'}), 500, headers
        
        # Respuesta exitosa
        result = {
            'product_id': product_id,
            'sentiment_score': round(sentiment.score, 3),
            'sentiment_magnitude': round(sentiment.magnitude, 3),
            'sentiment_classification': classification,
            'message': f'Review clasificada como: {classification}'
        }
        
        logger.info(f"Análisis completado: {classification}")
        return json.dumps(result), 200, headers
        
    except Exception as e:
        logger.error(f"Error: {str(e)}")
        return json.dumps({'error': f'Error interno: {str(e)}'}), 500, headers
```

## 🧪 Casos de Prueba

### Usando cURL

```bash
# URL de la función desplegada
FUNCTION_URL="https://us-central1-tokyo-comfort-472221-b8.cloudfunctions.net/analyze-sentiment"

# Test 1: Review positiva
curl -X POST "$FUNCTION_URL" \
    -H "Content-Type: application/json" \
    -d '{"product_id": "TEST001", "review_text": "Este producto es excelente, me encanta mucho"}'

# Test 2: Review negativa
curl -X POST "$FUNCTION_URL" \
    -H "Content-Type: application/json" \
    -d '{"product_id": "TEST002", "review_text": "Terrible producto, muy mala calidad"}'
```

### Usando Postman

**Método:** POST  
**URL:** `https://us-central1-tokyo-comfort-472221-b8.cloudfunctions.net/analyze-sentiment`  
**Headers:** `Content-Type: application/json`

**Body (JSON):**
```json
{
    "product_id": "PROD001",
    "review_text": "Este producto es absolutamente increíble, superó todas mis expectativas."
}
```

**Respuesta esperada:**
```json
{
    "product_id": "PROD001",
    "sentiment_score": 0.9,
    "sentiment_magnitude": 0.8,
    "sentiment_classification": "Positivo",
    "message": "Review clasificada como: Positivo"
}
```

## 📊 Consultas de Análisis en BigQuery

### 1. Resumen General

```sql
SELECT 
    COUNT(*) as total_reviews,
    ROUND(AVG(sentiment_score), 3) as avg_sentiment_global,
    COUNTIF(sentiment_classification = "Positivo") as total_positivas,
    COUNTIF(sentiment_classification = "Negativo") as total_negativas,
    COUNTIF(sentiment_classification = "Neutral") as total_neutrales,
    ROUND(COUNTIF(sentiment_classification = "Positivo") * 100.0 / COUNT(*), 1) as porcentaje_satisfaccion
FROM `YOUR_PROJECT_ID.sentiment_data.reviews`
```

### 2. Análisis por Producto

```sql
SELECT 
    product_id,
    COUNT(*) as total_reviews,
    ROUND(AVG(sentiment_score), 3) as avg_sentiment,
    COUNTIF(sentiment_classification = "Positivo") as positivas,
    COUNTIF(sentiment_classification = "Negativo") as negativas,
    ROUND(COUNTIF(sentiment_classification = "Positivo") * 100.0 / COUNT(*), 1) as porcentaje_positivo,
    CASE 
        WHEN AVG(sentiment_score) > 0.3 THEN "⭐⭐⭐⭐⭐ Excelente"
        WHEN AVG(sentiment_score) > 0.1 THEN "⭐⭐⭐⭐ Bueno" 
        WHEN AVG(sentiment_score) > -0.1 THEN "⭐⭐⭐ Regular"
        ELSE "⭐⭐ Malo"
    END as rating
FROM `YOUR_PROJECT_ID.sentiment_data.reviews` 
GROUP BY product_id
ORDER BY avg_sentiment DESC
```

### 3. Reviews Extremas

```sql
(SELECT 
    "MÁS POSITIVAS" as tipo,
    product_id,
    ROUND(sentiment_score, 3) as score,
    LEFT(review_text, 70) as review_preview
FROM `YOUR_PROJECT_ID.sentiment_data.reviews` 
ORDER BY sentiment_score DESC 
LIMIT 3)
UNION ALL
(SELECT 
    "MÁS NEGATIVAS" as tipo,
    product_id,
    ROUND(sentiment_score, 3) as score,
    LEFT(review_text, 70) as review_preview
FROM `YOUR_PROJECT_ID.sentiment_data.reviews` 
ORDER BY sentiment_score ASC 
LIMIT 3)
ORDER BY tipo DESC, score DESC
```

## 📂 Datos de Prueba

### Archivo `reviews_sample.csv`

```csv
product_id,review_text
PROD001,Este producto es increíble, superó todas mis expectativas. Lo recomiendo totalmente.
PROD001,Muy mala calidad, se rompió al segundo día de uso. No lo compren.
PROD001,Producto decente, cumple su función pero nada extraordinario.
PROD002,Excelente relación calidad-precio. Muy satisfecho con la compra.
PROD002,El peor producto que he comprado en mi vida. Servicio al cliente terrible.
PROD002,Está bien, aunque tardó mucho en llegar.
PROD003,Perfecto para lo que necesitaba. Llegó rápido y bien empacado.
PROD003,No funciona como se describe en la página. Muy decepcionado.
PROD003,Cumple con lo prometido. Buen producto.
PROD004,Increíble calidad de construcción. Vale cada peso pagado.
PROD004,Regular, esperaba algo mejor por el precio.
PROD004,Fantástico producto, lo volvería a comprar sin dudarlo.
```

## 🐍 Scripts de Procesamiento

### Script 1: Procesamiento Individual

**Archivo: `test_sentiment.py`**

```python
import requests
import json

def test_sentiment_analysis():
    """Script para probar análisis individual de reviews"""
    
    # URL de tu función desplegada (cambiar por la tuya)
    url = "https://us-central1-YOUR_PROJECT_ID.cloudfunctions.net/analyze-sentiment"
    
    # Reviews de prueba
    test_reviews = [
        {"product_id": "TEST001", "review_text": "Este producto es excelente, me encanta mucho"},
        {"product_id": "TEST002", "review_text": "Terrible producto, muy mala calidad"},
        {"product_id": "TEST003", "review_text": "Producto promedio, cumple su función"}
    ]
    
    print("🚀 INICIANDO PRUEBAS DE ANÁLISIS DE SENTIMIENTOS")
    print("=" * 60)
    
    for i, review in enumerate(test_reviews):
        print(f"[{i+1}] Procesando: {review['product_id']}")
        print(f"    Text: {review['review_text']}")
        
        try:
            response = requests.post(url, json=review, timeout=30)
            if response.status_code == 200:
                result = response.json()
                emoji = "😊" if result['sentiment_classification'] == 'Positivo' else "😞" if result['sentiment_classification'] == 'Negativo' else "😐"
                print(f"    ✅ {emoji} {result['sentiment_classification']} (Score: {result['sentiment_score']:.3f})")
            else:
                print(f"    ❌ Error: {response.status_code}")
        except Exception as e:
            print(f"    ❌ Error: {e}")
        
        print()

if __name__ == "__main__":
    test_sentiment_analysis()
```

### Script 2: Procesamiento Masivo

**Archivo: `process_csv.py`**

```python
import pandas as pd
import requests
import time

def process_csv_reviews(csv_file, function_url):
    """
    Procesa un archivo CSV completo de reviews
    """
    
    print(f"📁 Cargando reviews desde {csv_file}")
    df = pd.read_csv(csv_file)
    
    print(f"📊 Procesando {len(df)} reviews...")
    print("=" * 60)
    
    successful = 0
    failed = 0
    
    for index, row in df.iterrows():
        payload = {
            'product_id': str(row['product_id']),
            'review_text': str(row['review_text'])
        }
        
        print(f"[{index+1:2d}/{len(df)}] {row['product_id']}: {str(row['review_text'])[:50]}...")
        
        try:
            response = requests.post(function_url, json=payload, timeout=30)
            if response.status_code == 200:
                result = response.json()
                emoji = "😊" if result['sentiment_classification'] == 'Positivo' else "😞" if result['sentiment_classification'] == 'Negativo' else "😐"
                print(f"         ✅ {emoji} {result['sentiment_classification']} (Score: {result['sentiment_score']:.3f})")
                successful += 1
            else:
                print(f"         ❌ Error HTTP: {response.status_code}")
                failed += 1
        except Exception as e:
            print(f"         ❌ Error: {e}")
            failed += 1
        
        time.sleep(1)  # Pausa entre requests
    
    print(f"\n📊 RESUMEN:")
    print(f"✅ Exitosas: {successful}")
    print(f"❌ Fallidas: {failed}")
    print(f"📈 Total procesadas: {successful}")

if __name__ == "__main__":
    # Cambiar por tu URL de función
    FUNCTION_URL = "https://us-central1-YOUR_PROJECT_ID.cloudfunctions.net/analyze-sentiment"
    process_csv_reviews('reviews_sample.csv', FUNCTION_URL)
```

## 🔧 Comandos de Verificación

### Verificar despliegue
```bash
# Obtener URL de la función
gcloud functions describe analyze-sentiment --region=us-central1 --format="value(httpsTrigger.url)"

# Verificar logs
gcloud functions logs read analyze-sentiment --region=us-central1 --limit=50

# Verificar tabla en BigQuery
bq ls sentiment_data
bq show sentiment_data.reviews
```

### Monitoreo y depuración
```bash
# Ver métricas de la función
gcloud functions describe analyze-sentiment --region=us-central1

# Consultar datos recientes en BigQuery
bq query --use_legacy_sql=false \
'SELECT COUNT(*) as total FROM `YOUR_PROJECT_ID.sentiment_data.reviews`'

# Ver logs en tiempo real
gcloud functions logs tail analyze-sentiment --region=us-central1
```

## ⚠️ Personalización Necesaria

### Variables a Cambiar

Antes de ejecutar, debes cambiar las siguientes variables por las tuyas:

1. **Project ID:** Reemplaza `sentiment-analysis-demo-YYYY` con tu Project ID único
2. **URLs de Function:** Reemplaza `YOUR_PROJECT_ID` en los scripts
3. **Region:** Cambiar `us-central1` si prefieres otra región

### Checklist de Personalización

```bash
# 1. En todos los comandos SQL de BigQuery:
# Cambiar: sentiment-analysis-demo-YYYY
# Por: tu-project-id-real

# 2. En scripts de Python:
# Cambiar: YOUR_PROJECT_ID
# Por: tu-project-id-real

# 3. Verificar region:
# us-central1 (recomendado para latencia)
# us-east1 (alternativa)
```

## 🚀 Pasos de Replicación Completa

### Paso 1: Configuración inicial
```bash
export PROJECT_ID="tu-project-id-unico"
gcloud config set project $PROJECT_ID
```

### Paso 2: Habilitar servicios
```bash
gcloud services enable cloudfunctions.googleapis.com language.googleapis.com bigquery.googleapis.com
```

### Paso 3: Crear infraestructura
```bash
# BigQuery dataset
bq mk --dataset --location=US $PROJECT_ID:sentiment_data

# BigQuery table
bq mk --table $PROJECT_ID:sentiment_data.reviews \
    product_id:STRING,review_text:STRING,sentiment_score:FLOAT,sentiment_magnitude:FLOAT,sentiment_classification:STRING,processed_timestamp:TIMESTAMP
```

### Paso 4: Código y despliegue
```bash
# Crear archivos del proyecto
mkdir sentiment-function && cd sentiment-function

# Copiar código de main.py y requirements.txt del README

# Desplegar función
gcloud functions deploy analyze-sentiment \
    --runtime python39 --trigger-http --allow-unauthenticated \
    --entry-point analyze_sentiment --region us-central1
```

### Paso 5: Probar funcionamiento
```bash
# Obtener URL
FUNCTION_URL=$(gcloud functions describe analyze-sentiment --region=us-central1 --format="value(httpsTrigger.url)")

# Prueba rápida
curl -X POST "$FUNCTION_URL" \
    -H "Content-Type: application/json" \
    -d '{"product_id": "TEST", "review_text": "Este producto es excelente"}'
```

### Paso 6: Procesar datos de prueba
```bash
# Crear CSV con datos del README
# Ejecutar script de procesamiento
python3 process_csv.py
```

### Paso 7: Verificar en BigQuery
```bash
# Consultar resultados
bq query --use_legacy_sql=false \
'SELECT COUNT(*) FROM `'$PROJECT_ID'.sentiment_data.reviews`'
```

### Métricas del Proyecto
- ✅ **18+ reviews procesadas** exitosamente
- ✅ **4 productos analizados** con diferentes sentimientos
- ✅ **Precisión del modelo** validada manualmente
- ✅ **Latencia promedio** < 2 segundos por request
- ✅ **Disponibilidad** 99.9% (SLA de Cloud Functions)

### Distribución de Sentimientos
| Producto | Total Reviews | Positivas | Negativas | Rating |
|----------|---------------|-----------|-----------|---------|
| PROD003 | 3 | 66.7% | 33.3% | ⭐⭐⭐⭐ Excelente |
| PROD004 | 3 | 66.7% | 33.3% | ⭐⭐⭐⭐ Excelente |
| PROD001 | 3 | 50.0% | 50.0% | ⭐⭐⭐ Bueno |
| PROD002 | 3 | 50.0% | 50.0% | ⭐⭐⭐ Bueno |

## 💰 Análisis de Costos

### Costos Reales del Proyecto
- **Natural Language AI:** $0.04 USD (40 requests)
- **Cloud Functions:** $0.00 USD (dentro del free tier)
- **BigQuery:** $0.00 USD (dentro del free tier)
- **Cloud Storage:** $0.00 USD (dentro del free tier)

**TOTAL: $0.04 USD** (4 centavos)

### Proyección de Costos por Escala
| Escala | Reviews/mes | Costo Mensual |
|--------|-------------|---------------|
| Demo Académico | 100 | $0.10 USD |
| Proyecto Pequeño | 1,000 | $1.00 USD |
| Aplicación Media | 10,000 | $10.00 USD |
| Aplicación Grande | 100,000 | $107.00 USD |

## 🔧 Conceptos de Cloud Computing Demostrados

### 1. **Elasticidad y Auto-scaling**
- Cloud Functions escala automáticamente desde 0 a N instancias
- No hay gestión de servidores ni configuración de capacidad

### 2. **Pay-per-use**
- Solo pagas por requests procesados, no por tiempo de inactividad
- Facturación granular por millisegundo de ejecución

### 3. **Servicios Gestionados**
- No hay que gestionar ML models, bases de datos, o infraestructura
- Actualizaciones y parches automáticos

### 4. **Integración de APIs**
- Arquitectura basada en microservicios
- Comunicación stateless entre componentes

### 5. **Análisis en Tiempo Real**
- Processing inmediato de datos
- Consultas analíticas instantáneas en BigQuery

## 🚀 Escalabilidad y Extensiones

### Posibles Mejoras
1. **Streaming con Pub/Sub:** Análisis en tiempo real de miles de reviews
2. **Batch Processing:** Procesamiento masivo con Dataflow
3. **Cache con Memorystore:** Reducir latencia para consultas frecuentes
4. **API Gateway:** Gestión avanzada de APIs con autenticación
5. **ML Personalizado:** Modelos custom con Vertex AI

### Integración con Otros Servicios
- **Cloud Storage:** Para datasets grandes
- **Firestore:** Para aplicaciones web en tiempo real
- **Cloud Scheduler:** Para procesamiento programado
- **Looker Studio:** Para dashboards avanzados

## 📚 Referencias y Recursos

### Documentación Oficial
- [Cloud Functions](https://cloud.google.com/functions/docs)
- [Natural Language AI](https://cloud.google.com/natural-language/docs)
- [BigQuery](https://cloud.google.com/bigquery/docs)

### Tutoriales Complementarios
- [Cloud Architecture Center](https://cloud.google.com/architecture)
- [Serverless Best Practices](https://cloud.google.com/serverless)
- [ML on Google Cloud](https://cloud.google.com/ml-engine/docs)

## 👥 Contribuciones y Créditos

**Desarrollado para:** Curso de Cloud Computing  
**Tecnologías:** Google Cloud Platform  
**Lenguajes:** Python, SQL, JavaScript  
**Herramientas:** Cloud Shell, BigQuery Console, Postman  

## 📄 Licencia

Este proyecto está bajo la Licencia MIT - ver el archivo [LICENSE](LICENSE) para detalles.

## 🤝 Soporte

Para preguntas o issues:
1. Revisar la documentación de GCP
2. Consultar los logs en Cloud Functions
3. Verificar quotas y límites en Cloud Console
4. Contactar el equipo de soporte académico

---

**¡Proyecto completado exitosamente! 🎉**

*Demostración práctica de arquitectura serverless, machine learning como servicio, y análisis de big data en Google Cloud Platform.*
