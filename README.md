# Configuración de EdgeRouter para Balanceo de Carga (Load Balancing) y Conmutación por Error (Failover)

Esta guía describe cómo configurar un Ubiquiti EdgeRouter para balanceo de carga entre múltiples conexiones WAN y/o para conmutación por error (failover).

## Pasos de Configuración Base

Estos son los pasos fundamentales para configurar el balanceo de carga.

**1. Ingresar al modo de configuración:**

```bash
configure
```

**2. Crear un grupo de red para las direcciones IP privadas:**

Este grupo se utiliza para excluir el tráfico local del balanceo de carga.

```bash
set firewall group network-group PRIVATE_NETS network 192.168.0.0/16
set firewall group network-group PRIVATE_NETS network 172.16.0.0/12
set firewall group network-group PRIVATE_NETS network 10.0.0.0/8
```

**3. Crear una política de modificación de firewall (firewall modify policy):**

Esta política dirige el tráfico. Las primeras reglas excluyen el tráfico destinado a las redes locales y a las propias interfaces WAN del balanceo.

```bash
# Regla para el tráfico hacia redes privadas (no se balancea)
set firewall modify balance rule 10 action modify
set firewall modify balance rule 10 destination group network-group PRIVATE_NETS
set firewall modify balance rule 10 modify table main

# Regla para el tráfico destinado a la IP de la WAN1 (eth0) (no se balancea)
set firewall modify balance rule 20 action modify
set firewall modify balance rule 20 destination group address-group ADDRv4_eth0
set firewall modify balance rule 20 modify table main

# Regla para el tráfico destinado a la IP de la WAN2 (eth4) (no se balancea)
# Nota: Ajusta 'eth4' y 'ADDRv4_eth4' si tu segunda WAN usa otra interfaz.
set firewall modify balance rule 30 action modify
set firewall modify balance rule 30 destination group address-group ADDRv4_eth4
set firewall modify balance rule 30 modify table main
```

**4. Añadir una regla para enviar el resto del tráfico al grupo de balanceo de carga:**

Todo el tráfico que no coincide con las reglas anteriores se enviará al grupo de balanceo `G`.

```bash
set firewall modify balance rule 110 action modify
set firewall modify balance rule 110 modify lb-group G
```

**5. Aplicar la política de firewall a la interfaz LAN:**

Aplica la política `balance` al tráfico entrante (`in`) en tu interfaz LAN (por ejemplo, `switch0` o `eth1`, según tu configuración).

```bash
set interfaces switch switch0 firewall in modify balance
```
*Nota: Reemplaza `switch0` con tu interfaz LAN si es diferente (ej. `eth1`, `br0`).*

**6. Crear el grupo de balanceo de carga (Load-Balance Group):**

Este grupo (`G`) define qué interfaces WAN se utilizarán para el balanceo.

```bash
set load-balance group G interface eth0
set load-balance group G interface eth4
```
*Nota: Reemplaza `eth0` y `eth4` por tus interfaces WAN reales.*

**7. Opciones Adicionales para el Grupo de Balanceo:**

*   **`flush-on-active`**: Limpia todas las conexiones en la tabla de seguimiento (`conntrack`) cuando ocurre una transición de WAN (ej. una WAN se cae y otra se activa). Esto puede ser útil para forzar a las conexiones a reestablecerse por la nueva ruta.
    ```bash
    set load-balance group G flush-on-active enable
    ```

*   **`lb-local-metric-change`**: Cambia automáticamente la distancia de la ruta por defecto del router. Es especialmente útil en configuraciones de conmutación por error para asegurar que la ruta a través de la WAN de respaldo solo se use cuando la primaria falle.
    ```bash
    set load-balance group G lb-local-metric-change enable
    ```

**8. Confirmar y guardar la configuración:**

```bash
commit
save
```

---

## Modo de Conmutación por Error (Failover)

El modo de conmutación por error (Failover) permite designar una conexión WAN como primaria y otra(s) como secundaria(s) o de respaldo. La conexión secundaria solo se activará si la conexión primaria falla las pruebas de ruta (`route-test`).

**Configuración:**

Para configurar una interfaz WAN como "solo failover", se utiliza la opción `failover-only` dentro del grupo de balanceo de carga.

**Ejemplo de Configuración Failover:**

Supongamos que `eth0` es tu WAN principal y `eth4` es tu WAN de respaldo.

1.  **Definir `eth4` como `failover-only`:**
    ```bash
    set load-balance group G interface eth4 failover-only
    ```

2.  **Asegurar que `lb-local-metric-change` esté habilitado (recomendado para failover):**
    Esto ajusta automáticamente la métrica de la ruta para priorizar la WAN activa. Si no se configuró en los pasos base:
    ```bash
    set load-balance group G lb-local-metric-change enable
    ```

3.  **Configurar Pruebas de Ruta (Route Tests) Robustas:**
    Es crucial tener pruebas de ruta fiables para detectar cuándo la WAN primaria ha fallado. Ver la sección "Configuración Avanzada de Pruebas de Ruta (Route Tests)".

**Funcionamiento:**

*   En condiciones normales, todo el tráfico saldrá por `eth0`.
*   Si `eth0` falla las pruebas de ruta configuradas, el EdgeRouter automáticamente redirigirá el tráfico a través de `eth4`.
*   Cuando `eth0` vuelva a estar operativa y pase las pruebas de ruta, el tráfico volverá a utilizar `eth0` como la ruta principal (asumiendo que `lb-local-metric-change` está habilitado y las métricas de ruta se gestionan correctamente).

---

## Configuración Avanzada de Pruebas de Ruta (Route Tests)

Las pruebas de ruta (`route-test`) son esenciales para que el EdgeRouter detecte cuándo una conexión WAN no está operativa y necesita realizar una conmutación por error o ajustar el balanceo de carga.

**Importancia de Hosts de Destino Distintos:**

Es altamente recomendable configurar **diferentes hosts de destino (`target`) para cada interfaz WAN**. Si ambas WANs prueban contra el mismo host (ej. `1.1.1.1`), y ese host específico tiene problemas, ambas WANs podrían marcarse como caídas incorrectamente. Usar hosts distintos (ej. DNS públicos de diferentes proveedores) proporciona una verificación más robusta del estado real de cada conexión.

**Parámetros Comunes de `route-test`:**

*   `type ping target <host>`: Define el tipo de prueba (ping) y el destino.
*   `count success <N>`: Número de pings exitosos consecutivos para considerar la ruta como activa.
*   `count failure <N>`: Número de pings fallidos consecutivos para considerar la ruta como inactiva.
*   `initial-delay <segundos>`: Tiempo a esperar antes de iniciar la primera prueba después de un arranque o cambio de estado.
*   `interval <segundos>`: Frecuencia con la que se realiza la prueba.

**Ejemplo de Configuración de `route-test` con Hosts Distintos:**

```bash
# Configuración para WAN1 (eth0)
set load-balance group G interface eth0 route-test count success 5
set load-balance group G interface eth0 route-test count failure 3
set load-balance group G interface eth0 route-test initial-delay 60
set load-balance group G interface eth0 route-test interval 10
set load-balance group G interface eth0 route-test type ping target 1.1.1.1 # Cloudflare DNS

# Configuración para WAN2 (eth4)
set load-balance group G interface eth4 route-test count success 5
set load-balance group G interface eth4 route-test count failure 3
set load-balance group G interface eth4 route-test initial-delay 60
set load-balance group G interface eth4 route-test interval 10
set load-balance group G interface eth4 route-test type ping target 9.9.9.9 # Quad9 DNS
```

**Recomendaciones Adicionales:**

*   **Elije hosts fiables:** Utiliza direcciones IP de servidores DNS públicos conocidos (Cloudflare, Google, Quad9, etc.) u otros hosts que sepas que tienen alta disponibilidad.
*   **Ajusta los contadores y tiempos:** Los valores de `count success`, `count failure`, `initial-delay`, e `interval` pueden necesitar ajustes según la sensibilidad deseada y la estabilidad de tus conexiones. Valores más bajos harán la detección más rápida pero podrían generar falsos positivos en conexiones inestables.
*   **Considera `type default`:** Para algunas conexiones (como PPPoE o DHCP) que tienen su propia detección de estado de enlace, `type default` puede ser suficiente, ya que utiliza el estado del gateway proporcionado por el ISP. Sin embargo, un `type ping` a un host externo verifica la conectividad real a Internet.

---

## Modo "Sticky" (Persistencia de Conexión) en Balanceo de Carga

Cuando se utiliza el balanceo de carga, el EdgeRouter distribuye las nuevas conexiones entre las WANs activas. Sin embargo, para ciertas aplicaciones (especialmente HTTPS, banca online, sesiones de juego, etc.), es importante que todas las solicitudes de una misma sesión se dirijan a través de la misma WAN. Esto se conoce como persistencia de conexión o "stickiness".

Si las conexiones de una misma sesión saltan entre diferentes IPs públicas (diferentes WANs), algunos servicios pueden cerrar la sesión o comportarse de manera inesperada por motivos de seguridad.

**Configuración del Modo Sticky:**

El EdgeRouter permite habilitar la persistencia de conexión para el grupo de balanceo de carga.

```bash
# Habilitar persistencia basada en la dirección IP de origen
set load-balance group G sticky source-addr enable

# Habilitar persistencia basada en la dirección IP de destino
set load-balance group G sticky dest-addr enable

# Habilitar persistencia basada en el puerto de destino
set load-balance group G sticky dest-port enable
```

**Opciones Comunes para `sticky`:**

*   `source-addr enable`: Hace que las conexiones desde una misma IP de origen interna tiendan a usar la misma WAN. Esta es la opción más comúnmente útil.
*   `dest-addr enable`: Hace que las conexiones hacia una misma IP de destino externa tiendan a usar la misma WAN.
*   `dest-port enable`: Hace que las conexiones hacia un mismo puerto de destino tiendan a usar la misma WAN.

**Recomendaciones:**

*   Generalmente, habilitar `sticky source-addr enable` es un buen punto de partida para la mayoría de los escenarios de balanceo de carga, ya que ayuda a mantener la consistencia de la sesión para los clientes de la LAN.
*   Habilitar múltiples opciones de `sticky` puede hacer que el balanceo sea menos efectivo, ya que impone más restricciones. Prueba y ajusta según tus necesidades.
*   Recuerda que el modo "sticky" es una preferencia. Si una WAN se cae, las conexiones "pegadas" a ella se moverán a otra WAN activa.

---

## Pruebas de Velocidad (Opcional)

Puedes realizar pruebas de velocidad usando `iperf3` desde la CLI del EdgeRouter para verificar el rendimiento de tus conexiones WAN.

```bash
iperf3 -c bouygues.iperf.fr -R -P 10
```

Puedes encontrar una lista de servidores iPerf públicos aquí:
[https://iperf.fr/iperf-servers.php](https://iperf.fr/iperf-servers.php)
