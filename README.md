### Escuela Colombiana de Ingeniería
### Arquitecturas de Software - ARSW

## Escalamiento en Azure con Maquinas Virtuales, Sacale Sets y Service Plans

### Dependencias
* Cree una cuenta gratuita dentro de Azure. Para hacerlo puede guiarse de esta [documentación](https://azure.microsoft.com/es-es/free/students/). Al hacerlo usted contará con $100 USD para gastar durante 12 meses.

### Parte 0 - Entendiendo el escenario de calidad

Adjunto a este laboratorio usted podrá encontrar una aplicación totalmente desarrollada que tiene como objetivo calcular el enésimo valor de la secuencia de Fibonnaci.

**Escalabilidad**
Cuando un conjunto de usuarios consulta un enésimo número (superior a 1000000) de la secuencia de Fibonacci de forma concurrente y el sistema se encuentra bajo condiciones normales de operación, todas las peticiones deben ser respondidas y el consumo de CPU del sistema no puede superar el 70%.

### Parte 1 - Escalabilidad vertical

1. Diríjase a el [Portal de Azure](https://portal.azure.com/) y a continuación cree una maquina virtual con las características básicas descritas en la imágen 1 y que corresponden a las siguientes:
    * Resource Group = SCALABILITY_LAB
    * Virtual machine name = VERTICAL-SCALABILITY
    * Image = Ubuntu Server 
    * Size = Standard B1ls
    * Username = scalability_lab
    * SSH publi key = Su llave ssh publica

![Imágen 1](images/part1/part1-vm-basic-config.png)

2. Para conectarse a la VM use el siguiente comando, donde las `x` las debe remplazar por la IP de su propia VM (Revise la sección "Connect" de la virtual machine creada para tener una guía más detallada).

    `ssh scalability_lab@xxx.xxx.xxx.xxx`

3. Instale node, para ello siga la sección *Installing Node.js and npm using NVM* que encontrará en este [enlace](https://linuxize.com/post/how-to-install-node-js-on-ubuntu-18.04/).
4. Para instalar la aplicación adjunta al Laboratorio, suba la carpeta `FibonacciApp` a un repositorio al cual tenga acceso y ejecute estos comandos dentro de la VM:

    `git clone <your_repo>`

    `cd <your_repo>/FibonacciApp`

    `npm install`

5. Para ejecutar la aplicación puede usar el comando `npm FibinacciApp.js`, sin embargo una vez pierda la conexión ssh la aplicación dejará de funcionar. Para evitar ese compartamiento usaremos *forever*. Ejecute los siguientes comando dentro de la VM.

    ` node FibonacciApp.js`

6. Antes de verificar si el endpoint funciona, en Azure vaya a la sección de *Networking* y cree una *Inbound port rule* tal como se muestra en la imágen. Para verificar que la aplicación funciona, use un browser y user el endpoint `http://xxx.xxx.xxx.xxx:3000/fibonacci/6`. La respuesta debe ser `The answer is 8`.

![](images/part1/part1-vm-3000InboudRule.png)

7. La función que calcula en enésimo número de la secuencia de Fibonacci está muy mal construido y consume bastante CPU para obtener la respuesta. Usando la consola del Browser documente los tiempos de respuesta para dicho endpoint usando los siguintes valores:
    * 1000000
    * 1010000
    * 1020000
    * 1030000
    * 1040000
    * 1050000
    * 1060000
    * 1070000
    * 1080000
    * 1090000    

8. Dírijase ahora a Azure y verifique el consumo de CPU para la VM. (Los resultados pueden tardar 5 minutos en aparecer).

![Imágen 2](images/part1/part1-vm-cpu.png)

9. Ahora usaremos Postman para simular una carga concurrente a nuestro sistema. Siga estos pasos.
    * Instale newman con el comando `npm install newman -g`. Para conocer más de Newman consulte el siguiente [enlace](https://learning.getpostman.com/docs/postman/collection-runs/command-line-integration-with-newman/).
    * Diríjase hasta la ruta `FibonacciApp/postman` en una maquina diferente a la VM.
    * Para el archivo `[ARSW_LOAD-BALANCING_AZURE].postman_environment.json` cambie el valor del parámetro `VM1` para que coincida con la IP de su VM.
    * Ejecute el siguiente comando.

    ```
    newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10 &
    newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10
    ```

10. La cantidad de CPU consumida es bastante grande y un conjunto considerable de peticiones concurrentes pueden hacer fallar nuestro servicio. Para solucionarlo usaremos una estrategia de Escalamiento Vertical. En Azure diríjase a la sección *size* y a continuación seleccione el tamaño `B2ms`.

![Imágen 3](images/part1/part1-vm-resize.png)

11. Una vez el cambio se vea reflejado, repita el paso 7, 8 y 9.
12. Evalue el escenario de calidad asociado al requerimiento no funcional de escalabilidad y concluya si usando este modelo de escalabilidad logramos cumplirlo.
13. Vuelva a dejar la VM en el tamaño inicial para evitar cobros adicionales.

**Preguntas**

1. ¿Cuántos y cuáles recursos crea Azure junto con la VM?
2. ¿Brevemente describa para qué sirve cada recurso?
3. ¿Al cerrar la conexión ssh con la VM, por qué se cae la aplicación que ejecutamos con el comando `npm FibonacciApp.js`? ¿Por qué debemos crear un *Inbound port rule* antes de acceder al servicio?
4. Adjunte tabla de tiempos e interprete por qué la función tarda tando tiempo.
5. Adjunte imágen del consumo de CPU de la VM e interprete por qué la función consume esa cantidad de CPU.
6. Adjunte la imagen del resumen de la ejecución de Postman. Interprete:
    * Tiempos de ejecución de cada petición.
    * Si hubo fallos documentelos y explique.
7. ¿Cuál es la diferencia entre los tamaños `B2ms` y `B1ls` (no solo busque especificaciones de infraestructura)?
8. ¿Aumentar el tamaño de la VM es una buena solución en este escenario?, ¿Qué pasa con la FibonacciApp cuando cambiamos el tamaño de la VM?
9. ¿Qué pasa con la infraestructura cuando cambia el tamaño de la VM? ¿Qué efectos negativos implica?
10. ¿Hubo mejora en el consumo de CPU o en los tiempos de respuesta? Si/No ¿Por qué?
11. Aumente la cantidad de ejecuciones paralelas del comando de postman a `4`. ¿El comportamiento del sistema es porcentualmente mejor?

### Parte 2 - Escalabilidad horizontal

#### Crear el Balanceador de Carga

Antes de continuar puede eliminar el grupo de recursos anterior para evitar gastos adicionales y realizar la actividad en un grupo de recursos totalmente limpio.

1. El Balanceador de Carga es un recurso fundamental para habilitar la escalabilidad horizontal de nuestro sistema, por eso en este paso cree un balanceador de carga dentro de Azure tal cual como se muestra en la imágen adjunta.

![](images/part2/part2-lb-create.png)

2. A continuación cree un *Backend Pool*, guiese con la siguiente imágen.

![](images/part2/part2-lb-bp-create.png)

3. A continuación cree un *Health Probe*, guiese con la siguiente imágen.

![](images/part2/part2-lb-hp-create.png)

4. A continuación cree un *Load Balancing Rule*, guiese con la siguiente imágen.

![](images/part2/part2-lb-lbr-create.png)

5. Cree una *Virtual Network* dentro del grupo de recursos, guiese con la siguiente imágen.

![](images/part2/part2-vn-create.png)

#### Crear las maquinas virtuales (Nodos)

Ahora vamos a crear 3 VMs (VM1, VM2 y VM3) con direcciones IP públicas standar en 3 diferentes zonas de disponibilidad. Después las agregaremos al balanceador de carga.

1. En la configuración básica de la VM guíese por la siguiente imágen. Es importante que se fije en la "Avaiability Zone", donde la VM1 será 1, la VM2 será 2 y la VM3 será 3.

![](images/part2/part2-vm-create1.png)

2. En la configuración de networking, verifique que se ha seleccionado la *Virtual Network*  y la *Subnet* creadas anteriormente. Adicionalmente asigne una IP pública y no olvide habilitar la redundancia de zona.

![](images/part2/part2-vm-create2.png)

3. Para el Network Security Group seleccione "avanzado" y realice la siguiente configuración. No olvide crear un *Inbound Rule*, en el cual habilite el tráfico por el puerto 3000. Cuando cree la VM2 y la VM3, no necesita volver a crear el *Network Security Group*, sino que puede seleccionar el anteriormente creado.

![](images/part2/part2-vm-create3.png)

4. Ahora asignaremos esta VM a nuestro balanceador de carga, para ello siga la configuración de la siguiente imágen.

![](images/part2/part2-vm-create4.png)

5. Finalmente debemos instalar la aplicación de Fibonacci en la VM. para ello puede ejecutar el conjunto de los siguientes comandos, cambiando el nombre de la VM por el correcto

```
git clone https://github.com/daprieto1/ARSW_LOAD-BALANCING_AZURE.git

curl -o- https://raw.githubusercontent.com/creationix/nvm/v0.34.0/install.sh | bash
source /home/vm1/.bashrc
nvm install node

cd ARSW_LOAD-BALANCING_AZURE/FibonacciApp
npm install

npm install forever -g
forever start FibonacciApp.js
```

Realice este proceso para las 3 VMs, por ahora lo haremos a mano una por una, sin embargo es importante que usted sepa que existen herramientas para aumatizar este proceso, entre ellas encontramos Azure Resource Manager, OsDisk Images, Terraform con Vagrant y Paker, Puppet, Ansible entre otras.

#### Probar el resultado final de nuestra infraestructura

1. Porsupuesto el endpoint de acceso a nuestro sistema será la IP pública del balanceador de carga, primero verifiquemos que los servicios básicos están funcionando, consuma los siguientes recursos:

```
http://52.155.223.248/
http://52.155.223.248/fibonacci/1
```

2. Realice las pruebas de carga con `newman` que se realizaron en la parte 1 y haga un informe comparativo donde contraste: tiempos de respuesta, cantidad de peticiones respondidas con éxito, costos de las 2 infraestrucruras, es decir, la que desarrollamos con balanceo de carga horizontal y la que se hizo con una maquina virtual escalada.

3. Agregue una 4 maquina virtual y realice las pruebas de newman, pero esta vez no lance 2 peticiones en paralelo, sino que incrementelo a 4. Haga un informe donde presente el comportamiento de la CPU de las 4 VM y explique porque la tasa de éxito de las peticiones aumento con este estilo de escalabilidad.

```
newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10 &
newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10 &
newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10 &
newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10
```

**Preguntas**

* ¿Cuáles son los tipos de balanceadores de carga en Azure y en qué se diferencian?, ¿Qué es SKU, qué tipos hay y en qué se diferencian?, ¿Por qué el balanceador de carga necesita una IP pública?
* ¿Cuál es el propósito del *Backend Pool*?
* ¿Cuál es el propósito del *Health Probe*?
* ¿Cuál es el propósito de la *Load Balancing Rule*? ¿Qué tipos de sesión persistente existen, por qué esto es importante y cómo puede afectar la escalabilidad del sistema?.
* ¿Qué es una *Virtual Network*? ¿Qué es una *Subnet*? ¿Para qué sirven los *address space* y *address range*?
* ¿Qué son las *Availability Zone* y por qué seleccionamos 3 diferentes zonas?. ¿Qué significa que una IP sea *zone-redundant*?
* ¿Cuál es el propósito del *Network Security Group*?
* Informe de newman 1 (Punto 2)
* Presente el Diagrama de Despliegue de la solución.

---

## Desarrollo – Parte 2: Escalabilidad horizontal

> El paso a paso con todas las capturas está en [`EVIDENCIAS.md`](EVIDENCIAS.md). 

### Ajustes 

La suscripción de estudiante tiene varios límites, así que al final quedó un poco distinto:

| Guía | Lo que hicimos | Por qué |
|---|---|---|
| Región North Europe | **Central US** | La suscripción tiene una política (*Allowed resource deployment regions*) que solo permite `southcentralus`, `canadacentral`, `northcentralus`, `centralus` y `mexicocentral`. East US 2 falló por *policy violation*. |
| VMs tamaño B1ls | **Standard_D2s_v3** (VM1, VM3) y **Standard_D2s_v4** (VM2), 2 vCPU / 8 GB | Toda la serie B (B1ls, B1s, B2ats_v2) sale como *"Tamaño no disponible"* para la suscripción en Central US. La familia DSv3 tiene tope de 4 vCPU, por eso VM2 quedó en D2s_v4 (mismas specs). |
| 4 VMs | **3 VMs** | Tope de **6 vCPU regionales**: 3 VMs × 2 vCPU = 6. No hay tamaños de 1 vCPU disponibles para la suscripción. |
| IP pública en cada VM | IP pública en **VM1, VM2 y el LB**; VM3 solo IP privada | Tope de **3 IPs públicas por región** (`PublicIPCountLimitReached`). A VM3 se le quitó la IP después de instalar la app; el LB le llega por la IP privada, así que no afecta. |
| Instalación manual por SSH con nvm + `forever` | **cloud-init** ([`cloud-init-fibonacci.yaml`](cloud-init-fibonacci.yaml)) + servicio **systemd** | Cada VM se instala sola al arrancar. systemd, además de mantener la app viva sin sesión SSH, la vuelve a levantar si la VM se reinicia. |
| `&` para paralelizar newman | Varias ventanas de PowerShell (`Start-Process`) | PowerShell no acepta `&` como operador de segundo plano. |

### Infraestructura desplegada

| Recurso | Nombre | Configuración |
|---|---|---|
| Grupo de recursos | `SCALABILITY_LAB` | Central US |
| Virtual Network | `SCALABILITY_LAB_VIRTUAL_NETWORK` | Address space `10.1.0.0/16` |
| Subnet | `SCALABILITY_LAB_SUBNET` | Address range `10.1.0.0/24` |
| Load Balancer | `SCALABILITY_LAB_LOAD_BALANCER` | SKU Estándar, Público, Regional |
| Front-end | `LoadBalancerFrontEnd` | IP pública estática **zone-redundant** `48.214.164.243` |
| Backend pool | `SCALABILITY_LAB_BACKEND_POOL` | VM1, VM2, VM3 (por NIC) |
| Health probe | `SCALABILITY_LAB_HEALTH_PROBE` | TCP 3000, cada 5 s |
| Regla de balanceo | `SCALABILITY_LAB_LOAD_BALANCING_RULE` | TCP 80 → 3000, sin persistencia de sesión |
| NSG | `SCALABILITY_LAB_NSG` | Entrada: 22 (SSH, prioridad 1000) y 3000 (app, prioridad 310) |
| VM1 | Zona 1 | D2s_v3 · IP privada 10.1.0.4 |
| VM2 | Zona 2 | D2s_v4 · IP privada 10.1.0.6 |
| VM3 | Zona 3 | D2s_v3 · IP privada 10.1.0.5 |

### Probar el resultado final

**Punto 1 – Servicios básicos a través del LB** ([evidencia 13](EVIDENCIAS.md#13-prueba-de-la-app-a-través-del-load-balancer)):
- `http://48.214.164.243/` → `Hello World`
- `http://48.214.164.243/fibonacci/6` → `The answer is 8`

Además se probó cada nodo directo por su IP pública en el puerto 3000 .

![](images/part2/evidencias/13-prueba-lb-navegador.png)

**Punto 2 – newman, 2 ejecuciones en paralelo** (`-n 10`, `nth = 1000000`):

![](images/part2/evidencias/17-newman-2-paralelo.png)

| Ejecución | Peticiones | Fallidas | Promedio | Mín – Máx | Desv. est. |
|---|---|---|---|---|---|
| 1 | 10 | 0 | 19,6 s | 19,3 – 20,4 s | 277 ms |
| 2 | 10 | 0 | 24,1 s | 23,7 – 25,1 s | 424 ms |
| **Total** | **20** | **0** | | | **100 % de éxito** |

**Punto 3 – newman, 4 ejecuciones en paralelo con 3 nodos:**

![](images/part2/evidencias/19-newman-4-paralelo.png)

| Ejecución | Peticiones | Fallidas | Promedio | Mín – Máx | Desv. est. |
|---|---|---|---|---|---|
| 1 | 10 | 1 | 19,2 s | 19,2 – 19,5 s | 99 ms |
| 2 | 10 | 3 | 31,5 s | 21,1 – 56,6 s | 13,2 s |
| 3 | 10 | 1 | 27,8 s | 23,1 – 46,1 s | 9,1 s |
| 4 | 10 | 1 | 21,2 s | 21,1 – 21,6 s | 142 ms |
| **Total** | **40** | **6** (`ECONNRESET`) | | | **85 % de éxito** |

**Comportamiento de la CPU** (Percentage CPU, última hora):

![](images/part2/evidencias/20-cpu-3-vms-ultima-hora.png)

| VM | Prueba de 2 en paralelo | Prueba de 4 en paralelo |
|---|---|---|
| VM1 | ~0 % (no recibió carga) | ~50 % |
| VM2 | ~50 % | ~50 % |
| VM3 | ~50 % | ~50 % |

**Análisis del punto 3:**
- La 4.ª VM no se pudo crear por el tope de 6 vCPU, así que las 4 ejecuciones se repartieron entre 3 nodos y **a un nodo le tocaron dos flujos a la vez**.
- Esos dos flujos son los que muestran tiempos inestables (picos de 46 y 56 s, desviación de 9-13 s): FibonacciApp corre en **Node.js, de un solo hilo**, y mientras calcula una petición (~20 s) la otra espera en fila.
- Con el event loop bloqueado aparecen **6 `read ECONNRESET`** (el servidor cierra la conexión). Nuestra hipótesis es que los timers de keep-alive de Node se disparan tarde por el bloqueo y el servidor cierra sockets justo cuando newman manda la siguiente petición; al reconectar, el LB vuelve a hacer el hash y por eso también aparece algún fallo en los flujos que iban estables.
- **Ninguna VM pasa de ~50 % de CPU**, ni la que atendió dos flujos: con 2 vCPU, Node solo puede usar un núcleo. La saturación no se ve como más CPU, sino como más espera.
- **¿Por qué aumenta la tasa de éxito con este estilo de escalabilidad?** Porque cada nodo nuevo agrega un núcleo más que realmente atiende peticiones en paralelo. Con 2 flujos y 3 nodos, cada flujo tuvo su propio nodo: **100 % de éxito** y tiempos estables. Con 4 flujos y 3 nodos bajó a **85 %**. Con la 4.ª VM que pide la guía, cada flujo tendría su nodo y el resultado debería volver a parecerse al de la prueba de 2 (estable y sin fallos). Agrandar una sola VM no logra esto, porque Node no aprovecha los núcleos extra.

### Preguntas

**1. ¿Cuáles son los tipos de balanceadores de carga en Azure y en qué se diferencian? ¿Qué es SKU, qué tipos hay y en qué se diferencian? ¿Por qué el balanceador de carga necesita una IP pública?**

Azure tiene cuatro servicios de balanceo:

| Servicio | Capa | Alcance | Para qué sirve |
|---|---|---|---|
| **Azure Load Balancer** | L4 (TCP/UDP) | Regional (o global con el nivel *Global*) | Reparte conexiones TCP/UDP entre VMs. Es el que usamos. |
| **Application Gateway** | L7 (HTTP/HTTPS) | Regional | Enruta por URL o host, termina TLS, trae WAF y afinidad por cookie. |
| **Azure Front Door** | L7 (HTTP/HTTPS) | Global | Puerta de entrada global con caché, failover rápido entre regiones y WAF. |
| **Traffic Manager** | DNS | Global | No toca el tráfico: responde el DNS con el endpoint que convenga (por prioridad, rendimiento o geografía). |

El **SKU** es la "versión o plan" de un recurso, y define sus capacidades, límites, SLA y precio. Para el Load Balancer hay:
- **Básico:** sin zonas de disponibilidad, sin SLA y abierto por defecto. **Microsoft lo retiró el 30 de septiembre de 2025**, por eso ya no aparece como opción.
- **Estándar:** soporta zonas (frontends zone-redundant), tiene SLA, más instancias por pool, métricas en Azure Monitor y es **seguro por defecto** (no deja pasar nada que no permita un NSG). Es el que usamos.
- **Gateway:** para encadenar appliances virtuales de red de terceros (firewalls, inspección de paquetes) en medio del tráfico. No reparte carga entre VMs de aplicación.

Además el Estándar tiene dos **niveles**: *Regional* (una región, el nuestro) y *Global* (reparte entre balanceadores de varias regiones).

**¿Por qué IP pública?** Porque es un balanceador **público**: es la única puerta de entrada al sistema desde internet. Los clientes (newman en nuestro PC) solo conocen la IP del LB, y el LB habla con las VMs por sus IPs privadas. Si fuera un LB **interno**, tendría solo IP privada y únicamente sería accesible desde dentro de la VNet.

**2. ¿Cuál es el propósito del *Backend Pool*?**

Es el **grupo de instancias que van a atender el tráfico**: la lista de NICs (o IPs) a las que el LB les puede mandar peticiones. En nuestro caso tiene las NICs de VM1, VM2 y VM3. Es lo que hace fácil escalar horizontalmente: para agregar capacidad solo se mete otra VM al pool, sin cambiar la IP que usan los clientes ni ninguna otra configuración. (Evidencias 6 y 16.)

**3. ¿Cuál es el propósito del *Health Probe*?**

Revisar periódicamente si cada instancia del pool está viva, para que el LB **solo le mande tráfico a las sanas**. El nuestro intenta abrir una conexión TCP al puerto 3000 de cada VM cada 5 s; si una deja de responder, el LB la saca de la rotación hasta que vuelva.

Detalle interesante que vimos: un probe **TCP** solo comprueba que el puerto acepte conexiones, y eso lo hace el kernel aunque Node esté bloqueado calculando. O sea que una VM saturada sigue apareciendo como sana. Un probe **HTTP** a un endpoint de la app (por ejemplo `/`) sí detectaría que la VM no está respondiendo a tiempo.

**4. ¿Cuál es el propósito de la *Load Balancing Rule*? ¿Qué tipos de sesión persistente existen, por qué esto es importante y cómo puede afectar la escalabilidad del sistema?**

La regla **conecta las piezas**: dice qué tráfico que llega al front-end (IP, protocolo y puerto) se reparte a qué backend pool, a qué puerto del backend y con qué health probe. La nuestra toma TCP **80** del front-end y lo manda al puerto **3000** de las VMs sanas del pool.

Tipos de persistencia (*distribution mode*):

| Modo | Hash | Comportamiento |
|---|---|---|
| **Ninguna** (default, la nuestra) | 5 tuplas: IP origen, puerto origen, IP destino, puerto destino, protocolo | Cada conexión nueva puede caer en cualquier instancia. |
| **IP de cliente** | 2 tuplas: IP origen, IP destino | Todas las peticiones de una misma IP de cliente van a la misma instancia. |
| **IP de cliente y protocolo** | 3 tuplas: IP origen, IP destino, protocolo | Igual que la anterior, pero separando por protocolo. |

**Por qué importa:** si la aplicación guarda estado en memoria (una sesión de usuario, un carrito), el cliente necesita volver siempre al mismo servidor; si no, "pierde" su sesión.

**Cómo afecta la escalabilidad:** la persistencia **desbalancea la carga**. Si muchos usuarios salen por la misma IP (una oficina o una universidad detrás de NAT), todos caen en la misma VM mientras las demás están libres, y agregar VMs no ayuda a esos usuarios. En nuestra prueba, con persistencia por IP de cliente, **todas** las ejecuciones de newman (que salen de la misma IP) habrían caído en una sola VM y el balanceo no habría servido de nada. Por eso lo ideal es que la app sea **stateless** (el estado en una BD o caché compartida) y dejar la persistencia en *Ninguna*. Aun sin persistencia vimos un efecto parecido en pequeño: cada newman reutiliza su conexión TCP, así que todas sus peticiones caen en la misma VM.

**5. ¿Qué es una *Virtual Network*? ¿Qué es una *Subnet*? ¿Para qué sirven los *address space* y *address range*?**

- **Virtual Network (VNet):** la red privada y aislada de nuestros recursos en Azure, el equivalente a la red de un datacenter propio. Ahí las VMs se comunican entre sí y con el LB por IPs privadas.
- **Subnet:** una división de la VNet. Sirve para separar recursos (por ejemplo web, BD y administración) y aplicarles reglas distintas (NSG, tablas de rutas).
- **Address space:** el rango total de IPs privadas de la VNet, en notación CIDR. El nuestro es `10.1.0.0/16`, o sea 65.536 direcciones (10.1.0.0 – 10.1.255.255).
- **Address range:** el pedazo de ese espacio que le toca a cada subred. El nuestro es `10.1.0.0/24` (256 direcciones, de las cuales Azure se reserva 5). De ahí salieron las IPs de las VMs: 10.1.0.4, 10.1.0.5 y 10.1.0.6.

(Evidencias 1 a 3.)

**6. ¿Qué son las *Availability Zones* y por qué seleccionamos 3 diferentes zonas? ¿Qué significa que una IP sea *zone-redundant*?**

Las **zonas de disponibilidad** son ubicaciones físicamente separadas dentro de una misma región: uno o más datacenters con su propia energía, refrigeración y red. Pusimos VM1, VM2 y VM3 en las zonas 1, 2 y 3 para que, si se cae un datacenter completo (un corte de energía, un incendio, una falla de red), **las otras dos VMs sigan atendiendo** y el sistema no se caiga. Esto suma alta disponibilidad a la escalabilidad.

Una IP **zone-redundant** no vive en una sola zona: Azure la replica en todas las zonas de la región, así que sigue funcionando aunque una zona falle. Por eso el front-end del LB es zone-redundant: si la IP de entrada estuviera solo en la zona 1, una caída de esa zona tumbaría todo el sistema aunque las VMs de las zonas 2 y 3 estuvieran bien. (Evidencia 5.)

**7. ¿Cuál es el propósito del *Network Security Group*?**

Es el **firewall de red** de las VMs. Tiene reglas de entrada y salida que permiten o niegan tráfico según origen, destino, puerto y protocolo, y se evalúan por **prioridad** (el número más bajo va primero). El nuestro (`SCALABILITY_LAB_NSG`) deja entrar **22** (SSH, para administración) y **3000** (la app). Sin la regla del 3000, ni el LB ni el health probe podrían llegar a la FibonacciApp. Con el LB Estándar esto es obligatorio, porque es seguro por defecto: si el NSG no lo permite, el tráfico no pasa. Usamos **un solo NSG compartido** por las 3 VMs para que las reglas sean idénticas en todos los nodos. (Evidencia 10.)

### Informe de newman 1 (Punto 2): vertical vs. horizontal


| Criterio | Vertical – VM pequeña | Vertical – VM escalada | **Horizontal – LB + 3 VMs** |
|---|---|---|---|
| Tamaño(s) | _(Parte 1)_ | _(Parte 1)_ | 2× D2s_v3 + 1× D2s_v4 (2 vCPU / 8 GB c/u) |
| Peticiones exitosas (2 en paralelo) | _(Parte 1)_ / 20 | _(Parte 1)_ / 20 | **20 / 20 (100 %)** |
| Tiempo promedio por petición | _(Parte 1)_ | _(Parte 1)_ | **19,6 s y 24,1 s** (una VM por flujo) |
| Estabilidad (desv. est.) | _(Parte 1)_ | _(Parte 1)_ | 277 ms y 424 ms |
| CPU máxima por VM | _(Parte 1)_ | _(Parte 1)_ | ~50 % (un núcleo de dos) |
| Costo aprox. de cómputo | _(Parte 1)_ | _(Parte 1)_ | ~0,11 USD/h por VM según el portal (D2s_v3) → **≈ 0,33 USD/h** las 3, más IPs públicas. El LB Estándar entró en las horas gratis de la cuenta de estudiante. |

**Conclusiones del comparativo:**
- **Tiempo de respuesta:** cada petición individual tarda prácticamente lo mismo en una VM grande que en una pequeña, porque el cálculo es de un solo hilo y depende de la velocidad de un núcleo, no de cuántos haya. La diferencia está en la **concurrencia**: en una sola VM las peticiones simultáneas hacen fila; con el LB cada flujo cae en una VM distinta y no espera.
- **Peticiones exitosas:** con balanceo horizontal y un nodo por flujo concurrente no hubo fallos. Los fallos solo aparecieron cuando hubo más flujos que nodos (punto 3).
- **Costo:** el horizontal cuesta más por hora (3 máquinas más el LB), pero se puede ajustar a la demanda (agregar o quitar nodos, o automatizarlo con un VM Scale Set) y además da **alta disponibilidad** con las 3 zonas. El vertical tiene un techo (el tamaño máximo de VM), requiere **reiniciar la VM para cambiar de tamaño** (hay caída del servicio) y sigue siendo un punto único de falla.
- **Escenario de calidad:** con el balanceador y suficientes nodos para la concurrencia, **todas las peticiones se responden y la CPU de cada nodo se mantiene por debajo del 70 %** (~50 %), así que el escenario se cumple. Con más flujos concurrentes que nodos (punto 3) la CPU sigue por debajo del 70 %, pero ya no se responden todas las peticiones, así que para cumplirlo hay que escalar a un nodo por flujo concurrente.

### Diagrama de despliegue

![DiagramaDespliege.png](images/part2/DiagramaDespliege.png)

### Limpieza

Al terminar las pruebas las VMs se dejan **Detenidas (desasignadas)**, para que no cobren cómputo, y después de la entrega se elimina el grupo de recursos `SCALABILITY_LAB` completo, porque los discos y las IPs públicas siguen cobrando aunque las VMs estén apagadas.
