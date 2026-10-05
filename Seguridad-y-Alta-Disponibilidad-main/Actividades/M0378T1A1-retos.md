# Incidentes de seguridad - Juego de Rol

**Módulo:** 0378 Seguridad y Alta Disponibilidad

## Reto 1 - Infección por ransomware

### Introducción
La empresa tiene una oficina con varios ordenadores, una red con wifi y conexión a internet. Utiliza la información para contactar con clientes y proveedores, mantener una página web sencilla, hacer las facturas e intercambiar datos con la gestoría. 

### Descripción del incidente
Un martes, uno de los comerciales descubrió que no podía entrar en su ordenador. En la pantalla apareció un mensaje que pedía dinero a cambio de devolverle el acceso. En menos de una hora, la mayoría de los comerciales tenían el mismo problema. Se trataba de un ransomware, que es un tipo de malware que cifra los archivos del equipo y exige un rescate para poder recuperarlos.

Al investigar el origen, se comprobó que el ataque empezó en el ordenador del primer comercial afectado. Este empleado explicó que el día anterior, a última hora, había recibido un correo de un cliente que no recordaba. El correo llevaba un archivo adjunto que descargó y que no aparecía nada. Pensó que había sido un error y reenvió el archivo a otros compañeros para ver si ellos podían abrirlo. De esta forma, el malware se extendió a otros equipos de la oficina. Todo indica que el correo era falso y que lo envió un ciberdelincuente con la intención de conseguir dinero mediante un ataque de phishing. En este caso, el error no fue cometido con mala intención, sino por descuido y falta de conocimientos.

Los equipos afectados fueron los puestos de trabajo de los comerciales y los discos conectados al ordenador del primer afectado, que quedaron cifrados. Aun así, hubo una parte positiva: las copias de seguridad estaban guardadas en un disco externo que no estaba conectado al ordenador, por lo que el ransomware no pudo cifrarlas.

![ransomware](img/ransomware.jpg)

### Causas del incidente
El incidente empezó por el error de una persona, pero no debería ser posible que el fallo de un solo empleado pare la mitad de una oficina. Esto demuestra que la empresa tenía varios puntos débiles. En primer lugar, faltaba formación. El empleado no supo reconocer un correo sospechoso y no sabía que no debía reenviar un archivo dudoso a sus compañeros. En segundo lugar, no existían procedimientos claros sobre qué hacer con los correos extraños ni a quién avisar. Por último, las medidas técnicas no fueron suficientes, ya que el antimalware y el filtro de correo no detuvieron el archivo, y un solo equipo infectado pudo contagiar a otros con mucha rapidez.

### Consecuencias
Las consecuencias para la empresa son importantes. Con la mitad de la oficina parada, no se pueden tramitar pedidos, y esto supone una pérdida de dinero. A esto se suman los costes de reparación, ya que hay que limpiar los equipos, restaurar la información y probablemente contratar a personas especializadas. También se ha interrumpido el servicio a los clientes, los teléfonos no dejan de sonar y los empleados no saben qué explicaciones dar.

Además de los daños económicos, existe un daño a la imagen de la empresa. Los clientes pueden perder la confianza y pensar que no es una empresa segura, lo que podría llevarles a elegir a la competencia. También disminuye el rendimiento, porque los trabajadores pierden muchas horas de trabajo hasta que todo vuelve a la normalidad. Por último, si el ataque hubiera afectado a datos personales de clientes, la empresa podría tener problemas legales y recibir sanciones por no cumplir la normativa de protección de datos.

### Cómo resolver el incidente
Lo primero que hay que hacer es aislar los equipos infectados, desconectándolos de la red y del wifi para evitar que el malware se propague. Después hay que avisar al responsable de la empresa y al soporte informático, y pedir a todos los empleados que no abran más archivos sospechosos.

Por otro lado, hacer caso a los atacantes y pagar no garantiza que se recuperen los datos. Antes de limpiar los equipos hay que guardar las evidencias, como capturas de pantalla del mensaje, el correo original con el archivo adjunto y las horas en las que ocurrieron los hechos. Estas pruebas sirven para presentar una denuncia.

Una vez hecho esto, se deben formatear los equipos infectados, reinstalar el sistema y recuperar la información desde la copia de seguridad. También se tiene que cambiar todas las contraseñas, sobre todo las del correo y las de los servicios importantes. Por último, hay que informar a los clientes y toda persona que podría haber sido afectada, explicando que se ha producido un problema técnico, que se está trabajando para solucionarlo y que se les avisará cuando el servicio vuelva a la normalidad.

### Cómo prevenir el incidente
Para evitar que esto vuelva a ocurrir, la medida más importante es la formación de los empleados. Deben aprender a reconocer correos falsos, a no abrir archivos adjuntos dudosos y a avisar al responsable antes de reenviar nada sospechoso. También es necesario contar con un protocolo de actuación que explique qué hacer y a quién llamar desde el primer momento.

En el aspecto técnico, la empresa debe tener un antimalware actualizado en todos los equipos y un filtro de correo que bloquee el spam y el phishing. También es importante mantener el software siempre actualizado, ya que las actualizaciones corrigen fallos de seguridad que los atacantes pueden aprovechar. Un cortafuegos y una red bien organizada ayudarían a que un equipo infectado no pueda contagiar fácilmente a los demás. Además, cada usuario debería tener solo los permisos que necesita para su trabajo, de manera que el malware llegue menos lejos.

Por último, las copias de seguridad son fundamentales. Deben hacerse de forma periódica, guardarse desconectadas de la red y en más de un lugar, y hay que comprobar de vez en cuando que se pueden restaurar correctamente. También puede ser útil contratar un seguro de ciberriesgos que ayude a cubrir los gastos si se produce otro ataque.

### Lo aprendido
Con este ejercicio se ha aprendido que un solo correo puede paralizar una empresa entera si no hay formación ni medidas de seguridad adecuadas. El factor humano es el punto más débil, pero también se puede reforzar con formación y normas claras. Las copias de seguridad desconectadas fueron lo que salvó a la empresa, y por eso es imprescindible tenerlas y comprobarlas. Ante un correo sospechoso, lo correcto es no abrirlo, no reenviarlo y avisar al responsable de informática. También se ha comprendido que es más barato prevenir que reparar, y que toda empresa necesita un plan de respuesta antes de que ocurra un incidente.

### Conclusión
El incidente empezó con un descuido, pero se convirtió en un problema grave por la falta de formación, de procedimientos y de medidas técnicas. La empresa pudo recuperarse gracias a las copias de seguridad, aunque sufrió pérdidas económicas y daños en su imagen. Para evitar que se repita, es necesario formar al personal, mantener los sistemas protegidos y actualizados, y tener un plan claro para actuar ante cualquier incidente de seguridad.
