# Sistema de arbitraje por conmutador de red

Trabajo de graduación de Ingeniería Electrónica, Universidad del Valle de Guatemala (2026).
Autor: Alex James Ramírez Aldana.

Este repositorio contiene el código del sistema que permite a cuatro tableros didácticos con PLC Siemens S7-300 compartir la línea de producción didáctica del Laboratorio de Automatización Industrial. Un PLC Central S7-1500 actúa como árbitro: asigna el control de la línea a un solo tablero a la vez y conecta sus entradas y salidas con las máquinas.

## Arquitectura

| Nodo | Equipo | IP |
|---|---|---|
| PLC Central (árbitro) | S7-1500, CPU 1511C-1 PN | 192.168.10.1 |
| PLC didácticos C1–C4 | S7-300, CPU 315-2 PN/DP | 192.168.10.2 – 192.168.10.5 |
| Servidor Node-RED | PC del laboratorio | 192.168.10.20 |

- Red en estrella sobre un conmutador TP-Link TL-SG1016PE.
- Comunicación S7 con los bloques GET y PUT. En la comunicación, el PLC Central es el cliente y los PLC didácticos son los servidores.
- La cola de turnos atiende primero al tablero con menos turnos recibidos y desempata por orden de llegada.
- El tablero con el turno recibe los sensores de la línea en sus propias entradas (I8.5–I9.7), y sus salidas (Q12.0–Q13.2) se replican en el PLC Central.
- Una interfaz en Node-RED muestra el estado del sistema y permite asignar turnos de forma manual.

## Estructura

```
├── README.md
├── SistemaArbitraje_Cliente_Arbitro_Final.zip   Proyecto de TIA Portal V18 (PLC Central y PLC didácticos)
├── Dashboard_Arbitraje_28_10.json               Flujo de Node-RED (tablero de estado y panel manual)
└── dependencias_node-red.txt                    Paquetes y configuración de la conexión con el PLC Central
```

## Requisitos

- TIA Portal V18 o superior.
- Node.js y Node-RED, con los paquetes `@flowfuse/node-red-dashboard` 1.31.0 y `node-red-contrib-s7` 3.1.3.

## Documentación

El trabajo escrito, la guía de usuario e instalación, los diagramas eléctricos y de red y el mapa de E/S están en la carpeta compartida del trabajo de graduación (OneDrive UVG).

## Programas de los PLC (TIA Portal V18)

`SistemaArbitraje_Cliente_Arbitro_Final.zip` contiene el proyecto de TIA Portal `SistemaArbitraje_Cliente_Arbitro`, con el PLC Central y los cuatro PLC didácticos. Se abre con TIA Portal V18 o superior.

### PLC Central (S7-1500, CPU 1511C-1 PN, 192.168.10.1)

| Bloque | Función |
|---|---|
| OB1 | Lee los sensores, los envía al cliente con el turno, comunica con C1–C4 y aplica los comandos a las salidas |
| OB30 (100 ms) | Ejecuta la supervisión y el arbitraje |
| OB100 | Inicializa el turno, la cola y los estados internos al arrancar |
| FB_Comunicacion [FB1] | GET (8 B) y PUT (12 B) con cada PLC didáctico (instancias DB11, DB5, DB6, DB7) |
| FB_Supervision [FB2] | Señal de vida, detección de desconexión y registro de solicitudes y liberaciones |
| FB_Arbitraje [FB3] | Duración del turno, fin de turno, cola ordenada, asignación y habilitación de salidas |
| FC_Leer_Sensores [FC1] | Copia %I10.5–%I11.7 a DB_Linea.Sensores |
| FC_Aplicar_Comandos [FC2] | Escribe DB_Linea.Comandos en %Q4.0–%Q5.2 |
| FC_Distribuir_Sensores [FC4] | Envía los sensores solo al cliente con el turno |
| DB_Arbitraje [DB1], DB_Clientes [DB10], DB_Linea [DB2] | Estado del arbitraje, de cada cliente y de la línea |

### PLC didácticos C1–C4 (S7-300, CPU 315-2 PN/DP, 192.168.10.2–.5)

| Bloque | Función |
|---|---|
| FC_Espejo_Sensores [FC5] | Copia los sensores de la línea a I8.5–I9.7 cuando el PLC tiene el turno |
| FC_Gestion_Turno [FC1] | Pide (I8.2) o libera (I8.3) el turno con el selector del tablero |
| FC_Heartbeat [FC2] | Señal de vida de 1 Hz |
| FC3 «Programa del estudiante» | Espacio para el programa de la práctica |
| FC_Espejo_Comandos [FC6] | Envía Q12.0–Q13.2 al PLC Central solo cuando el PLC tiene el turno |
| DB_Arbitraje_Cliente [DB1] | Zona leída por el PLC Central (bytes 0–7) y zona escrita por él (bytes 8–19) |

El FC3 de C2 y C3 contiene los programas con los que se operó la Roscadora durante las pruebas. El de C3 es el programa de la Roscadora usado en la Innovacon 2026 de la UVG; no forma parte del desarrollo de este trabajo.

### Puesta en marcha

1. Abrir el proyecto en TIA Portal V18 o superior.
2. Verificar la IP de cada PLC y el ID_Cliente (1 a 4) en el DB1 de cada PLC didáctico.
3. En el PLC Central, habilitar el acceso PUT/GET y mantener DB1 y DB10 con acceso estándar, que Node-RED necesita para leer los datos.
4. Compilar, descargar en cada PLC y pasarlos a RUN.

El mapa completo de entradas y salidas y la conexión con la Roscadora se encuentran en la guía de usuario e instalación.

## Interfaz de visualización (Node-RED)

`Dashboard_Arbitraje_28_10.json` es el flujo exportado de Node-RED. Contiene dos páginas:

- **Tablero de estado** (`/dashboard/arbitraje`): PLC con el turno, tiempo restante, cola, conexión y turnos de cada PLC, motivo del último fin y total de asignaciones.
- **Panel de asignación manual** (`/dashboard/manual`): asignar turno, volver a automático, liberar turno y cambiar la duración del turno. Los botones solo funcionan después de marcar «Habilitar escritura», y cada acción pide confirmación.

### Funcionamiento

- Lee 37 variables de DB1 (DB_Arbitraje) y DB10 (DB_Clientes) del PLC Central cada 500 ms.
- Muestra el aviso «SIN DATOS» si pasan más de 1,5 s sin una lectura.
- Antes de escribir en el PLC, un filtro comprueba los rangos: Cliente_Forzado de 0 a 4 y Tiempo_Turno_Config de 60 a 5400 s.

### Instalación

1. Instalar Node.js y Node-RED: `npm install -g --unsafe-perm node-red`.
2. Iniciar Node-RED con `node-red` y abrir http://127.0.0.1:1880.
3. Instalar los paquetes indicados en `dependencias_node-red.txt` (Menú → Manage palette → Install).
4. Importar `Dashboard_Arbitraje_28_10.json` (Menú → Import) y presionar Deploy.
5. Configurar la PC con la IP 192.168.10.20/24.

### Limitaciones

- La interfaz no tiene usuario ni contraseña y usa HTTP sin cifrar.
- Solo se abre desde la PC del laboratorio, porque el puerto 1880 no está habilitado en el cortafuegos.
