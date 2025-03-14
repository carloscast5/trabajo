
# Manual del Programador: Guía Técnica de la Aplicación para Auditoría de Logs y Simulación de Ataques

Este manual está diseñado para desarrolladores interesados en comprender, mantener y ampliar la aplicación que consta de dos scripts principales: uno para el análisis de logs (`analisis_logs_admin.sh`) y otro para la simulación de ataques utilizando Hydra (`run_hydra.sh`). La aplicación se creó con el objetivo de auditar el archivo de logs del sistema y validar el funcionamiento de las medidas de seguridad mediante la simulación controlada de ataques.

---

## 1. Objetivos de la Aplicación

- **Automatización y Auditoría de Logs**:  
  Analizar automáticamente el archivo `/var/log/auth.log` para identificar patrones de intentos fallidos de autenticación y posibles ataques de fuerza bruta.

- **Detección y Alerta**:  
  Medir la cantidad de intentos fallidos (por IP y por día) y, en base a un umbral configurable por el usuario, emitir alertas automáticas mediante correo electrónico.  

- **Simulación de Ataques**:  
  Probar el sistema de auditoría utilizando el script `run_hydra.sh`, que simula ataques de fuerza bruta (usando Hydra) para generar entradas en el log del sistema y permitir la validación del análisis.

---

## 2. Tecnologías Usadas

- **Shell Scripting (Bash)**:  
  Permite la ejecución de comandos del sistema para analizar logs, gestionar reportes y automatizar tareas.

- **Zenity**:  
  Utilizado para proporcionar una interfaz gráfica interactiva que facilita la introducción de parámetros y la visualización de informes.

- **Hydra**:  
  Herramienta externa que simula ataques de fuerza bruta, generando eventos que son posteriormente procesados por la aplicación.

- **Utilidades Unix**:  
  Se emplean herramientas como `grep`, `awk`, `sort`, `uniq` para filtrar y procesar la información de los logs.  
- **xdotool**:  
  Utilizado para obtener el identificador de la ventana activa y asociar correctamente los diálogos gráficos, minimizando advertencias de GTK.

- **Mailutils**:  
  Se utiliza para enviar correos electrónicos de alerta a un destinatario configurado.

---

## 3. Estructura del Código y Descripción de Componentes

### 3.1. Script: analisis_logs_admin.sh

Este script es el núcleo de la auditoría de logs y consta de las siguientes secciones:

#### a) Configuración Inicial

Define variables clave como la ruta del archivo de log (`LOG_FILE`), el directorio donde se guardarán los reportes (`REPORT_DIR`), y la configuración para enviar alertas de correo:
```bash
LOG_FILE="/var/log/auth.log"
REPORT_DIR="/var/log/reportes_logs"
DESTINATARIO="destinatario@dominio.com"
ASUNTO="Alerta: Accesos fallidos detectados"
```
Se utiliza `xdotool` para obtener el identificador de la ventana activa y vincular los diálogos de Zenity:
```bash
PARENT_WINDOW=$(xdotool getactivewindow)
```
Si el directorio de reportes no existe, se crea.

#### b) Verificación del Archivo de Logs

La función `check_log_file` se asegura de que el archivo de log exista en el sistema:
```bash
function check_log_file() {
    if [ ! -f "$LOG_FILE" ]; then
        zenity --error --text="El archivo de logs no existe: $LOG_FILE" --attach="$PARENT_WINDOW"
        return 1
    fi
    return 0
}
```
Esta verificación ayuda a evitar errores en ejecución cuando el archivo requerido falta o tiene permisos restringidos.

#### c) Análisis y Reporte de Logs

La función `analizar_logs` realiza los siguientes pasos:
- **Solicitud de Parámetro**:  
  Pide al usuario un umbral de intentos fallidos a través de un diálogo Zenity.
- **Validación**:  
  Verifica que el valor ingresado no esté vacío y sea numérico.
- **Extracción de Datos**:  
  Procesa el log utilizando `grep` y `awk` para contar los intentos fallidos agrupados por dirección IP y por día.
- **Generación del Reporte**:  
  Crea un archivo de reporte con marca de tiempo incluido y lo muestra con un diálogo Zenity de tipo `text-info`.
- **Envío de Alertas**:  
  Si se identifica alguna IP con un número de intentos mayor o igual que el umbral, se envía una alerta mediante la función `enviar_alerta`.

Fragmento representativo del análisis:
```bash
Resultados=$(grep -a "Failed password" "$LOG_FILE" | awk '{print $(NF-3)}' | sort | uniq -c | sort -nr)
```

#### d) Funciones Adicionales

- **mostrar_estadisticas**:  
  Resume el total de fallos y las 10 IPs con mayor número de intentos fallidos.
- **ver_detalle_ip**:  
  Permite al usuario ingresar una IP y muestra en detalle las entradas correspondientes en el log.
- **listar_reportes y eliminar_reportes**:  
  Permiten al usuario ver todos los reportes generados y eliminar aquellos que ya no sean requeridos.
- **mostrar_intentos_por_dia**:  
  Muestra los intentos fallidos agrupados por día, facilitando el seguimiento por fecha.

#### e) Menú Interactivo

Utiliza Zenity para presentar un menú interactivo que permite seleccionar entre las diversas funcionalidades ofrecidas. Gracias a este menú, el usuario puede ejecutar múltiples tareas sin cerrar la aplicación.

### 3.2. Script: run_hydra.sh

Este script se encarga de simular un ataque de fuerza bruta. Sus partes clave son:

#### a) Configuración de Ataque

Define variables como:
```bash
TARGET="127.0.0.1"
USER="testuser"
PASSWORD_LIST="/path/to/passwords.txt"
```
Estas variables permiten configurar el objetivo del ataque, el usuario a atacar y la lista de contraseñas para probar.

#### b) Verificación de Instalación de Hydra

Antes de iniciar el ataque, el script verifica que Hydra se encuentre instalado:
```bash
if ! command -v hydra &> /dev/null; then
    echo "Hydra no está instalado. Instálalo y vuelve a intentar."
    exit 1
fi
```

#### c) Ejecución del Ataque

Se llama a Hydra con los parámetros configurados:
```bash
hydra -l "$USER" -P "$PASSWORD_LIST" ssh://"$TARGET"
```
Esto genera entradas en el log `/var/log/auth.log`, lo que permite que el script `analisis_logs_admin.sh` posteriormente identifique y analice estos intentos fallidos.

---

## 4. Usos de Archivos Log Asociados

- **/var/log/auth.log**:  
  Es el archivo que almacena los registros de autenticación del sistema. El script `analisis_logs_admin.sh` lo analiza para identificar intentos fallidos y posibles ataques de fuerza bruta.  
- **Directorios de Reportes**:  
  Los reportes generados son almacenados en el directorio configurado (`/var/log/reportes_logs`). Cada reporte contiene datos de los intentos agrupados por IP y por día, lo que resulta útil para análisis posteriores y revisiones de seguridad.

---

## 5. Propuestas de Mejora

La aplicación es código abierto y se puede mejorar o extender en diversos aspectos:

- **Interfaz Gráfica Persistente**:  
  Actualmente, se utilizan diálogos modales de Zenity. Se podría desarrollar una interfaz más robusta (por ejemplo, utilizando YAD o una aplicación web) que permita una experiencia de usuario más fluida y persistente.

- **Visualización de Datos**:  
  Incorporar gráficos (mediante `gnuplot` o bibliotecas en Python como `matplotlib`) para visualizar tendencias de ataques a lo largo del tiempo.

- **Compatibilidad con Múltiples Servicios**:  
  Extender la funcionalidad para auditar no solo SSH, sino otros servicios (FTP, HTTP, etc.) que puedan registrar intentos fallidos de autenticación.

- **Notificaciones en Tiempo Real**:  
  Implementar un sistema de notificaciones en tiempo real (mediante WebSockets u otro mecanismo) para alertar al administrador tan pronto se detecten patrones anómalos.

- **Optimización y Escalabilidad**:  
  Revisar el procesamiento de logs para manejar grandes volúmenes de datos, optimizando el uso de herramientas como `awk` o considerando su implementación en lenguajes como Python para mayor eficiencia.

- **Documentación y Modularidad**:  
  Mejorar la documentación interna del código y modularizar las funciones para facilitar el mantenimiento y la incorporación de nuevas funcionalidades.

---

## 6. Conclusión

Este manual proporciona una guía completa para comprender y extender la aplicación de auditoría de logs y simulación de ataques. Se han detallado los objetivos, las tecnologías utilizadas, la estructura del código y los usos específicos de archivos log. Además, se han propuesto mejoras futuras que permitirán adaptar la aplicación a nuevos requerimientos y aumentar su robustez. Al ser código abierto, se invita a cualquier desarrollador a contribuir y mejorar la aplicación, fomentando una comunidad colaborativa.

¡Gracias por utilizar y mejorar esta herramienta!
