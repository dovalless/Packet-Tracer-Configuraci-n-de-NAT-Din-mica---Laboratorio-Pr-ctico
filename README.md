# 🌐 Packet Tracer: Configuración de NAT Dinámica - Laboratorio Práctico

<div align="center">

**Laboratorio CISCO - Network Address Translation Dinámica**

[![Cisco Packet Tracer](https://img.shields.io/badge/Cisco-Packet_Tracer-1BA0D7?style=for-the-badge&logo=cisco&logoColor=white)](https://www.netacad.com)
[![NAT Protocol](https://img.shields.io/badge/Protocol-NAT-00A86B?style=for-the-badge)](https://www.cisco.com/)
[![CCNA](https://img.shields.io/badge/Certification-CCNA-blue?style=for-the-badge)](https://www.cisco.com/c/en/us/training-events/training-certifications/certifications/associate/ccna.html)
[![License](https://img.shields.io/badge/License-MIT-yellow?style=for-the-badge)](LICENSE)

[🎯 Objetivos](#-objetivos) • 
[⚙️ Configuración](#️-configuración-paso-a-paso) • 
[🔍 Verificación](#️-verificación) • 
[❓ Preguntas](#️-preguntas-del-laboratorio) • 
[👨‍💻 Autor](#️-autor)

</div>

---

## 📋 Descripción del Proyecto
Este laboratorio de Cisco Packet Tracer implementa **NAT Dinámica** básica, demostrando cómo una organización puede conectar múltiples dispositivos internos a Internet utilizando un conjunto limitado de direcciones IP públicas. A diferencia de PAT (sobrecarga), este método asigna una dirección pública única por cada conexión simultánea.

### 🎯 Objetivos
**Parte 1:** Configurar NAT Dinámica utilizando ACL y un pool de direcciones  
**Parte 2:** Verificar la implementación de NAT Dinámica y analizar sus limitaciones  

---

## 🛠️ Topología y Escenario

### 🔧 Dispositivos Involucrados
| Dispositivo | Red Interna | Función | Observación |
|-------------|-------------|---------|-------------|
| **R2** | Gateway | Router con NAT | Dispositivo de borde |
| **L1** | 172.16.0.0/16 | Dispositivo LAN | Primer dispositivo |
| **PC1** | 172.16.0.0/16 | Computadora | Segundo dispositivo |
| **PC2** | 172.16.0.0/16 | Computadora | Tercer dispositivo |
| **Server1** | Internet | Servidor Web | Destino externo |

### 🌐 Direccionamiento IP
| Elemento | Dirección/Máscara | Tipo | Notas |
|----------|-------------------|------|-------|
| **Red Interna** | 172.16.0.0/16 | Privada | 3 dispositivos a traducir |
| **Pool NAT** | 209.165.200.228/30 | Pública | Solo 2 direcciones usables |
| **Dirección Inicial Pool** | 209.165.200.229 | Pública | Primera IP usable |
| **Dirección Final Pool** | 209.165.200.230 | Pública | Segunda IP usable |

---

## ⚙️ Configuración Paso a Paso

### Parte 1: Configurar NAT Dinámica

#### Paso 1: Configurar ACL para Tráfico Permitido
```cisco
! Crear ACL 1 para permitir toda la red 172.16.0.0/16
R2(config)# access-list 1 permit 172.16.0.0 0.0.255.255
! Esta ACL identifica el tráfico que será traducido
```

#### Paso 2: Configurar Pool de Direcciones NAT
```cisco
! Configurar pool NAT con 2 direcciones públicas disponibles
R2(config)# ip nat pool NAT_POOL 209.165.200.229 209.165.200.230 netmask 255.255.255.252
! El pool incluye las dos direcciones usables del bloque /30
```

**Nota Importante:** La topología tiene 3 dispositivos internos (L1, PC1, PC2) pero el pool NAT solo tiene 2 direcciones públicas. Esto crea una situación de contención.

#### Paso 3: Asociar ACL con el Pool NAT
```cisco
! Vincular la ACL 1 con el pool NAT (sin overload)
R2(config)# ip nat inside source list 1 pool NAT_POOL
! Nota: NO se usa 'overload', por lo que es NAT dinámica 1:1
```

#### Paso 4: Configurar Interfaces NAT
```cisco
! Identificar interface externa (hacia Internet)
R2(config)# interface [INTERFAZ_EXTERNA]
R2(config-if)# ip nat outside

! Identificar interfaces internas (hacia red LAN)
R2(config)# interface [INTERFAZ_INTERNA_1]
R2(config-if)# ip nat inside

R2(config)# interface [INTERFAZ_INTERNA_2]
R2(config-if)# ip nat inside
```

### Parte 2: Verificar la Implementación de NAT

#### Paso 1: Acceder a Servicios Web
1. Abrir navegador web en L1, PC1 o PC2
2. Acceder a la página web de Server1
3. Verificar conectividad exitosa

#### Paso 2: Ver Traducciones NAT
```cisco
! Mostrar tabla de traducciones NAT activas
R2# show ip nat translations
```

**Salida Esperada:**
```
Pro Inside global      Inside local       Outside local      Outside global
--- 209.165.200.229    172.16.x.x         ---                ---
--- 209.165.200.230    172.16.y.y         ---                ---
```
*(Donde x.x e y.y son las direcciones de dos de los tres dispositivos internos)*

---

## ❓ Preguntas del Laboratorio

### Pregunta: ¿Qué sucederá si más de 2 dispositivos intentan acceder a Internet?
**Respuesta Detallada:**

Cuando más de 2 dispositivos intenten acceder a Internet simultáneamente, se producirán los siguientes problemas:

1. **Agotamiento de Direcciones:** El pool NAT solo tiene 2 direcciones públicas disponibles (209.165.200.229 y 209.165.200.230).

2. **Fallos de Conexión:** El tercer dispositivo que intente establecer una conexión recibirá un error o timeout porque:
   - No hay direcciones públicas disponibles en el pool
   - NAT dinámica sin overload asigna 1 IP pública por dispositivo
   - El router no puede crear más traducciones

3. **Comportamiento del Sistema:**
   - Los primeros 2 dispositivos funcionarán normalmente
   - El 3er dispositivo no obtendrá traducción NAT
   - Las conexiones existentes se mantendrán
   - El dispositivo 3 deberá esperar a que un dispositivo libere su IP pública

4. **Síntomas Observables:**
   ```
   Dispositivo 1: ✓ Conexión exitosa
   Dispositivo 2: ✓ Conexión exitosa  
   Dispositivo 3: ✗ Timeout/Error de conexión
   ```

### Análisis de la Limitación
**Causa Raíz:** NAT dinámica tradicional (sin PAT) tiene una relación 1:1 entre direcciones internas y públicas.

**Solución Recomendada:**
```cisco
! Modificar el comando NAT para habilitar sobrecarga (PAT)
R2(config)# no ip nat inside source list 1 pool NAT_POOL
R2(config)# ip nat inside source list 1 pool NAT_POOL overload
```
Con PAT, los 3 dispositivos podrían compartir las 2 IPs públicas mediante multiplexación por puertos.

---

## 📊 Análisis Técnico

### Comparación: NAT Dinámica vs NAT con Sobrecarga (PAT)

| Característica | NAT Dinámica (Este Lab) | NAT con Sobrecarga (PAT) |
|----------------|-------------------------|--------------------------|
| **Relación IPs** | 1:1 (una pública por interna) | 1:Muchos (múltiples por una pública) |
| **Pool Requerido** | Múltiples IPs | Mínimo 1 IP |
| **Eficiencia** | Baja (desperdicio de IPs) | Alta (máximo aprovechamiento) |
| **Escalabilidad** | Limitada por tamaño del pool | Limitada por número de puertos (~65k) |
| **Configuración** | `ip nat inside source list X pool Y` | `ip nat inside source list X pool Y overload` |

### 📈 Estadísticas Clave
- **Dispositivos Internos:** 3 (L1, PC1, PC2)
- **IPs Públicas Disponibles:** 2 (209.165.200.229-230)
- **Capacidad Máxima Simultánea:** 2 conexiones
- **Direcciones Desperdiciadas:** 1 dispositivo no puede conectarse
- **Ratio de Uso:** 66% (2 de 3 dispositivos)

### ⚠️ Limitaciones Identificadas
1. **Problema de Escalabilidad:** Cada nuevo dispositivo requiere una IP pública adicional
2. **Ineficiencia de Recursos:** IPs públicas ociosas cuando dispositivos no están activos
3. **Costo Operativo:** Necesidad de adquirir más bloques IP públicos
4. **Complejidad de Gestión:** Mantener inventario de IPs públicas asignadas

---

## 💡 Conceptos Fundamentales Aprendidos

### 🎯 NAT Dinámica Básica
- **Definición:** Traducción 1:1 temporal entre direcciones privadas y públicas
- **Mecanismo:** Tabla de mapeos dinámicos que expiran después de timeout
- **Ventaja:** Simple de configurar y entender
- **Desventaja:** Requiere muchas IPs públicas

### 🔧 Componentes de Configuración NAT
1. **ACL (Access Control List):** Identifica tráfico a traducir
   ```cisco
   access-list 1 permit 172.16.0.0 0.0.255.255
   ```

2. **Pool NAT:** Conjunto de direcciones públicas disponibles
   ```cisco
   ip nat pool NAT_POOL inicio fin netmask mascara
   ```

3. **Asociación ACL-Pool:** Vincula tráfico con direcciones
   ```cisco
   ip nat inside source list 1 pool NAT_POOL
   ```

4. **Interfaces:** Define zonas inside/outside
   ```cisco
   interface X
   ip nat inside/outside
   ```

### 📖 Comandos de Verificación Clave
```cisco
! Ver traducciones activas
R2# show ip nat translations

! Ver estadísticas NAT
R2# show ip nat statistics

! Ver configuración NAT activa
R2# show running-config | include nat

! Limpiar traducciones (útil para pruebas)
R2# clear ip nat translation *
```

---

## 🚀 Solución de Problemas NAT Dinámica

### Síntomas Comunes y Diagnóstico

#### ❌ No hay traducciones NAT
```cisco
! Verificar configuración básica
show running-config | section nat
show access-lists 1
show ip nat translations

! Verificar conectividad básica
ping 172.16.x.x    # Desde router a dispositivo interno
ping 209.165.200.228  # Verificar red externa
```

#### ❌ Solo algunos dispositivos funcionan
```cisco
! Verificar uso del pool
show ip nat translations
show ip nat statistics

! Mensaje típico en logs:
%NAT: translation failed, no available ports or addresses
```

#### ❌ Las traducciones no se crean
```cisco
! Verificar ACL
show ip access-lists

! Verificar interfaces NAT
show ip interface brief | include NAT
show running-config interface [interfaz]

! Probar con debug
debug ip nat
debug ip packet
```

### 🔍 Herramientas de Diagnóstico
| Comando | Propósito | Ejemplo de Salida Útil |
|---------|-----------|------------------------|
| `show ip nat translations` | Ver mapeos activos | Verificar IPs asignadas |
| `show ip nat statistics` | Ver uso de pool | "Total translations: 2" |
| `debug ip nat` | Ver traducciones en tiempo real | Ver paquetes siendo traducidos |
| `show access-lists` | Ver hits en ACL | "10 matches" indica tráfico |

### 📋 Checklist de Configuración NAT
- [ ] ACL configurada correctamente
- [ ] Pool NAT con direcciones válidas
- [ ] Comando de asociación ACL-Pool
- [ ] Interfaces marcadas como inside/outside
- [ ] Rutas configuradas hacia Internet
- [ ] No hay ACLs bloqueando tráfico

---

## 📚 Recursos Adicionales

### Documentación Oficial Cisco
- [Configuración de NAT Dinámica](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipaddr_nat/configuration/15-mt/nat-15-mt-book.html)
- [Comandos NAT de Cisco IOS](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipaddr_nat/command/nat-cr-book.html)
- [Guía de Estudio CCNA NAT](https://learningnetwork.cisco.com/s/article/ccna-nat-configuration-guide)

### Libros Recomendados
- "CCNA 200-301 Official Cert Guide, Volume 1" - Capítulo NAT
- "Network Address Translation" - K. Holdaway (especializado)
- "Cisco Router Configuration Handbook" - Sección NAT/PAT

### Laboratorios Relacionados
- **NAT Estático:** Traducción 1:1 permanente para servidores
- **PAT (NAT Overload):** Múltiples dispositivos con una IP
- **NAT64:** Traducción IPv6 a IPv4
- **Twice NAT:** Traducción en ambos sentidos

### 🎓 Preguntas de Práctica CCNA
1. ¿Cuál es el timeout predeterminado para traducciones NAT dinámicas?
2. ¿Cómo se ve afectado el rendimiento con NAT dinámica vs PAT?
3. ¿Qué comando muestra el número de hits en una ACL?
4. ¿Cómo se configura NAT para permitir acceso externo a un servidor interno?

---

## 👨‍💻 Autor

<div align="center">

**Darwin Manuel Ovalles Cesar**

<p align="center">
<a href="https://www.linkedin.com/in/darwin-manuel-ovalles-cesar-dev" target="_blank">
<img align="center" src="https://raw.githubusercontent.com/rahuldkjain/github-profile-readme-generator/master/src/images/icons/Social/linked-in-alt.svg" alt="LinkedIn - Darwin Ovalles" height="40" width="50" />
</a>
<a href="https://github.com/dovalless" target="_blank">
<img align="center" src="https://raw.githubusercontent.com/rahuldkjain/github-profile-readme-generator/master/src/images/icons/Social/github.svg" alt="GitHub - Darwin Ovalles" height="40" width="50" />
</a>
</p>

💼 **LinkedIn**: [darwin-manuel-ovalles-cesar-dev](https://www.linkedin.com/in/darwin-manuel-ovalles-cesar-dev/)  
🌐 **GitHub**: [@dovalless](https://github.com/dovalless)  
🎓 **Certificaciones**: CCNA, Network+, A+  

*"Este laboratorio demuestra una lección importante en redes: comprender las limitaciones de cada tecnología es tan crucial como saber configurarla. NAT dinámica nos enseña sobre la gestión eficiente de recursos IP escasos."*

**#Cisco #PacketTracer #NAT #CCNA #Networking #IPv4 #NetworkEngineering**

</div>

---

## 📄 Licencia

Este proyecto está bajo la Licencia MIT. Consulta el archivo [LICENSE](LICENSE) para más detalles.

```
MIT License
Copyright (c) 2024 Darwin Manuel Ovalles Cesar
```

---

## 🙏 Agradecimientos

- **Cisco Systems** - Por Packet Tracer y recursos educativos
- **Comunidad de Networking** - Por compartir conocimiento abiertamente
- **Estudiantes de Redes** - Por su curiosidad y preguntas desafiantes

<div align="center">

### ⭐ ¿Te resultó útil este laboratorio? Comparte con otros estudiantes ⭐

### 🔄 **Reflexión Final:** 
*"La NAT dinámica sin overload es como tener un estacionamiento con espacios numerados: solo puede entrar un auto por espacio. PAT es como un estacionamiento vertical: muchos autos en el mismo espacio, pero en diferentes niveles (puertos)."*

**Desarrollado con 💙 para futuros ingenieros de redes**

---
*Laboratorio completado: Packet Tracer - Configuración de NAT Dinámica*  
*Habilidades demostradas: NAT Dinámica, ACLs, Troubleshooting, Análisis de Limitaciones*

</div>
```
