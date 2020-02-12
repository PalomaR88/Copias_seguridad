# Copias de seguridad
Formas que ayudan en la pérdida de datos:
- Redundancia. Este método permite borrar las réplicas, luego, ante un fallo humono, no es muy eficaz. 
- Backup. Copia de seguridad. Permite guardar los datos, no solo actuales, sino históricos. 

La suma de ambas es la forma correcta. 


## 1. Interrogantes para establecer las copias de seguridad
Para guardar los datos de una organización hay que establecer una estrategia que guarde cada cierto tiempo y determinados documentos, porque no todo puede guardarse.

### 1.1. ¿Qué?
Lo que se pueda recoger de otra fuente no se guarda. Podríamos dividir en dos partes:
#### Datos: 
- /home (o directorio del usuario de Windows: Users\\) -> esto es muy grande, luego lo que se suele hacer es habilitar un directorio específico donde guardar los documentos importantes del usuario.
- /var/lib -> no completo, sino los datos de la base o bases de datos que se guardan aquí. Por ejemplo: /var/lib/ldap.
- /var/www
- logs
- /opt/?

#### Configuración:
- /etc

### 1.2. ¿Cómo?
Las copias pueden ser **completa** o según los **cambios**. Los cambios pueden ser:
- Incremental: se guarda una completa y se van guardando los cambios con respecto a esa copia completa.
- Diferencial: se guardan los cambios pero con respecto al dia anterior. A diferencia del anterior, los cambios solo se guardan una vez. 
- Delta diferencial: guarda los binarios que se han cambiado. Y para rescatarlos tiene que ir mirando todos los cambios que se han realizado anteriormente. 


### 1.3. ¿Cuándo?
No todos los datos deben tener un periodo igual, porque los datos cambian con una periocidad diferente dependiendo del tipo y uso de datos que se quiere guardar. 

Cuando se necesitan muchas copias de seguridad se usan snapshot (se inventó para esto), hace una "fotografía" del sistema en el momento y a partir de ahí se crea la copia. Después se puede borrar la snapshot.

Lo recomendable es realizar una copia de seguridad todos los días, pero no necesariamente de todos los ficheros. 

Una vez copiado, se archiva (suele usarse .tar). Y para que ocupe menos puede ser que se comprima. 
code G
Después, también hay que decidir cuando se hace una copia de seguridad completa. Y cuanto tiempo se guarda permanentemente.

### 1.4. ¿Dónde? 
**Ubicación física**
- Primero se almacenan localmente. Problema, la accesibilidad de estos datos. 
- Guardar otra en otra ubicación. Normalemnte, en un sitio seguro y suficientemente alejado. Y esto es normativa, porque tiene que haber un filtro, pero es muy según qué datos. 

**Medio**
Hay que hacer copias de las copias, porque todos los dispositivos se degradan. Por eso hay que hacer periodicamente copias de las copias. 

También se puede hacer a través de la nube (que es lo mismo pero de la integridad de las copias se encarga otro). Esto se suele hacer cifrando las copias antes. 
