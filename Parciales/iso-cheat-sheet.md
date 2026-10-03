# 🐧 GUÍA DE SUPERVIVENCIA: ISO (UNLP) - PRIMER PARCIAL
*Este Cheat Sheet interactivo compila de manera compacta y rigurosa todas las fórmulas, algoritmos, comandos y sintaxis de scripting obligatorios para las Prácticas 1 y 2 de la cátedra de Introducción a los Sistemas Operativos (UNLP).*

---

## 💻 PARTE 1: BASH SCRIPTING & COMANDOS (Práctica 1)

### 1. Sintaxis Esencial de Bash
Todo script de monitoreo o administración en el parcial debe utilizar esta base de control:

```bash
#!/bin/bash
# ----------------------------------------------------
# 1. VALIDACIÓN DE PARÁMETROS
# ----------------------------------------------------
# $# representa la cantidad de argumentos recibidos.
if [ $# -lt 1 ]; then
    echo "Error: Debe pasar al menos un parámetro."
    exit 1
fi

# ----------------------------------------------------
# 2. VARIABLES ESPECIALES (¡Teoría obligatoria!)
# ----------------------------------------------------
# $0 -> Nombre del script ejecutado (ej: ./script.sh)
# $1, $2... -> Parámetros posicionales
# $# -> Cantidad de parámetros recibidos
# $@ -> Lista completa de parámetros (útil para iterar: for p in "$@")
# $? -> Estado de retorno del último comando ejecutado (0 = Éxito)

# ----------------------------------------------------
# 3. COMPARACOINES NUMÉRICAS Y DE ARCHIVOS
# ----------------------------------------------------
# Numéricas: -eq (==) | -ne (!=) | -gt (>) | -lt (<) | -ge (>=) | -le (<=)
# De Archivo:
#   -e $1 -> Verdadero si el elemento existe en el sistema
#   -f $1 -> Verdadero si existe y es un archivo regular
#   -d $1 -> Verdadero si existe y es un directorio
#   -r $1 -> Verdadero si el usuario tiene permiso de lectura
#   -w $1 -> Verdadero si el usuario tiene permiso de escritura
#   -x $1 -> Verdadero si el usuario tiene permiso de ejecución
```

### 2. Estructura de Arreglos (Arrays)
```bash
# Declarar arreglo vacío
mi_arreglo=()

# Agregar elementos al final
mi_arreglo+=("nuevo_elemento")

# Obtener longitud (cantidad de elementos)
longitud=${#mi_arreglo[@]}

# Obtener todos los elementos
todos_los_elementos=${mi_arreglo[@]}

# Recorrer el arreglo con un bucle
for item in "${mi_arreglo[@]}"; do
    echo "Procesando: $item"
done
```

### 3. Plantilla de Monitoreo ("Demonios" cada X segundos)
Suele evaluarse como ejercicio de desarrollo en el examen escrito.
```bash
#!/bin/bash
# Monitorear indefinidamente un proceso/usuario/archivo cada 5 segundos
count=0
while true; do
    # Ejemplo: Chequear si el proceso apache se está ejecutando
    if ps -e | grep "apache" > /dev/null; then
        count=$((count + 1))
        echo "Detectado apache (Ocurrencia: $count)"
        
        # Salida controlada al llegar al límite
        if [ $count -eq 10 ]; then
            echo "Se detectó el proceso 10 veces. Finalizando..."
            exit 50
        fi
    fi
    sleep 5  # El temporizador (Timer)
done
```

### 4. Diccionario de Comandos Clave (FHS & Administración)
*   **Permisos Octales (`chmod`):** `chmod [propietario][grupo][otros] archivo`
    *   Lectura (`r`) = **4** | Escritura (`w`) = **2** | Ejecución (`x`) = **1**
    *   *Ejemplo:* `chmod 751 xxxx` $\rightarrow$ Propietario (4+2+1=7: rwx) | Grupo (4+1=5: r-x) | Otros (1: --x).
*   **Permisos sobre Directorios:**
    *   `r`: Permite listar su contenido (ver qué archivos hay adentro).
    *   `w`: Permite crear, renombrar o eliminar archivos dentro del directorio.
    *   `x`: Permite acceder al directorio (hacer `cd` hacia él).
*   **Filtros y Búsquedas (`find` / `grep`):**
    *   `find /etc -name "*.conf"`: Busca en tiempo real recorriendo físicamente los directorios.
    *   `grep -i "patron" archivo`: Busca coincidencias de texto dentro del archivo (ignora mayúsculas/minúsculas con `-i`).
*   **Empaquetado y Compresión (`tar`):**
    *   *Empaquetar:* `tar -cvf misLogs.tar /tmp/logs` (Agrupa sin reducir tamaño).
    *   *Empaquetar y comprimir:* `tar -czvf misLogs.tar.gz /tmp/logs` (Usa gzip para comprimir).
    *   *Descomprimir:* `tar -xzvf misLogs.tar.gz -C /destino` (Extrae en la ruta indicada).

---

## ⏱️ PARTE 2: PLANIFICACIÓN DE CPU (Práctica 2 - Procesos)

### 1. Fórmulas de Tiempos de Procesador
*   **Tiempo de Retorno ($TR$):** Tiempo total que el proceso pasa en el sistema (ejecución + espera + E/S).
    $$TR = \text{Tiempo de Finalización (Fin)} - \text{Tiempo de Arribo (Llegada)}$$
*   **Tiempo de Espera ($TE$):** Tiempo acumulado que el proceso pasa listo pero sin procesador.
    $$TE = TR - TCPU - T_{E/S}$$
    *(Si no hay operaciones de E/S intermedias, la fórmula se simplifica a: $TE = TR - TCPU$)*.
*   **Promedios del Lote:**
    $$TPR = \frac{\sum TR}{N} \quad\text{y}\quad TPE = \frac{\sum TE}{N}$$

### 2. Algoritmo Virtual Round Robin (VRR)
Es la solución de diseño para evitar el castigo a los procesos I/O-bound en Round Robin convencional:

```text
               ┌───────────────────────────────────────┐
               │          PROCESO EN EJECUCIÓN         │
               └───────────────────┬───────────────────┘
                                   │
                   ¿Qué causó la salida de la CPU?
                                   │
         ┌─────────────────────────┴─────────────────────────┐
    Fin de Quantum                                     Petición de E/S
         │                                                   │
         ▼                                                   ▼
┌──────────────────┐                               ┌───────────────────┐
│  Vuelve al final │                               │ Realiza la E/S y  │
│ de la Cola Ready │                               │ guarda Q_restante │
│  (Q_original)    │                               └─────────┬─────────┘
└──────────────────┘                                         │
                                                             ▼
                                                   ┌───────────────────┐
                                                   │ Ingresa a la      │
                                                   │   COLA AUXILIAR   │
                                                   └─────────┬─────────┘
                                                             │
                                                             ▼
                                                   ┌───────────────────┐
                                                   │ Ejecuta antes con │
                                                   │  su Q_restante    │
                                                   └───────────────────┘
```
*   **Criterio de Prioridad:** La Cola Auxiliar (*AuxQ*) tiene prioridad absoluta de despacho sobre la Cola Principal (*ReadyQ*).
*   **Quantum Fraccionado:** Si un proceso se bloquea tras consumir $t$ tiempo de un quantum original $Q$, al salir de E/S ejecutará con $Q_{restante} = Q - t$. Si agota este fragmento sin bloquearse nuevamente, vuelve a la Cola Principal con un quantum normal.

### 3. Estimación de Ráfagas (SJF y SRTF)
Se utiliza para predecir la duración de la próxima ráfaga de CPU ($S_{n+1}$) basándose en el comportamiento anterior:

*   **Fórmula de Media Aritmética Simple:**
    $$S_{n+1} = \frac{1}{n} T_n + \frac{n-1}{n} S_n$$
    *Donde $T_n$ es la ráfaga real número $n$, y $S_n$ es la estimación anterior. El valor inicial $S_1$ no influye en las estimaciones posteriores.*
*   **Fórmula de Suavizado Exponencial:**
    $$S_{n+1} = \alpha T_n + (1 - \alpha) S_n$$
    *   **$\alpha \to 1$ (cercano a 1):** Da un peso dominante a las ráfagas **más recientes** (ideal para procesos con cambios bruscos de comportamiento).
    *   **$\alpha \to 0$ (cercano a 0):** Da mayor peso al **historial acumulado** (el proceso tiene "memoria a largo plazo", las estimaciones son muy estables y lentas al cambio).

---

## 🧠 PARTE 3: ADMINISTRACIÓN DE MEMORIA (Práctica 2 - Memoria)

### 1. Traducción en Segmentación
*   **Formato de dirección lógica:** `segmento:desplazamiento` (ej. `0002:5678` $\rightarrow$ Segmento 2, Desplazamiento 5678).
*   **Algoritmo de Validación de Hardware:**
    1. Buscar la fila del `segmento` en la Tabla de Segmentos.
    2. Obtener el **Tamaño** límite y la **Dirección Base**.
    3. **Chequeo de Límite (MMU):** ¿El desplazamiento es estrictamente menor al tamaño del segmento?
       $$\text{¿Desplazamiento} < \text{Tamaño del Segmento?}$$
       *   **NO:** Se produce una interrupción por violación de acceso (**Segmentation Fault**). El direccionamiento aborta.
       *   **SÍ:** Se calcula la dirección física.
    4. **Cálculo de la Dirección Física ($DF$):**
       $$DF = \text{Dirección Base} + \text{Desplazamiento}$$

### 2. Traducción en Paginación
*   **Concepto clave:** La memoria física se divide en **Marcos (Frames)** y la memoria lógica del proceso en **Páginas**, ambos de idéntico tamaño (potencias de 2, ej: 2 KiB = 2048 bytes).
*   **Paso 1: Determinar el Número de Página ($P$) y el Desplazamiento ($D$):**
    $$P = \text{Parte Entera de } \left(\frac{\text{Dirección Lógica}}{\text{Tamaño de Página}}\right)$$
    $$D = \text{Resto de } \left(\frac{\text{Dirección Lógica}}{\text{Tamaño de Página}}\right)$$
*   **Paso 2: Buscar el Marco ($M$) en la Tabla de Páginas:**
    *   Se busca el índice $P$ en la tabla para obtener su correspondiente número de marco físico $M$.
*   **Paso 3: Calcular la Dirección Física ($DF$):**
    $$DF = (M \times \text{Tamaño de Página}) + D$$

### 3. Reemplazo de Páginas (Memoria Virtual)
*   **Page Fault (Fallo de Página):** Ocurre cuando el proceso hace referencia a una página que no está cargada en memoria física RAM.
*   **LRU (Least Recently Used):** Reemplaza la página que **no ha sido utilizada por el mayor intervalo de tiempo** (mira hacia el pasado para decidir).
*   **FIFO con Segunda Chance:**
    *   Utiliza un bit de referencia (bit $R$).
    *   Si la página en el frente de la cola tiene $R = 0$, es reemplazada inmediatamente.
    *   Si tiene $R = 1$, se le da una "segunda oportunidad": el bit $R$ se limpia a $0$, la página se mueve al final de la cola, y el puntero avanza a la siguiente.
*   **¡Detalle de Examen UNLP! Descarga Asincrónica:**
    Si el enunciado dice que se cuenta con $N$ marcos pero **se debe reservar 1 marco para la descarga asincrónica de páginas**, el algoritmo real de reemplazo se ejecuta únicamente sobre **$N - 1$ marcos activos**. El marco reservado nunca contiene páginas del proceso de manera permanente.

---

## 🗂️ PARTE 4: ENTRADA/SALIDA (I-NODOS)

### 1. Estructura de Asignación Indexada (I-Nodos UNLP)
Un I-Nodo típico de la UNLP contiene un array de punteros a bloques de disco (ej. 16 direcciones):
*   **Punteros Directos:** Apuntan directamente a bloques de datos (ej. primeros 10 punteros).
*   **Puntero Indirecto Simple:** Apunta a un bloque de disco que **contiene direcciones** a bloques de datos.
*   **Puntero Indirecto Doble:** Apunta a un bloque que contiene direcciones a bloques que a su vez contienen direcciones a bloques de datos.
*   **Puntero Indirecto Triple:** Añade un tercer nivel de indirección.

#### **Cálculo de Referencias por Bloque:**
$$\text{Direcciones por Bloque} = \frac{\text{Tamaño del Bloque (Bytes)}}{\text{Tamaño de Dirección/Puntero (Bytes)}}$$
*Ejemplo:* Si el bloque es de $2 \text{ KiB} = 2048 \text{ Bytes}$ y cada dirección es de $32 \text{ bits} = 4 \text{ Bytes}$:
$$\text{Direcciones por Bloque} = \frac{2048}{4} = \mathbf{512 \text{ direcciones}}$$

#### **Cálculo del Tamaño Máximo de un Archivo:**
$$\text{Máx Bloques} = \text{Directos} + (\text{Simples} \times N) + (\text{Dobles} \times N^2) + (\text{Triples} \times N^3)$$
$$\text{Tamaño Máximo} = \text{Máx Bloques} \times \text{Tamaño del Bloque}$$
*(Donde $N$ es la cantidad de direcciones por bloque calculado anteriormente).*

---

### 🏁 TABLA RESUMEN DE PROCESOS (Para comparar algoritmos)

| Algoritmo | Apropiativo (Preemptive) | Favorece a... | Desventaja / Riesgo |
| :--- | :---: | :--- | :--- |
| **FCFS** | No | Procesos CPU-bound | Efecto convoy, malos tiempos de respuesta interactivos. |
| **SJF** | No | Procesos cortos | Inanición (*starvation*) de procesos largos. |
| **SRTF** | Sí | Procesos I/O-bound (cortos) | Inanición de procesos largos (CPU-bound). |
| **Round Robin** | Sí (por fin de Quantum) | Equitativo (según Q) | Perjudica a procesos interactivos si no se usa VRR. |
| **Prioridades** | Ambos (según variante) | Procesos de alta prioridad | Inanición de procesos de baja prioridad (solución: *Aging*). |
| **VRR** | Sí | Procesos I/O-bound (interactivos) | Sobrecarga por gestión de cola auxiliar y quantums fraccionados. |
