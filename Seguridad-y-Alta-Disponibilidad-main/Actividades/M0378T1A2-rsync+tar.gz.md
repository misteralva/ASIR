Copias de seguridad en Linux con rsync, tar.gz y systemd
Módulo M0378 – Seguridad · Informe técnico y guía práctica paso a paso
Este documento tiene tres partes. La Parte 1 responde a las preguntas teóricas del informe. La Parte 2 es una guía práctica, muy detallada, para hacer las copias con rsync y tar.gz y automatizarlas con systemd. La Parte 3 repite la práctica dentro de la máquina virtual, mostrando lo que aparece en pantalla en cada paso.
Parte 1. Parte escrita (teoría)
1. Copias de seguridad
¿Qué son las copias de seguridad y por qué debemos hacerlas?
Una copia de seguridad (en inglés, backup) es una copia de los datos que se guarda en otro lugar, para poder recuperarlos si los originales se pierden o se dañan.
Debemos hacer copias porque los datos se pueden perder por muchos motivos:
• Fallo de un disco duro.
• Borrado por error de un fichero.
• Virus o ransomware (programa que cifra los datos y pide dinero).
• Robo del equipo.
• Incendio, inundación u otro desastre.
• Errores al actualizar un sistema.
Sin una copia, la pérdida puede ser definitiva. Además, una copia permite volver a una versión anterior de un fichero. La idea principal es esta: un dato que solo existe en un sitio no está protegido.
¿Qué criterios se usan para clasificar la información y decidir qué datos guardar?
No todos los datos tienen la misma importancia. Antes de copiar, hay que clasificarlos con estos criterios:
• Importancia (criticidad): si se pierde ese dato, ¿se para el trabajo de la empresa?
• Confidencialidad: datos personales, contraseñas o datos bancarios. Estos necesitan además cifrado.
• Frecuencia de cambio: los datos que cambian mucho necesitan copias más frecuentes.
• Obligaciones legales: por ejemplo, la ley de protección de datos (RGPD) o la conservación de facturas durante varios años.
• Dificultad de recuperarlos: si se pueden volver a crear fácilmente o no.
Con estos criterios, la información se puede clasificar en cuatro grupos: crítica, importante, útil y prescindible (por ejemplo, archivos temporales o la caché). Se copia primero y con más frecuencia lo crítico.
¿De qué depende la duración de una copia completa?
• Cantidad de datos: más datos tardan más.
• Velocidad del disco de origen y del disco de destino: un disco lento hace más lenta la copia.
• Velocidad de la red, si la copia se envía a otro equipo.
• Número de ficheros: muchos ficheros pequeños tardan más que pocos ficheros grandes.
• Compresión y cifrado: gastan procesador y alargan el tiempo.
• Carga del sistema: si el equipo está ocupado con otras tareas, la copia va más despacio.
¿Qué tipos de discos se pueden usar para las copias? ¿Influyen en el tiempo? ¿Cuándo se hacen las copias y las restauraciones?
Soporte
Características
Disco duro (HDD)
Barato y con mucha capacidad. Es lento.
Disco SSD
Muy rápido, pero más caro por cada GB.
Disco externo USB
Cómodo y portátil. Su velocidad depende del puerto USB.
Cinta magnética (LTO)
Muy barata para guardar mucho tiempo. Es lenta para buscar datos.
NAS o SAN
Almacenamiento en red, para empresas.
Nube
Almacenamiento en internet de otra empresa.
Sí influyen en el tiempo: un SSD o una buena red hacen la copia mucho más rápida que un HDD antiguo o una conexión lenta.
Las copias se suelen hacer por la noche o en fin de semana, porque hay menos gente trabajando y la copia no ralentiza los sistemas. A esto se le llama ventana de copia. Las restauraciones se hacen cuando hay una incidencia (un fallo o un borrado), y lo antes posible. También se hacen restauraciones de prueba cada cierto tiempo para comprobar que las copias sirven.
¿Qué es una copia incremental y qué diferencia tiene con la diferencial?
Primero hay que conocer la copia completa: copia todos los datos, siempre.
• Copia incremental: copia solo lo que ha cambiado desde la última copia (sea completa o incremental).
• Copia diferencial: copia solo lo que ha cambiado desde la última copia completa.
Tipo
Qué copia
Ventaja
Inconveniente
Completa
Todo
Restaurar es fácil
Tarda mucho y ocupa mucho
Incremental
Cambios desde la última copia
Rápida y ocupa poco
Para restaurar hacen falta la completa y todas las incrementales, en orden
Diferencial
Cambios desde la última completa
Para restaurar solo hace falta la completa y la última diferencial
Cada día ocupa más que la anterior
Ejemplo: el lunes se hace una copia completa. El martes cambia el fichero A. El miércoles cambia el fichero B.
• Con copias incrementales: el martes se guarda A y el miércoles se guarda B.
• Con copias diferenciales: el martes se guarda A y el miércoles se guardan A y B.
¿Dónde podemos almacenar la copia de seguridad?
• En otro disco del mismo equipo (es lo menos seguro, porque si se rompe el equipo se pierde todo).
• En un disco externo USB.
• En un NAS de la oficina.
• En otro servidor, dentro o fuera de la empresa.
• En cintas.
• En la nube.
• En otra sede o edificio, para estar a salvo de incendios e inundaciones.
¿En qué consiste la estrategia 3-2-1?
Es una regla muy conocida para hacer copias seguras:
• 3 copias de los datos: el original y dos copias más.
• 2 soportes distintos: por ejemplo, un disco duro y la nube.
• 1 copia fuera del lugar de trabajo (offsite), para protegerse de robos o desastres.
Así, es muy difícil perderlo todo a la vez.
¿Qué herramientas de cifrado permiten proteger las copias?
Cifrar significa convertir los datos en algo ilegible sin una clave. Hay dos momentos que proteger: los datos guardados y los datos durante el envío.
• GPG: cifra ficheros, por ejemplo un .tar.gz, con una contraseña o una clave.
• OpenSSL: herramienta de cifrado que viene en Linux.
• 7-Zip: crea archivos comprimidos con cifrado AES-256.
• LUKS: cifra un disco o una partición entera en Linux.
• VeraCrypt: crea volúmenes cifrados.
• restic, BorgBackup y duplicity: programas de copias de seguridad que cifran automáticamente.
• SSH: cifra los datos mientras viajan por la red. Rsync lo usa para enviar copias de forma segura.
¿Qué debemos considerar al hacer copias en la nube?
• Seguridad: que los datos viajen y se guarden cifrados, de preferencia con una clave que solo tengamos nosotros.
• Dónde están los servidores: la ley de protección de datos (RGPD) exige cuidado con los datos personales fuera de la Unión Europea.
• Coste: se paga por el espacio y, a veces, por descargar los datos.
• Velocidad: subir y bajar muchos datos depende de nuestra conexión a internet.
• Fiabilidad del proveedor: qué garantías ofrece y qué pasa si cierra.
• Cuánto tiempo guarda las copias y si se pueden restaurar con facilidad.
• Acceso: usar contraseñas fuertes y autenticación en dos pasos.
¿Qué es un SAN?
Una SAN (Storage Area Network, red de área de almacenamiento) es una red de alta velocidad que conecta servidores con sistemas de almacenamiento. Los servidores ven el almacenamiento como si fuera un disco propio (acceso a nivel de bloque). Usa tecnologías como Fibre Channel o iSCSI. Es muy rápida y fiable, pero cara, y se usa sobre todo en empresas grandes y centros de datos.
¿Qué es un NAS? Ejemplos de pago y libres
Un NAS (Network Attached Storage, almacenamiento conectado a la red) es un equipo con discos que comparte ficheros por la red local. Los usuarios acceden a carpetas compartidas (acceso a nivel de fichero) con protocolos como SMB/CIFS o NFS. Es más sencillo y barato que una SAN.
• De pago: Synology, QNAP, NetApp y Dell EMC.
• Libres (software gratuito): TrueNAS, OpenMediaVault y Rockstor.
2. Proceso desarrollado en la parte práctica con rsync y tar.gz
Creación y restauración de una copia de una carpeta de contenidos
En una misma máquina virtual (MV)
Con rsync:
rsync -avzh origen/ destino/
Esto copia el contenido de origen a destino. Para restaurar, se invierten origen y destino:
rsync -avzh destino/ origen/
Con tar.gz:
tar -czf destino/copia.tar.gz origen/
tar -xzf destino/copia.tar.gz -C restauracion/
La primera orden crea el archivo comprimido. La segunda lo extrae en la carpeta restauracion. Las opciones son: c crear, x extraer, z comprimir con gzip, f indicar el nombre del archivo y -C entrar en una carpeta antes de actuar.
En dos MV (una es el origen y la otra el destino)
Con rsync, se añade usuario@IP: delante de la ruta del equipo remoto. Los datos viajan por SSH, cifrados:
rsync -avzh origen/ usuario@192.168.1.78:/home/usuario/backups/
rsync -avzh usuario@192.168.1.78:/home/usuario/backups/ origen/
La primera orden hace la copia. La segunda restaura, porque trae los datos de vuelta.
Con tar.gz, se crea el archivo y se envía al otro equipo (con rsync o con una tubería de SSH):
tar -czf - origen/ | ssh usuario@192.168.1.78 'cat > /home/usuario/backups/copia.tar.gz'
Aquí el guion - hace que tar no cree un fichero, sino que envíe su resultado por la tubería |. SSH lo lleva al otro equipo, y cat > lo guarda en un fichero.
Si hay que hacer un script de copias, ¿rsync o tar.gz?
• rsync es mejor para mantener una carpeta sincronizada. Copia solo lo que ha cambiado, así que es rápido y gasta poca red. Pero la copia queda como una carpeta normal y solo guarda la última versión.
• tar.gz es mejor para guardar versiones históricas. Cada copia es un único fichero comprimido con la fecha, así se puede volver a cualquier día. Además, ocupa menos espacio gracias a la compresión.
¿Cuál es la mejor opción?
No hay una opción mejor para todos los casos. Mi elección es usar las dos juntas: rsync para la copia diaria rápida al equipo remoto, y tar.gz con copias completa e incrementales para tener un histórico de versiones. Si solo pudiera elegir una, elegiría rsync para copias remotas automáticas, por ser más rápido y eficiente en red. Si lo importante fuera poder volver a versiones antiguas, elegiría tar.gz.
3. Rsync
¿Qué es rsync? ¿En qué contexto se usa?
Rsync (remote synchronization, sincronización remota) es una herramienta de Linux que copia y sincroniza ficheros y carpetas, en el mismo equipo o entre equipos distintos. Su gran ventaja es que solo copia lo que ha cambiado: compara el tamaño y la fecha de modificación de cada fichero y, con su algoritmo de transferencia por diferencias, solo envía las partes modificadas.
Se usa para:
• Hacer copias de seguridad.
• Sincronizar carpetas entre servidores.
• Migrar datos de un equipo a otro.
• Crear réplicas (espejos) de una web o de un repositorio.
Ejemplo de uso entre dos MV diferentes
rsync -avzh /home/alumno/origen/ punky@192.168.1.78:/home/punky/backups/
Explicación de cada parte:
• -a (archivo): conserva permisos, propietario, grupo, fechas y enlaces simbólicos.
• -v (verbose): muestra lo que va haciendo.
• -z: comprime los datos mientras viajan por la red.
• -h: muestra los tamaños en forma fácil de leer.
• /home/alumno/origen/: la carpeta que se copia. La barra final significa «copia el contenido».
• punky@192.168.1.78:/home/punky/backups/: usuario, IP y carpeta del equipo remoto.
Sin la barra final (origen), rsync copia la carpeta entera y crea backups/origen/. Con la barra (origen/), copia solo lo que hay dentro.
4. Tar.gz
¿Qué es el tar.gz?
tar (tape archive) junta muchos ficheros y carpetas en un solo archivo (.tar), sin comprimir. gzip comprime ese archivo y le añade la extensión .gz. El resultado es un único fichero .tar.gz, fácil de mover, enviar y guardar. Un fichero .tgz es lo mismo.
¿Cómo se usa el tar.gz para copias incrementales y diferenciales?
Tar usa la opción --listed-incremental=FICHERO.snar. El fichero .snar (snapshot) guarda una lista de lo que ya se copió. Con ella, tar sabe qué ha cambiado.
Copia incremental: se usa siempre el mismo fichero .snar.
# 1.ª vez: como no hay .snar, tar copia todo (copia completa)
tar --listed-incremental=datos.snar -czf completa.tar.gz origen/

# Después de cambiar algo: solo copia lo nuevo desde la última vez
tar --listed-incremental=datos.snar -czf incremental1.tar.gz origen/
Cada ejecución actualiza datos.snar, así que la siguiente copia solo guarda los cambios desde la anterior.
Copia diferencial: hay que conseguir que cada copia compare siempre con la completa. Para eso, se guarda una copia del .snar de la completa y se restaura antes de cada diferencial.
# Copia completa y guardado de su .snar
tar --listed-incremental=completa.snar -czf completa.tar.gz origen/

# Antes de cada diferencial, partimos del .snar de la completa
cp completa.snar dif.snar
tar --listed-incremental=dif.snar -czf diferencial1.tar.gz origen/

# La siguiente diferencial vuelve a empezar desde el .snar de la completa
cp completa.snar dif.snar
tar --listed-incremental=dif.snar -czf diferencial2.tar.gz origen/
Así, diferencial2 contiene todo lo cambiado desde la completa, no solo lo último.
Restaurar: primero la completa y después las incrementales, en orden. Con --listed-incremental=/dev/null le decimos a tar que estamos restaurando.
tar --listed-incremental=/dev/null -xzf completa.tar.gz -C restauracion/
tar --listed-incremental=/dev/null -xzf incremental1.tar.gz -C restauracion/
En el caso de la diferencial, solo hacen falta la completa y la última diferencial.
Parte 2. Guía práctica paso a paso
Esta guía está pensada para seguirla en orden, sin saltarse pasos. Cada orden va acompañada de una explicación sencilla de lo que hace.
Antes de empezar
Qué necesitas:
• Dos máquinas virtuales (MV) con Ubuntu o Debian. La llamaremos MV1 (origen: aquí están los datos) y MV2 (destino: aquí se guardan las copias).
• Las dos MV conectadas entre sí, con Adaptador puente o con una red solo anfitrión en VirtualBox.
Nombres de ejemplo que usaremos (cámbialos por los tuyos):
Dato
Valor de ejemplo
Usuario en MV1
alumno
Usuario en MV2
punky
IP de MV2
192.168.1.78
Para saber la IP de una máquina, escribe en ella:
ip a
Esta orden muestra las tarjetas de red. Busca la línea que empieza por inet en la tarjeta que está en uso (por ejemplo enp0s3).
Paso 0. Instalar los programas necesarios
En MV1 y en MV2:
sudo apt update
sudo apt install rsync openssh-server
• sudo ejecuta la orden con permisos de administrador.
• apt update actualiza la lista de programas disponibles.
• apt install instala los programas. rsync es la herramienta de copias y openssh-server permite conectarse por SSH a esa máquina.
Comprueba que SSH está funcionando en MV2:
sudo systemctl status ssh
Debe aparecer en verde la palabra active (running). Para salir de la pantalla, pulsa la tecla q.
Comprueba que MV1 ve a MV2 (desde MV1):
ping -c 3 192.168.1.78
ping envía pequeños mensajes a la otra máquina y -c 3 indica que sean solo 3. Si aparecen respuestas, hay conexión. Si no, revisa la configuración de red de VirtualBox.
Práctica 1. Copia en local con rsync y con tar.gz
Esta parte se hace solo en MV1.
Paso 1. Crear las carpetas y los ficheros de prueba
cd ~
mkdir origen destino
touch origen/file{1..10}
ls origen
• cd ~ va a tu carpeta personal.
• mkdir origen destino crea dos carpetas.
• touch origen/file{1..10} crea 10 ficheros vacíos llamados file1, file2... hasta file10.
• ls origen lista los ficheros para comprobar que se han creado.
Para que tengan algo dentro:
echo 'texto de prueba' > origen/file1
echo escribe un texto y el signo > lo guarda dentro del fichero file1.
Paso 2. Copia completa con rsync
rsync -avzh origen/ destino/
Significado de cada parte:
• -a: modo archivo. Mantiene permisos, fechas, propietario, grupo y enlaces.
• -v: muestra por pantalla qué va copiando.
• -z: comprime los datos mientras se copian.
• -h: muestra los tamaños de forma fácil de leer.
• origen/: la carpeta de origen. La barra final es importante (ver el Paso 5).
• destino/: la carpeta donde se guarda la copia.
Comprueba el resultado:
ls -l destino
ls -l muestra la lista larga, con permisos, propietario, tamaño y fecha de cada fichero. Deben aparecer los 10 ficheros.
Paso 3. Copia incremental con rsync
Primero, vuelve a ejecutar la misma orden sin cambiar nada:
rsync -avzh origen/ destino/
No se copia ningún fichero, porque rsync detecta que no ha cambiado nada. Ahora haz cambios:
echo 'cambio nuevo' >> origen/file1
touch origen/file11
rsync -avzh origen/ destino/
• >> añade texto al final del fichero (con un solo > se borraría el contenido anterior).
• touch origen/file11 crea un fichero nuevo.
Esta vez, rsync copia solo file1 y file11. Así funciona rsync en las copias incrementales: compara el tamaño y la fecha de cada fichero entre origen y destino, y solo copia los que son nuevos o distintos. En los ficheros grandes, además, envía solo las partes que cambiaron.
Paso 4. Comprobar los atributos, los usuarios y las modificaciones
Mira los detalles de los ficheros en origen y en destino y compáralos:
ls -l origen
ls -l destino
• ¿Se mantienen los atributos? Sí. Gracias a -a, se mantienen los permisos y las fechas de modificación.
• ¿Se mantienen los usuarios? Solo si rsync se ejecuta como administrador (sudo). Un usuario normal no puede dar ficheros a otro usuario. Para comprobarlo:
sudo chown root:root origen/file2
sudo rsync -avzh origen/ destino/
ls -l destino/file2
chown root:root cambia el propietario y el grupo del fichero a root. Después de copiar con sudo, el fichero en el destino también pertenece a root.
• ¿Qué pasa si modifico un fichero en origen? Se modifica en el destino solo cuando se vuelve a ejecutar rsync. No es automático.
• ¿Y si borro un fichero en origen? Por defecto, rsync no lo borra del destino. Para que lo borre hay que añadir --delete. Antes de usarlo, se prueba con -n (dry run, simulación):
rm origen/file3
rsync -avzh --delete -n origen/ destino/
Con -n rsync solo muestra lo que haría (aparecerá deleting file3) sin tocar nada. Quita el -n solo si estás seguro.
Paso 5. La barra final: origen/ frente a origen
¿Es lo mismo rsync -avzh origen/ destino/ que rsync -avzh origen destino/? No es lo mismo. Pruébalo:
mkdir destino2
rsync -avzh origen destino2/
ls destino2
• Con origen/ (con barra): se copia el contenido de la carpeta. Los ficheros quedan directamente dentro de destino/.
• Con origen (sin barra): se copia la carpeta entera. Se crea destino2/origen/ y los ficheros quedan dentro.
Paso 6. Restaurar con rsync
Para restaurar, se invierten origen y destino. Primero simulamos una pérdida de datos:
rm origen/file5 origen/file6
ls origen
rsync -avzh destino/ origen/
ls origen
Los dos ficheros borrados vuelven a aparecer, porque ahora la carpeta destino es la fuente de la copia.
Paso 7. Repetir con tar.gz: copia completa
tar -czvf destino/copia_completa.tar.gz origen/
tar -tzvf destino/copia_completa.tar.gz
• -c: crear un archivo.
• -z: comprimirlo con gzip.
• -v: mostrar los ficheros que se incluyen.
• -f destino/copia_completa.tar.gz: el nombre del archivo que se crea.
• -t: listar el contenido sin extraerlo (así comprobamos que la copia está bien).
Paso 8. Copia incremental con tar.gz
rm -f datos.snar
tar --listed-incremental=datos.snar -czvf destino/tar_completa.tar.gz origen/
echo 'nuevo' > origen/file20
echo 'cambio' >> origen/file1
tar --listed-incremental=datos.snar -czvf destino/tar_incremental1.tar.gz origen/
tar -tzvf destino/tar_incremental1.tar.gz
• rm -f datos.snar borra el fichero de memoria anterior, para empezar desde cero.
• La primera orden tar crea la copia completa y el fichero datos.snar (snapshot, la lista de lo que ya se copió).
• Después se cambian cosas en origen.
• La segunda orden tar, con el mismo datos.snar, guarda solo los cambios: es la incremental.
• La última orden lista el contenido. Solo deben aparecer file20 y file1 (además de la carpeta origen/).
Paso 9. Copia diferencial con tar.gz
tar --listed-incremental=completa.snar -czvf destino/dif_completa.tar.gz origen/
cp completa.snar dif.snar
echo 'uno' > origen/cambio1
tar --listed-incremental=dif.snar -czvf destino/dif_1.tar.gz origen/
cp completa.snar dif.snar
echo 'dos' > origen/cambio2
tar --listed-incremental=dif.snar -czvf destino/dif_2.tar.gz origen/
tar -tzvf destino/dif_2.tar.gz
• Se hace la copia completa y se guarda su completa.snar.
• Antes de cada diferencial, cp completa.snar dif.snar copia ese fichero. Así, tar siempre compara con la completa y no con la copia anterior.
• El resultado: dif_2.tar.gz contiene cambio1 y cambio2. Esa es la diferencia con la incremental.
Paso 10. Restaurar con tar.gz
mkdir restauracion
tar --listed-incremental=/dev/null -xzvf destino/dif_completa.tar.gz -C restauracion/
tar --listed-incremental=/dev/null -xzvf destino/dif_2.tar.gz -C restauracion/
diff -r origen restauracion/origen
• -x: extraer.
• -C restauracion/: extraer dentro de esa carpeta.
• --listed-incremental=/dev/null: avisa a tar de que estamos restaurando y no creando una copia nueva.
• Se extrae primero la completa y después la copia que corresponda (en la incremental, todas en orden; en la diferencial, solo la última).
• diff -r compara las dos carpetas. Si no muestra nada, son iguales y la restauración es correcta.
Práctica 2. Copia por la red a un equipo remoto
Ahora se usan las dos máquinas. Todo se ejecuta desde MV1.
Paso 11. Probar la conexión SSH
ssh punky@192.168.1.78
La primera vez pregunta si confías en la máquina. Escribe yes y después la contraseña de punky. Si entras, verás el prompt de MV2. Para volver a MV1, escribe exit.
Paso 12. Copia por red con rsync
ssh punky@192.168.1.78 'mkdir -p ~/backups'
rsync -avzh origen/ punky@192.168.1.78:backups/
• La primera orden ejecuta mkdir -p ~/backups en MV2 para crear la carpeta de destino. La opción -p evita error si ya existe.
• La segunda orden es el rsync de siempre, pero el destino lleva usuario@IP: delante. La ruta backups/ es relativa a la carpeta personal de punky.
• Rsync usa SSH por debajo, así que los datos viajan cifrados.
Comprueba en MV2:
ls -l ~/backups
Recuerda la diferencia entre origen/ y origen: con barra se copia el contenido, sin barra se copia la carpeta entera.
Paso 13. Copia por red con tar.gz
tar -czvf copia_red.tar.gz origen/
scp copia_red.tar.gz punky@192.168.1.78:backups/
• La primera orden crea el archivo .tar.gz en MV1.
• scp (secure copy) copia un fichero a otra máquina por SSH. Tiene la forma scp fichero usuario@IP:carpeta.
También se puede hacer todo en un solo paso, sin crear el fichero local:
tar -czf - origen/ | ssh punky@192.168.1.78 'cat > ~/backups/copia_pipe.tar.gz'
El - hace que tar no cree fichero, sino que envíe el resultado por la tubería |. SSH lo lleva a MV2, y cat > lo guarda en un fichero.
Paso 14. Restaurar por red
Con rsync, se invierte el sentido (ahora MV2 es el origen):
mkdir restauracion_red
rsync -avzh punky@192.168.1.78:backups/ restauracion_red/
Con tar.gz, se trae el archivo y se extrae:
scp punky@192.168.1.78:backups/copia_red.tar.gz .
tar -xzvf copia_red.tar.gz -C restauracion_red/
El punto . al final de scp significa «la carpeta donde estoy».
Práctica 3. Automatizar sin escribir la contraseña
Si el script tiene que pedir la contraseña cada vez, no puede ejecutarse solo. Hay dos soluciones.
Método A. sshpass (fácil, pero poco seguro)
sudo apt install sshpass
echo 'laContraseñaDePunky' > ~/.clave_ssh
chmod 600 ~/.clave_ssh
sshpass -f ~/.clave_ssh rsync -avzh -e ssh origen/ punky@192.168.1.78:backups/
• sshpass escribe la contraseña por nosotros.
• La contraseña se guarda en un fichero, y chmod 600 hace que solo tu usuario pueda leerlo.
• -f indica el fichero con la contraseña.
• -e ssh le dice a rsync que use SSH.
Por qué es poco seguro: la contraseña queda guardada en texto normal en un fichero. Si alguien lo lee, tiene acceso al servidor. Además, si se escribe en la propia orden (-p contraseña), queda visible en el historial y en la lista de procesos.
Método B. Claves SSH (el recomendado)
Una clave SSH es un par de ficheros: una clave privada (se queda en MV1 y nunca se comparte) y una clave pública (se copia a MV2). Cuando MV1 se conecta, MV2 comprueba que la privada encaja con la pública, y no hace falta contraseña.
En MV1:
ssh-keygen -t rsa -b 4096
ssh-copy-id punky@192.168.1.78
ssh punky@192.168.1.78
• ssh-keygen -t rsa -b 4096 crea el par de claves. -t rsa es el tipo y -b 4096 el tamaño, que cuanto mayor es más seguro. Pulsa Enter a todo para no poner contraseña a la clave (así los scripts pueden usarla solos).
• ssh-copy-id copia la clave pública al fichero ~/.ssh/authorized_keys de MV2. Pide la contraseña de punky por última vez.
• La tercera orden prueba la conexión. Ya no debe pedir contraseña. Si la pide, algo ha fallado y no hay que seguir.
Ahora el rsync funciona sin pedir nada:
rsync -avzh origen/ punky@192.168.1.78:backups/
Conclusión: el método B es más seguro porque no hay ninguna contraseña guardada en texto, y la clave privada solo está en tu equipo. Por eso usaremos las claves SSH en los scripts.
Práctica 4. Scripts de copia de seguridad en el equipo remoto
Un script es un fichero de texto con órdenes que se ejecutan una detrás de otra. Así no hay que escribirlas a mano cada vez. Haremos dos: uno con rsync y otro con tar.gz.
Paso 15. Script con rsync
En MV1, crea el fichero:
sudo nano /usr/local/bin/backup_rsync.sh
nano es un editor de texto sencillo. Pega este contenido, y para guardar pulsa Ctrl+O, después Enter, y para salir Ctrl+X.
#!/bin/bash
# Copia de seguridad con rsync a un equipo remoto
set -euo pipefail

# ---------- VARIABLES (cámbialas por las tuyas) ----------
ORIGEN="/home/alumno/origen/"
USUARIO="punky"
IP="192.168.1.78"
DESTINO="/home/punky/backups/rsync"

# ---------- COMPROBACIÓN ----------
if [ ! -d "$ORIGEN" ]; then
    echo "ERROR: no existe la carpeta de origen $ORIGEN"
    exit 1
fi

echo "[$(date '+%F %T')] Empezando la copia con rsync..."

# Comprueba la conexión y crea la carpeta remota si no existe
ssh -o BatchMode=yes -o ConnectTimeout=10 "$USUARIO@$IP" "mkdir -p $DESTINO"

# ---------- LA COPIA ----------
rsync -avzh -e "ssh -o BatchMode=yes" "$ORIGEN" "$USUARIO@$IP:$DESTINO/"

echo "[$(date '+%F %T')] Copia con rsync terminada correctamente."
Explicación de cada parte:
• #!/bin/bash: indica que el script se ejecuta con bash. Debe ir siempre en la primera línea.
• set -euo pipefail: hace que el script se pare si algo falla (-e), que dé error si usamos una variable que no existe (-u), y que detecte fallos dentro de una tubería (-o pipefail). Es muy importante para que un error no pase desapercibido.
• Las variables (ORIGEN, USUARIO...) guardan datos que usamos varias veces. Si cambia la IP, solo se cambia en un sitio. Se usan escribiendo $ delante.
• if [ ! -d "$ORIGEN" ]: pregunta si la carpeta de origen no existe (! es «no» y -d es «es una carpeta»). Si no existe, muestra un error y termina con exit 1. Un código distinto de 0 significa «ha fallado».
• $(date '+%F %T'): escribe la fecha y la hora, para que en el registro se vea cuándo ocurrió cada cosa.
• ssh -o BatchMode=yes: si SSH necesitara contraseña, falla en lugar de quedarse esperando. Es necesario cuando el script se ejecuta solo.
• -o ConnectTimeout=10: espera como máximo 10 segundos para conectar.
• mkdir -p $DESTINO: crea la carpeta de destino en el equipo remoto.
• rsync -avzh -e "ssh ...": hace la copia. -e indica el programa que se usa para conectar (SSH).
Paso 16. Script con tar.gz
sudo nano /usr/local/bin/backup_tar.sh
Pega este contenido:
#!/bin/bash
# Copia con tar.gz: completa los domingos e incremental el resto de días
set -euo pipefail

# ---------- VARIABLES ----------
ORIGEN="/home/alumno/origen"
USUARIO="punky"
IP="192.168.1.78"
DESTINO="/home/punky/backups/tar"
DIAS_GUARDAR=30

TRABAJO="$HOME/backup_tar"
SNAR="$TRABAJO/contenidos.snar"
FECHA=$(date +%F)

# ---------- COMPROBACIONES ----------
if [ ! -d "$ORIGEN" ]; then
    echo "ERROR: no existe la carpeta de origen $ORIGEN"
    exit 1
fi

mkdir -p "$TRABAJO"
echo "[$(date '+%F %T')] Empezando la copia con tar.gz..."

ssh -o BatchMode=yes -o ConnectTimeout=10 "$USUARIO@$IP" "mkdir -p $DESTINO"

# ---------- TIPO DE COPIA ----------
if [ "$(date +%u)" -eq 7 ] || [ ! -f "$SNAR" ]; then
    rm -f "$SNAR"
    TIPO="completa"
else
    TIPO="incremental"
fi

ARCHIVO="$TRABAJO/backup_${TIPO}_${FECHA}.tar.gz"

# ---------- CREAR EL TAR.GZ ----------
tar --listed-incremental="$SNAR" -czf "$ARCHIVO" \
    -C "$(dirname "$ORIGEN")" "$(basename "$ORIGEN")"

# ---------- ENVIAR AL EQUIPO REMOTO ----------
rsync -avh -e "ssh -o BatchMode=yes" "$ARCHIVO" "$USUARIO@$IP:$DESTINO/"
rm -f "$ARCHIVO"

# ---------- BORRAR COPIAS ANTIGUAS EN EL REMOTO ----------
ssh -o BatchMode=yes "$USUARIO@$IP" \
    "find $DESTINO -name '*.tar.gz' -mtime +$DIAS_GUARDAR -delete"

echo "[$(date '+%F %T')] Copia $TIPO con tar.gz terminada correctamente."
Explicación de las partes nuevas:
• TRABAJO y SNAR: carpeta de trabajo local y fichero .snar, que es la «memoria» de tar para saber qué se ha copiado ya.
• date +%u: da el día de la semana (1 es lunes y 7 es domingo).
• if ... -eq 7 || ... ! -f: si es domingo o si todavía no existe el .snar, se borra el .snar y se hace una copia completa. Si no, se hace una incremental.
• ARCHIVO=...: el nombre del fichero lleva el tipo y la fecha, por ejemplo backup_incremental_2026-10-16.tar.gz.
• tar --listed-incremental=...: crea la copia usando el .snar.
• -C "$(dirname "$ORIGEN")" "$(basename "$ORIGEN")": dirname da la carpeta padre y basename el nombre de la carpeta. Así el .tar.gz guarda solo origen/... y no toda la ruta.
• rsync ... "$ARCHIVO": envía el .tar.gz al equipo remoto. Después rm -f lo borra del equipo local, porque ya está guardado.
• find ... -mtime +30 -delete: en el equipo remoto, busca copias de más de 30 días y las borra, para que el disco no se llene.
Paso 17. Dar permisos y probar los scripts a mano
sudo chmod +x /usr/local/bin/backup_rsync.sh /usr/local/bin/backup_tar.sh
/usr/local/bin/backup_rsync.sh
/usr/local/bin/backup_tar.sh
• chmod +x da permiso de ejecución a los scripts. Sin esto, Linux no los ejecuta.
• Después se ejecutan a mano. Comprueba en MV2 que han llegado los ficheros con ls -lR ~/backups.
Importante: si los scripts no funcionan a mano, tampoco funcionarán con systemd. Si algo falla, mira la tabla de errores al final.
Práctica 5. Programar las copias con systemd
¿Por qué systemd y no crontab?
En esta práctica, en lugar de lo que propone el enunciado, no usamos crontab, que es una tabla donde se escriben tareas programadas, sino systemd. Solo como comparación, para ejecutar el script todos los días a las 19:30 con crontab bastaría esta línea (se edita con crontab -e):
30 19 * * * /usr/local/bin/backup_rsync.sh
Systemd también puede programar tareas, con unos ficheros llamados timers (temporizadores). Se necesitan dos ficheros por tarea:
• El service (.service): dice qué se ejecuta.
• El timer (.timer): dice cuándo se ejecuta.
Aspecto
crontab
systemd timer
Configuración
Una línea
Dos ficheros
Registro de errores
Solo correo o syslog
Automático, con journalctl
Esperar a que haya red
No
Sí (After=network-online.target)
Si el equipo estaba apagado
La tarea se pierde
Persistent=true la ejecuta al encender
Control de recursos
No
Sí (Nice= y otras opciones)
Rapidez de preparación
Muy rápida
Más lenta
Paso 18. Crear el service de rsync
sudo nano /etc/systemd/system/backup-rsync.service
[Unit]
Description=Copia de seguridad con rsync al equipo remoto
Wants=network-online.target
After=network-online.target

[Service]
Type=oneshot
User=alumno
ExecStart=/usr/local/bin/backup_rsync.sh
• /etc/systemd/system/ es la carpeta donde systemd busca las unidades creadas por el administrador.
• Description: texto que se ve en systemctl status.
• Wants= y After=network-online.target: pide que la red esté lista antes de empezar. Es necesario porque la copia es remota.
• Type=oneshot: el script se ejecuta, termina, y systemd lo da por finalizado.
• User=alumno: el usuario que ejecuta el script. Debe ser el usuario que tiene la clave SSH (cámbialo por el tuyo).
• ExecStart=: la ruta completa del script.
Paso 19. Crear el timer de rsync
sudo nano /etc/systemd/system/backup-rsync.timer
[Unit]
Description=Programa la copia con rsync todos los días a las 19:30

[Timer]
OnCalendar=*-*-* 19:30:00
Persistent=true

[Install]
WantedBy=timers.target
• OnCalendar=*-*-* 19:30:00: formato año-mes-día hora. Los asteriscos significan «cualquiera», así que se ejecuta todos los días a las 19:30.
• Persistent=true: si el equipo estaba apagado a esa hora, la copia se hace al encenderlo.
• WantedBy=timers.target: hace que el timer se active al arrancar el sistema.
• El timer lanza automáticamente el service que tiene el mismo nombre (backup-rsync).
Para comprobar que una hora está bien escrita:
systemd-analyze calendar '*-*-* 19:30:00'
Paso 20. Crear el service y el timer de tar.gz
Son iguales, cambiando el nombre y el script. Crea /etc/systemd/system/backup-tar.service:
[Unit]
Description=Copia de seguridad con tar.gz al equipo remoto
Wants=network-online.target
After=network-online.target

[Service]
Type=oneshot
User=alumno
ExecStart=/usr/local/bin/backup_tar.sh
Y /etc/systemd/system/backup-tar.timer (a las 20:00, para que no coincida con el de rsync):
[Unit]
Description=Programa la copia con tar.gz todos los días a las 20:00

[Timer]
OnCalendar=*-*-* 20:00:00
Persistent=true

[Install]
WantedBy=timers.target
Paso 21. Activar los timers
sudo systemctl daemon-reload
sudo systemctl enable --now backup-rsync.timer
sudo systemctl enable --now backup-tar.timer
• daemon-reload: obliga a systemd a leer los ficheros nuevos. Hay que hacerlo cada vez que se crea o cambia una unidad.
• enable: activa el timer para que arranque siempre con el sistema.
• --now: además, lo arranca ahora mismo.
• Se activa el .timer, no el .service.
Paso 22. Comprobar que funciona
systemctl list-timers
sudo systemctl start backup-rsync.service
systemctl status backup-rsync.service
journalctl -u backup-rsync.service
• list-timers: muestra todos los timers, cuándo se ejecutarán la próxima vez y cuándo lo hicieron por última vez.
• start: lanza la copia ahora, sin esperar al horario. Así puedes probarla.
• status: muestra si la última ejecución terminó bien. Con oneshot, lo normal es ver inactive (dead) y un status=0/SUCCESS.
• journalctl -u: muestra los mensajes que escribió el script (echo) y los errores. Esta es una gran ventaja de systemd: los registros se guardan solos.
Para ver una ejecución automática sin esperar a las 19:30, cambia temporalmente en el timer OnCalendar=*:0/2 (cada 2 minutos), ejecuta sudo systemctl daemon-reload y sudo systemctl restart backup-rsync.timer, espera y mira journalctl. Después, devuelve la hora original.
Paso 23. Probar la restauración final
Borra algo de origen y recúpralo desde la copia remota:
rm ~/origen/file7
rsync -avzh punky@192.168.1.78:/home/punky/backups/rsync/ ~/origen/
ls ~/origen
Para tar.gz, descarga el archivo y extráelo como en el Paso 14, usando --listed-incremental=/dev/null y empezando siempre por la copia completa.
Errores frecuentes y soluciones
Problema
Causa probable
Solución
Permission denied (publickey)
La clave no se copió bien a MV2
Repetir ssh-copy-id
El script falla con systemd pero va bien a mano
User= distinto del usuario que tiene la clave SSH
Poner el usuario correcto en el .service
Host key verification failed
Nunca te conectaste a MV2 con ese usuario
Hacer una vez ssh punky@IP y escribir yes
Permission denied al ejecutar el script
Falta el permiso de ejecución
sudo chmod +x /usr/local/bin/backup_*.sh
El timer no aparece en list-timers
No se hizo daemon-reload o no se activó
sudo systemctl daemon-reload y enable --now
Connection timed out
MV2 apagada o mala configuración de red
Comprobar con ping y revisar la red de VirtualBox
Conclusión: crontab frente a systemd
Systemd, con sus timers, es una opción más robusta y fácil de supervisar que crontab para tareas importantes como una copia de seguridad. Guarda los registros de errores automáticamente, permite esperar a que haya red, y recupera las ejecuciones perdidas con Persistent=true.
Sin embargo, crontab sigue siendo más rápido y simple para tareas sencillas: una sola línea basta. Además, sigue muy extendido y funciona en sistemas que no usan systemd.
Para una copia de seguridad remota con rsync o tar.gz, mi elección es systemd, porque la copia depende de la red y es importante saber si ha fallado. Para una tarea sencilla y local, crontab es suficiente.
Parte 3. La práctica en la máquina virtual, con lo que verás en pantalla
Esta parte repite la práctica tal y como se hace en VirtualBox: primero se preparan las máquinas, y después se muestra cada orden junto con lo que aparece en el terminal. Así puedes comprobar si tu pantalla se parece a la de los ejemplos.
Cómo leer los ejemplos
• Las líneas que empiezan por alumno@mv1:~$ son órdenes que escribes tú en la máquina MV1. Lo que va después del $ es lo que debes escribir.
• Las líneas que empiezan por punky@mv2:~$ son órdenes en la máquina MV2.
• Las líneas sin ese principio son lo que el programa responde.
• Los números (tamaños, fechas, velocidades) serán distintos en tu pantalla. Es normal.
• Truco para no perderte: mira siempre el principio de la línea (el prompt). Si pone mv1 estás en la máquina de origen, y si pone mv2 estás en la de destino.
A. Preparar las máquinas en VirtualBox
A1. Crear las dos máquinas
1. Abre VirtualBox y pulsa Nueva.
2. Pon el nombre mv1, elige la imagen ISO de Ubuntu (o Debian) y pulsa Siguiente.
3. Asigna 2 GB de memoria RAM y 20 GB de disco (valores suficientes para la práctica).
4. Inicia la máquina e instala el sistema. Crea el usuario alumno.
5. Repite todo para crear mv2, con el usuario punky.
Si prefieres ahorrar tiempo, puedes clonar mv1 para crear mv2. En ese caso, al clonar marca la opción Reinicializar la dirección MAC de todas las tarjetas de red. Si no lo haces, las dos máquinas tendrían la misma dirección MAC y fallaría la red.
A2. Conectar las máquinas en la misma red
Con cada máquina apagada:
1. Selecciónala y pulsa Configuración, después Red.
2. En Adaptador 1, marca Habilitar adaptador de red.
3. En Conectado a, elige Adaptador puente y selecciona tu tarjeta de red real (por ejemplo la wifi o la ethernet).
4. Pulsa Aceptar.
Con el adaptador puente, cada máquina virtual recibe una IP de tu router, como si fuera un ordenador más de la casa. Por eso las dos se ven entre sí.
Si no puedes usar el adaptador puente, otra opción es poner las dos en el mismo Red NAT (se crea en Archivo > Herramientas > Administrador de red > Redes NAT).
A3. Poner nombre a cada máquina
Si las has creado por separado, ya tienen nombre. Si has clonado, cambia el nombre de mv2:
punky@ubuntu:~$ sudo hostnamectl set-hostname mv2
hostnamectl set-hostname cambia el nombre de la máquina. Cierra el terminal y ábrelo de nuevo: el prompt pasará a ser punky@mv2:~$.
A4. Averiguar la IP de MV2
En MV2 escribe:
punky@mv2:~$ ip -4 a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    inet 192.168.1.78/24 brd 192.168.1.255 scope global dynamic enp0s3
       valid_lft 86213sec preferred_lft 86213sec
• ip -4 a muestra las tarjetas de red y sus direcciones IPv4 (-4).
• La tarjeta lo es la del propio equipo (127.0.0.1). Ignórala.
• La tarjeta enp0s3 es la real. Su IP es la que aparece después de inet, antes de la barra: 192.168.1.78. El /24 es la máscara de red y no forma parte de la IP.
• Apunta esta IP. Es la que usaremos en todas las órdenes, en lugar de 192.168.1.78 si la tuya es otra.
A5. Instalar los programas
En MV1 y en MV2:
alumno@mv1:~$ sudo apt update
[sudo] password for alumno:
Hit:1 http://archive.ubuntu.com/ubuntu noble InRelease
...
Reading package lists... Done
alumno@mv1:~$ sudo apt install rsync openssh-server
...
rsync is already the newest version (3.2.7-1).
...
• La primera vez que usas sudo pide tu contraseña. Al escribirla no aparece nada en pantalla (ni asteriscos). Es normal: escríbela y pulsa Enter.
• Si dice rsync is already the newest version, ya estaba instalado y no pasa nada.
• Cuando pregunte Do you want to continue? [Y/n], escribe Y y pulsa Enter.
A6. Comprobar que las máquinas se ven
Desde MV1:
alumno@mv1:~$ ping -c 3 192.168.1.78
PING 192.168.1.78 (192.168.1.78) 56(84) bytes of data.
64 bytes from 192.168.1.78: icmp_seq=1 ttl=64 time=0.612 ms
64 bytes from 192.168.1.78: icmp_seq=2 ttl=64 time=0.498 ms
64 bytes from 192.168.1.78: icmp_seq=3 ttl=64 time=0.540 ms

--- 192.168.1.78 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2038ms
• Todo correcto: 3 received, 0% packet loss. Hay conexión.
• Algo va mal: 100% packet loss o Destination Host Unreachable. Revisa que las dos máquinas estén encendidas, con el adaptador puente y en la misma red.
B. Práctica 1 en MV1: copias en local
B1. Crear carpetas y ficheros
alumno@mv1:~$ mkdir origen destino
alumno@mv1:~$ touch origen/file{1..10}
alumno@mv1:~$ echo 'texto de prueba' > origen/file1
alumno@mv1:~$ ls origen
file1  file10  file2  file3  file4  file5  file6  file7  file8  file9
Las órdenes que crean cosas (mkdir, touch, echo) no muestran nada cuando salen bien. Que no aparezca texto significa que todo ha ido bien. Solo ls muestra la lista.
Observa que file10 aparece antes que file2. Linux ordena por letras, no por números.
B2. Primera copia con rsync (completa)
alumno@mv1:~$ rsync -avzh origen/ destino/
sending incremental file list
./
file1
file10
file2
file3
file4
file5
file6
file7
file8
file9

sent 806 bytes  received 252 bytes  2.12K bytes/sec
total size is 16  speedup is 0.02
Cómo leer esta salida:
• sending incremental file list: rsync empieza a construir la lista de ficheros. Es su forma de trabajar y no tiene que ver con las copias incrementales de las que hablamos en la teoría.
• ./: la propia carpeta origen, que también se copia.
• Cada nombre (file1, file10...) es un fichero que se ha copiado en esta ejecución.
• sent ... received ...: los datos enviados y recibidos.
• total size is 16: el tamaño total de los ficheros (file1 tiene 16 caracteres y el resto están vacíos).
Comprueba el resultado:
alumno@mv1:~$ ls -l destino
total 4
-rw-rw-r-- 1 alumno alumno 16 oct 20 10:12 file1
-rw-rw-r-- 1 alumno alumno  0 oct 20 10:12 file10
-rw-rw-r-- 1 alumno alumno  0 oct 20 10:12 file2
...
Las columnas son: permisos, número de enlaces, propietario, grupo, tamaño en bytes, fecha y nombre.
B3. Segunda copia sin cambios, y tercera con cambios
alumno@mv1:~$ rsync -avzh origen/ destino/
sending incremental file list

sent 267 bytes  received 12 bytes  558.00 bytes/sec
total size is 16  speedup is 0.06
No aparece ningún fichero: rsync ha comprobado que todo es igual y no ha copiado nada. Ahora cambiamos cosas:
alumno@mv1:~$ echo 'cambio nuevo' >> origen/file1
alumno@mv1:~$ touch origen/file11
alumno@mv1:~$ rsync -avzh origen/ destino/
sending incremental file list
./
file1
file11

sent 394 bytes  received 54 bytes  896.00 bytes/sec
total size is 29  speedup is 0.06
Solo aparecen file1 (modificado) y file11 (nuevo). Esto demuestra que rsync copia solo lo que ha cambiado. Esta captura es una buena prueba para el informe.
B4. Simular un borrado (con --delete y -n)
alumno@mv1:~$ rm origen/file3
alumno@mv1:~$ rsync -avzh --delete -n origen/ destino/
sending incremental file list
./
deleting file3

sent 215 bytes  received 22 bytes  474.00 bytes/sec
total size is 29  speedup is 0.06 (DRY RUN)
• -n hace una simulación: muestra lo que haría, pero no lo hace. Fijate en (DRY RUN) al final.
• deleting file3 indica que, sin el -n, rsync borraría file3 del destino porque ya no existe en el origen.
• Si ejecutas la orden sin -n, el fichero se borra de verdad de destino.
B5. Restaurar con rsync
alumno@mv1:~$ rm origen/file5 origen/file6
alumno@mv1:~$ ls origen
file1  file10  file11  file2  file4  file7  file8  file9
alumno@mv1:~$ rsync -avzh destino/ origen/
sending incremental file list
./
file5
file6

sent 222 bytes  received 54 bytes  552.00 bytes/sec
total size is 29  speedup is 0.11
Se invierte el orden (destino/ primero, origen/ después) y los dos ficheros borrados vuelven a aparecer. Compruébalo con ls origen.
B6. Copia y restauración con tar.gz
alumno@mv1:~$ tar -czvf destino/copia_completa.tar.gz origen/
origen/
origen/file1
origen/file10
...
alumno@mv1:~$ tar -tzvf destino/copia_completa.tar.gz
drwxrwxr-x alumno/alumno     0 2026-10-20 10:20 origen/
-rw-rw-r-- alumno/alumno    29 2026-10-20 10:18 origen/file1
-rw-rw-r-- alumno/alumno     0 2026-10-20 10:12 origen/file10
...
• -v hace que tar muestre cada fichero mientras lo guarda.
• -t lista el contenido del archivo sin extraerlo. Es la forma de comprobar que la copia está bien hecha.
• En el listado, la d inicial indica una carpeta y el guion -, un fichero.
Para las copias incremental y diferencial, repite los Pasos 8 y 9 de la Parte 2 y haz siempre un tar -tzvf al final: en la incremental, el listado debe mostrar solo los ficheros cambiados.
Para restaurar y comprobar:
alumno@mv1:~$ mkdir restauracion
alumno@mv1:~$ tar --listed-incremental=/dev/null -xzvf destino/copia_completa.tar.gz -C restauracion/
origen/
origen/file1
...
alumno@mv1:~$ diff -r origen restauracion/origen
alumno@mv1:~$
diff -r no muestra nada cuando las dos carpetas son iguales. Si muestra líneas, hay diferencias. Si ves un aviso tar: origen: Directory is new mientras creas la primera copia incremental, es normal: tar indica que no tenía memoria previa de esa carpeta.
C. Práctica 2 desde MV1 a MV2: copias por la red
C1. Primera conexión SSH
alumno@mv1:~$ ssh punky@192.168.1.78
The authenticity of host '192.168.1.78 (192.168.1.78)' can't be established.
ED25519 key fingerprint is SHA256:Qm3k...x7Zc.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.1.78' (ED25519) to the list of known hosts.
punky@192.168.1.78's password:
Welcome to Ubuntu 24.04 LTS ...
punky@mv2:~$
• La pregunta aparece solo la primera vez: SSH te dice que no conoce esa máquina. Escribe yes completo (no solo y).
• La contraseña que escribes es la de punky, y tampoco se ve.
• Fíjate en el prompt: ha cambiado de alumno@mv1 a punky@mv2. Ahora lo que escribes se ejecuta en MV2.
• Para volver a MV1 escribe exit:
punky@mv2:~$ exit
logout
Connection to 192.168.1.78 closed.
alumno@mv1:~$
C2. Copia por red con rsync
alumno@mv1:~$ ssh punky@192.168.1.78 'mkdir -p ~/backups'
punky@192.168.1.78's password:
alumno@mv1:~$ rsync -avzh origen/ punky@192.168.1.78:backups/
punky@192.168.1.78's password:
sending incremental file list
./
file1
file10
...

sent 1.02K bytes  received 285 bytes  523.20 bytes/sec
total size is 29  speedup is 0.02
Pide la contraseña en cada orden porque todavía no hemos configurado las claves SSH (se hace en C4). Después, comprueba en MV2:
punky@mv2:~$ ls ~/backups
file1  file10  file11  file2  file4  file5  file6  file7  file8  file9
Si ves los ficheros en MV2, la copia por la red ha funcionado.
C3. Copia por red con tar.gz
alumno@mv1:~$ tar -czvf copia_red.tar.gz origen/
...
alumno@mv1:~$ scp copia_red.tar.gz punky@192.168.1.78:backups/
punky@192.168.1.78's password:
copia_red.tar.gz                              100%  187    94.2KB/s   00:00
scp muestra una barra con el nombre del fichero, el porcentaje (100% = enviado entero), el tamaño y la velocidad.
Para restaurar, haz el camino contrario:
alumno@mv1:~$ mkdir restauracion_red
alumno@mv1:~$ rsync -avzh punky@192.168.1.78:backups/ restauracion_red/
C4. Configurar las claves SSH (para que no pida contraseña)
alumno@mv1:~$ ssh-keygen -t rsa -b 4096
Generating public/private rsa key pair.
Enter file in which to save the key (/home/alumno/.ssh/id_rsa):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /home/alumno/.ssh/id_rsa
Your public key has been saved in /home/alumno/.ssh/id_rsa.pub
The key fingerprint is:
SHA256:... alumno@mv1
• A cada pregunta, pulsa Enter sin escribir nada. Así se usa la ruta por defecto y la clave no lleva contraseña.
• Se crean dos ficheros: id_rsa (clave privada, no se comparte nunca) e id_rsa.pub (clave pública, esta es la que se envía).
alumno@mv1:~$ ssh-copy-id punky@192.168.1.78
...
punky@192.168.1.78's password:

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'punky@192.168.1.78'"
and check to make sure that only the key(s) you wanted were added.
Esta es la última vez que escribes la contraseña de punky. Ahora la prueba definitiva:
alumno@mv1:~$ ssh punky@192.168.1.78
Welcome to Ubuntu 24.04 LTS ...
punky@mv2:~$ exit
Si entra sin pedir contraseña, está bien hecho. Si la pide, repite ssh-copy-id antes de seguir, porque los scripts automáticos fallarían.
D. Prácticas 4 y 5 en MV1: scripts y systemd
D1. Crear el script con nano
alumno@mv1:~$ sudo nano /usr/local/bin/backup_rsync.sh
Se abre una pantalla casi vacía. Es el editor nano. En la parte de abajo ves unos atajos, como ^O Write Out y ^X Exit. El símbolo ^ significa la tecla Ctrl.
1. Pega el script de la Parte 2 (Paso 15). En el terminal de VirtualBox se pega con Ctrl+Shift+V. Si no funciona, activa Dispositivos > Portapapeles compartido > Bidireccional en el menú de la máquina virtual, o escribe el script a mano.
2. Para guardar, pulsa Ctrl+O. Abajo aparece File Name to Write: /usr/local/bin/backup_rsync.sh. Pulsa Enter para confirmar.
3. Para salir, pulsa Ctrl+X.
Haz lo mismo con backup_tar.sh. Después da permisos de ejecución y comprueba:
alumno@mv1:~$ sudo chmod +x /usr/local/bin/backup_rsync.sh /usr/local/bin/backup_tar.sh
alumno@mv1:~$ ls -l /usr/local/bin/backup_*.sh
-rwxr-xr-x 1 root root 1143 oct 20 19:05 /usr/local/bin/backup_rsync.sh
-rwxr-xr-x 1 root root 1796 oct 20 19:08 /usr/local/bin/backup_tar.sh
La x en -rwxr-xr-x indica que el fichero se puede ejecutar. Si no aparece, falta el chmod +x.
D2. Ejecutar los scripts a mano
alumno@mv1:~$ /usr/local/bin/backup_rsync.sh
[2026-10-20 19:12:03] Empezando la copia con rsync...
sending incremental file list
./
file1
...

sent 1.02K bytes  received 285 bytes  2.61K bytes/sec
total size is 29  speedup is 0.02
[2026-10-20 19:12:05] Copia con rsync terminada correctamente.
Si sale bien, aparecen las dos líneas con la fecha (Empezando... y terminada correctamente) y entre ellas la salida de rsync. Ahora el de tar.gz:
alumno@mv1:~$ /usr/local/bin/backup_tar.sh
[2026-10-20 19:14:40] Empezando la copia con tar.gz...
sending incremental file list
backup_completa_2026-10-20.tar.gz

sent 389 bytes  received 35 bytes  848.00 bytes/sec
total size is 275  speedup is 0.65
[2026-10-20 19:14:41] Copia completa con tar.gz terminada correctamente.
La primera vez es siempre una copia completa, porque todavía no existe el fichero .snar. Al día siguiente (si no es domingo) dirá Copia incremental. Comprueba en MV2:
punky@mv2:~$ ls -l ~/backups/tar
-rw-rw-r-- 1 punky punky 275 oct 20 19:14 backup_completa_2026-10-20.tar.gz
No sigas hasta que los dos scripts funcionen a mano. Si fallan, mira la tabla de errores de la Parte 2.
D3. Crear las unidades de systemd
Se crean con nano, igual que los scripts, pero con sudo porque están en una carpeta del sistema:
alumno@mv1:~$ sudo nano /etc/systemd/system/backup-rsync.service
alumno@mv1:~$ sudo nano /etc/systemd/system/backup-rsync.timer
alumno@mv1:~$ sudo nano /etc/systemd/system/backup-tar.service
alumno@mv1:~$ sudo nano /etc/systemd/system/backup-tar.timer
Pega en cada uno el contenido de los Pasos 18, 19 y 20 de la Parte 2, y no olvides cambiar User=alumno si tu usuario se llama de otra forma. Comprueba que existen los cuatro:
alumno@mv1:~$ ls /etc/systemd/system/backup-*
/etc/systemd/system/backup-rsync.service  /etc/systemd/system/backup-tar.service
/etc/systemd/system/backup-rsync.timer    /etc/systemd/system/backup-tar.timer
D4. Activar los timers
alumno@mv1:~$ sudo systemctl daemon-reload
alumno@mv1:~$ sudo systemctl enable --now backup-rsync.timer
Created symlink /etc/systemd/system/timers.target.wants/backup-rsync.timer → /etc/systemd/system/backup-rsync.timer.
alumno@mv1:~$ sudo systemctl enable --now backup-tar.timer
Created symlink /etc/systemd/system/timers.target.wants/backup-tar.timer → /etc/systemd/system/backup-tar.timer.
• daemon-reload no muestra nada si todo va bien.
• Los mensajes Created symlink confirman que el timer queda activado para cada arranque. Un symlink es un acceso directo: systemd crea uno para saber qué debe arrancar.
• Si aparece un error como Failed to enable unit: Unit file ... does not exist, el nombre del fichero está mal escrito.
D5. Ver los timers programados
alumno@mv1:~$ systemctl list-timers --all | grep backup
Tue 2026-10-20 19:30:00 CEST  12min left  n/a  n/a  backup-rsync.timer  backup-rsync.service
Tue 2026-10-20 20:00:00 CEST  42min left  n/a  n/a  backup-tar.timer    backup-tar.service
• | grep backup filtra la lista para ver solo nuestros timers (la barra | pasa el resultado a grep, que muestra solo las líneas con esa palabra).
• La primera columna es cuándo se ejecutará y la segunda cuánto falta. Las columnas con n/a (última ejecución) se rellenarán cuando se haya ejecutado una vez.
• La última pareja de columnas indica qué timer lanza qué service.
D6. Lanzar la copia con systemd y ver el resultado
alumno@mv1:~$ sudo systemctl start backup-rsync.service
alumno@mv1:~$ systemctl status backup-rsync.service
○ backup-rsync.service - Copia de seguridad con rsync al equipo remoto
     Loaded: loaded (/etc/systemd/system/backup-rsync.service; static)
     Active: inactive (dead) since Tue 2026-10-20 19:20:11 CEST; 8s ago
    Process: 2105 ExecStart=/usr/local/bin/backup_rsync.sh (code=exited, status=0/SUCCESS)
   Main PID: 2105 (code=exited, status=0/SUCCESS)
Cómo interpretarlo:
• Active: inactive (dead) no es un error. Como el service es oneshot, se ejecuta, termina y queda parado. Es lo normal.
• status=0/SUCCESS significa que el script terminó bien (el código 0 es éxito).
• Para salir de esta pantalla, pulsa q.
Ahora mira el registro (el log) de esa ejecución:
alumno@mv1:~$ journalctl -u backup-rsync.service
oct 20 19:20:09 mv1 systemd[1]: Starting backup-rsync.service - Copia de seguridad con rsync al equipo remoto...
oct 20 19:20:09 mv1 backup_rsync.sh[2105]: [2026-10-20 19:20:09] Empezando la copia con rsync...
oct 20 19:20:10 mv1 backup_rsync.sh[2105]: sending incremental file list
oct 20 19:20:11 mv1 backup_rsync.sh[2105]: [2026-10-20 19:20:11] Copia con rsync terminada correctamente.
oct 20 19:20:11 mv1 systemd[1]: backup-rsync.service: Deactivated successfully.
oct 20 19:20:11 mv1 systemd[1]: Finished backup-rsync.service - Copia de seguridad con rsync al equipo remoto.
Todo lo que el script escribe con echo queda guardado aquí, con la fecha y el nombre del programa. Con crontab esto no ocurre solo. También puedes ver el registro en directo con journalctl -u backup-rsync.service -f (la opción -f significa follow, seguir). Para parar, pulsa Ctrl+C.
D7. Probar un fallo (demuestra la ventaja de systemd)
1. Apaga MV2 desde VirtualBox.
2. En MV1 lanza la copia:
alumno@mv1:~$ sudo systemctl start backup-rsync.service
Job for backup-rsync.service failed because the control process exited with error code.
See "systemctl status backup-rsync.service" and "journalctl -xeu backup-rsync.service" for details.
alumno@mv1:~$ systemctl status backup-rsync.service
× backup-rsync.service - Copia de seguridad con rsync al equipo remoto
     Loaded: loaded (/etc/systemd/system/backup-rsync.service; static)
     Active: failed (Result: exit-code) since Tue 2026-10-20 19:40:12 CEST; 5s ago
    Process: 2210 ExecStart=/usr/local/bin/backup_rsync.sh (code=exited, status=255/EXCEPTION)
alumno@mv1:~$ journalctl -u backup-rsync.service -n 5
...
oct 20 19:40:12 mv1 backup_rsync.sh[2210]: ssh: connect to host 192.168.1.78 port 22: No route to host
oct 20 19:40:12 mv1 systemd[1]: backup-rsync.service: Failed with result 'exit-code'.
• systemd detecta el fallo y lo marca en rojo como failed. La línea status=255 sale porque SSH no pudo conectar.
• journalctl ... -n 5 muestra solo las últimas 5 líneas del registro, y ahí aparece el motivo real del error. Según el estado de tu red, el mensaje puede ser Connection timed out en lugar de No route to host.
• Gracias a set -euo pipefail y a ssh -o BatchMode=yes -o ConnectTimeout=10, el script no se queda colgado: falla rápido y deja un aviso claro.
3. Enciende MV2, espera un minuto y repite el start. Ahora debe terminar con status=0/SUCCESS.
D8. Ver una ejecución automática sin esperar a las 19:30
Para no esperar horas, cambia el timer temporalmente:
alumno@mv1:~$ sudo nano /etc/systemd/system/backup-rsync.timer
Cambia la línea OnCalendar=*-*-* 19:30:00 por OnCalendar=*:0/2 (significa «cada 2 minutos»). Guarda con Ctrl+O, Enter y sal con Ctrl+X. Después:
alumno@mv1:~$ sudo systemctl daemon-reload
alumno@mv1:~$ sudo systemctl restart backup-rsync.timer
alumno@mv1:~$ journalctl -u backup-rsync.service -f
En dos minutos verás aparecer sola una nueva ejecución en el registro, sin que hayas ejecutado nada. Es la prueba de que el timer funciona. Cuando termines, vuelve a poner OnCalendar=*-*-* 19:30:00 y repite daemon-reload y restart, para que la copia no se ejecute cada 2 minutos.
D9. Restauración final
alumno@mv1:~$ rm ~/origen/file7
alumno@mv1:~$ rsync -avzh punky@192.168.1.78:backups/rsync/ ~/origen/
sending incremental file list
./
file7

sent 38 bytes  received 148 bytes  372.00 bytes/sec
total size is 29  speedup is 0.16
alumno@mv1:~$ ls ~/origen
file1  file10  file11  file2  file4  file5  file6  file7  file8  file9
El fichero borrado vuelve a aparecer. Ten en cuenta que en los scripts el destino es /home/punky/backups/rsync, y por eso aquí la ruta remota es backups/rsync/.
E. Capturas de pantalla para el informe
Haz una captura de cada una de estas pruebas. Sirven como evidencia de que has hecho la práctica.
N.º
Qué captura
Orden o lugar
1
Las dos máquinas con su IP
ip -4 a en MV1 y en MV2
2
Conexión entre ellas
ping -c 3 192.168.1.78
3
Copia completa con rsync
rsync -avzh origen/ destino/
4
Copia incremental con rsync (solo cambios)
Segunda ejecución tras echo y touch
5
Contenido de un .tar.gz
tar -tzvf ...
6
Incremental y diferencial con tar
Los dos listados tar -tzvf
7
Restauración correcta
diff -r sin salida
8
Copia por red
rsync a MV2 y ls ~/backups en MV2
9
Conexión por claves SSH
ssh punky@IP sin contraseña
10
Los scripts
ls -l /usr/local/bin/backup_*.sh y su contenido con cat
11
Los timers activos
systemctl list-timers --all filtrado
12
Ejecución con éxito
systemctl status con status=0/SUCCESS
13
Registro en el journal
journalctl -u backup-rsync.service
14
Prueba de fallo
status en failed y su journalctl
15
Restauración desde la máquina remota
El rsync de vuelta y ls
