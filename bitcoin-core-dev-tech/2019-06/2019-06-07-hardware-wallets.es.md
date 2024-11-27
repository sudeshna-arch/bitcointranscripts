---
title: Hardware Wallets
transcript_by: Bryan Bishop
translation_by: Blue Moon
tags: ['hardware-wallet', 'hwi']
speakers: ['Sjors Provoost', 'Jonas Schnelli', 'Andrew Chow']
date: 2019-06-07
---

<https://twitter.com/kanzure/status/1136924010955104257>

¿Cuánto debería hacer Bitcoin Core y cuánto otras bibliotecas? Andrew Chow escribió la maravillosa herramienta HWI. Ahora mismo tenemos un pull request para soportar firmantes externos. El script HWI puede hablar con la mayoría de los monederos hardware porque tiene todos los controladores incorporados, y puede obtener claves de ellos, y firmar transacciones arbitrarias. Eso es más o menos lo que hace. Es un poco manual, sin embargo. Tienes que introducir algunos comandos python; llamar a algún RPC de Bitcoin Core para obtener el resultado; así que escribí algunos métodos RPC de conveniencia para Bitcoin Core que te permiten hacer las mismas cosas con menos comandos. Así que ahora puedes hacer enumeratesigners y bitcoin-cli fetchkeys y hará todo el baile para obtener todas las claves del dispositivo y ponerlas en la cartera. Las claves públicas, no las privadas. Sí, las claves privadas serían bitcoin-cli footgun. Luego hay un comando para mostrar la dirección. Cualquier dirección que quieras, entonces ese método RPC buscaría en la billetera, encontraría la dirección real, sacaría las claves, le diría al firmante, obtendría la ruta de derivación para que no necesites encontrar manualmente la ruta de derivación. También está signtransaction que hace toda la magia de componer un - es esencialmente lo mismo que sendmany o createpsbt. Le das un montón de direcciones de destino; creará un PSBT, lo enviará al dispositivo, esperará a que el dispositivo lo firme y luego lo difundirá.

Idealmente no deberíamos tener tantos comandos, todo debería ocurrir internamente. Imagina que tuvieras una cartera de descriptores perfecta. ¿Cuál sería el delta? El método de conveniencia se podría quitar del pull request y moverlo aparte. El que está ahí en un nivel inferior es processpsbt. Así que hay un signerprocesspsbt que hace lo que hace processpsbt. Lo único que hace el signerprocesspsbt es eso y algunas cosas más. Podría quitar eso del pull request.

Relacionado con lo de la re-arquitectura de la billetera, suena como si tuvieras atascadas las billeteras de hardware. Bueno, es agnóstico al firmante. Podrías usarlo con monederos de hardware, pero más adelante quizá también con multisig. Vamos a tener un descriptor nativo scriptpubkey manager, y luego un scriptpubkey manager externo. El gestor es responsable de la firma. Será responsable de la firma.

Sigo pensando que la API no es ideal para su adopción masiva. Una vez propuse, no como BIP, sino en algún foro, sobre un esquema de url para la comunicación entre watchonly y dispositivos firmantes. Creo que el modelo actual no funcionará con los monederos móviles, justo porque el script HWI no es algo que se pueda transportar; así que usarían los comandos RPC supongo. La GUI necesitaría una forma de mostrar las direcciones de un dispositivo, y también una forma de firmar cosas. Tiene que haber una clase para comunicarse entre el firmante y la GUI.

¿Por qué no trabajar en una API que sea completamente agnóstica del vendedor? Existe un monedero software, Bitcoin Core, y un monedero hardware como los tres que representa HWI. ¿No podría ser totalmente flexible? El esquema URL puede ser ideal porque Bitcoin Core podría llamar a https://sign/ o algo así. Esto requiere un servidor en funcionamiento. Una idea era tener json-rpc. Algo necesita escuchar ese esquema url. Así que esto podría funcionar en ambas direcciones. ¿Esto no requiere que el usuario inicie algunas acciones? No puedes imponer qué aplicaciones manejan qué URIs. Si instalas un trozo de malware, puedes redirigir URIs y no... bueno, tener malware en la máquina es bastante arbitrario. Las GUIs lanzan una aplicación específica; ¿hay un esquema URI donde el tipo correcto de aplicación pueda responder a eso? Una cartera arbitraria puede escuchar el esquema URI.

Tener un script python separado es desde una perspectiva de revisión de código algo más fácil, porque podemos mantenerlo separado del proyecto principal al menos durante un tiempo. De lo contrario, tendría que incluir los controladores USB. El formato general con el que nos comunicamos, hay un mensaje signtransaction que toma algunos argumentos, y podemos estandarizar el protocolo pero no el método de conexión por ahora. En este momento es sólo la ejecución de otro comando con un montón de argumentos. Yo quería un esquema que funcione para todo el ecosistema. Necesitamos hacer un modelo donde todos los monederos sepan genéricamente cómo hablar con todos los esquemas de hardware. Los esquemas URI podrían funcionar para esto. Con HWI, son los mismos comandos para todos los dispositivos excepto Trezor. Más o menos. Así que todos los mismos comandos están disponibles. Siempre que hagas algo, independientemente del dispositivo, podrías usar aproximadamente los mismos comandos cada vez. La única cosa con HWI que es super bitcoin específica es lo que sea el getkeypool porque eso utiliza el comando importmulti que es más fácil. He añadido un pull request a HWI para obtener descriptores que obtiene descriptores en una matriz que es super universal.

Cuando miro en la cosa URI, me encontré con la única manera de dos aplicaciones para comunicarse que cubre todos los dispositivos principales, es el esquema de URI. Usted podría agregar a HWI o alguna otra herramienta, algún controlador de URI que mapea el URI a los comandos y luego mantiene una nota acerca de cómo responder a donde vino. Cada comando que llames en HWI podría tener un disparador de devolución de llamada opcional.

Si tienes varias billeteras de hardware, porque estás usando multisig, y quieres elegir una billetera de hardware en particular. Sería solucionable con algunos pasos adicionales. Usted pasa una huella digital a HWI del dispositivo que desea. Bitcoin Core conoce los diferentes monederos por hardware porque cuando importas... El monedero de Bitcoin Core tiene entradas de huellas digitales maestras, y entonces puedes consultar a HWI por la huella digital maestra. Dice «utilizar un dispositivo para esta huella digital» y luego lo utilizará.

Es muy importante tener un protocolo que pueda ampliarse para realizar otras funciones que los monederos por hardware no hacen actualmente, como las reglas de consenso. También debe ser un protocolo universal transportable que funcione para todos los monederos de software y todos los monederos de hardware. Un proyecto más grande sería ampliar libwally para tener todos los controladores USB y todas las cosas en él; se toma todo el material de Python en HWI y lo convertimos en C. Digamos que soy un monedero sólo de reloj, puedo hacer un PSBT, tengo una dirección que quiero verificar - muy alto nivel en la capa de transporte, que es transportable en todos los dispositivos. La mensajería llega a través de un comando, pero la mensajería podría llegar a través de json-rpc o un esquema URI. Deberíamos hacer este esquema URI ahora, no más tarde. Antes de que esté grabado en piedra, deberíamos hacer algo que sea más independiente de la plataforma.

Hay una buena separación entre recibir los comandos desde la línea de comandos y realmente ejecutarlos con todos los argumentos. Es fácilmente portable a un wrapper para simplemente poner un wrapper alrededor de los comandos. Alguien podría hacer un pull request y añadir un servidor json-rpc, o un manejador URI. O simplemente poner la capa de transporte del contenido que estás moviendo. Alguien agregó cli.py que toma la entrada de la línea de comandos, y luego llama a funciones que son agnósticas al hecho de que fueron llamadas desde la línea de comandos, y esas funciones devuelven un objeto json. Creo que eso ya está ahí. Me gustaría ver un servidor json-rpc, pero el problema con él es que cualquier cosa en el sistema puede llamarlo, en particular ... sí, usted podría hacer algo de autenticación. Puedes usar una cookie. Esto parece prevenible.

Siempre estoy preocupado por el proceso de inicialización. Digamos que si hablas con extensiones de monedero de hardware UI, llegas más a la gente normal que no está acostumbrada a usar PSBT. ¿Qué pasa con la inicialización del hardware? La configuración del dispositivo, sí. Usando Trezor, necesitas usar Chrome para inicializar Trezor para usarlo con Bitcoin Core. ¿Sólo se puede utilizar Bitcoin Core? Trezor sí, Ledger no. La configuración de Ledger no está relacionada con tu ordenador, lo cual está bien. El problema con la configuración es que para algunos dispositivos hay interacción para la configuración. Es completamente interactivo, lo que significa que si quieres integrarlo en Bitcoin Core tendrías que tener algo que gestione la interacción, algo específico del dispositivo. Con otros dispositivos, como Ledgers, la configuración es completamente independiente de tu ordenador. Lo conectas a la corriente, no necesariamente a un puerto USB, y puedes configurar el dispositivo. Esta diferencia hace que la configuración sea bastante molesta. Cuando descargas Bitcoin Core, lo inicias, y te pregunta si quieres hacer un nuevo monedero, y entonces te pregunta si quieres usar un monedero hardware, y entonces mágicamente enumera todos los monederos hardware, seleccionas el monedero que quieres, el boom de las claves está ahí, y cuando firma una transacción se da cuenta de que necesita preguntar por un dispositivo externo y entonces te pregunta. Dado un dispositivo ya inicializado, la configuración desde Bitcoin Core para registrar el dispositivo es bastante fácil. Si podemos hacer la configuración algo genérico, pero sí cada dispositivo tiene su propio proceso.

Mi tercera preocupación, lo siento por traer sólo preocupaciones y no respuestas, todo el modelo de plugin. Vi que con electrum es una sola base de código mantenido por ThomasV así que cada vez que hay un error en uno de esos plugins, como Trezor encontró un error en su propio plugin dentro de electrum, no tienen control para arreglarlo. Es una capa eliminada de responsabilidad. No tienen control sobre el ciclo de lanzamiento. Si es crucial quizás llamen a ThomasV pero no es fiable. El controlador debería ser responsabilidad de los vendedores, lo que violamos con HWI. Ahora mismo tienen esa responsabilidad. La razón por la que HWI hace eso es para eliminar la necesidad de dependencias externas. Revisé los repositorios git de Ledger y Trezor, los cloné y luego borré los archivos no relacionados. Quería esto para que las bibliotecas fueran las dependencias de modo que cuando actualizan un controlador, usted tendría automáticamente el controlador actualizado del proveedor. El problema es que añaden un montón de otras dependencias como las solicitudes y otras dependencias para hacer la obtención de la red y otras cosas peligrosas. Si no tienes cuidado, Trezor llamará automáticamente a sus servidores para recuperar transacciones anteriores.

Creo que HWI está violando algunas capas. Estos monederos hardware querían interactuar con Bitcoin Core pero no sabían cómo. Nosotros deberíamos proporcionar una API, no HWI. Ellos estaban dispuestos a escribir estas cosas, pero ahora lo hemos hecho nosotros. Creo que valdría la pena pedir a los fabricantes de monederos hardware que proporcionen una versión reducida de sus propias librerías. O podrían proporcionar una interfaz equivalente a HWI. Tenemos HWI como un comando python con ciertas entradas y salidas, y cualquier vendedor que pueda producir entradas y salidas ese es el controlador. Así que esto puede definir la interfaz. Parece que los vendedores de billeteras de hardware dudan en hacer algo. Los controladores HWI son bastante delgados de todos modos, con los que están atascados si los vendedores no hacen nada. El controlador es como tal vez 100-200 líneas.

Con un esquema URI, todos los vendedores podrían escribir una única aplicación para iOS o escritorio y la API funcionaría en todas las plataformas. Tenemos que vivir con el hecho de que debido a las amenazas de seguridad USB no podemos tener un modelo de plugin en Bitcoin Core. Luego está la cuestión del IDH y el dispositivo de almacenamiento y todo eso, es demasiado complicado.

El Nano X tiene soporte bluetooth... así que podrías tener un driver que se comunique por bluetooth. Para iOS lo necesitas, ¿no? No puedes registrar dongles USB o algo así. Puedes, sólo necesitas un programa hecho para Apple y usar sus instalaciones y datos y un montón de dinero. Los monederos de hardware pueden hacer eso. Así que el vendedor tiene que proporcionar una aplicación que cumpla con los requisitos de interfaz que usted desea. El vendedor es responsable de proporcionarla. Pero ahora es molesto porque tienes un monedero de hardware, y tu ordenador, y ahora tiene que ir a través de una aplicación de iOS, pero también podrías simplemente conectarlo por USB al ordenador.

Este HWI puede expandirse a un esquema URI o a un esquema json-rpc. La biblioteca es lo suficientemente flexible como para añadirlo. Para la forma en que lo hacemos ahora, no creo que lo necesitemos. Para el punto de vista de usabilidad, imaginando cómo se vería esto con la GUI. Tienes que decirle a los usuarios que bajen por esta cosa, y tienes que apuntar a través del script HWI y eso parece tedioso. Creo que está bien para first.... Bitcoin Core podría solo una vez crear una cartera, podría utilizar una cartera de hardware, podría utilizar esquemas URI, no necesita instalar nada. Me gusta esto más que un servidor json-rpc porque no se está ejecutando todo el tiempo. No tengo la intención de hacer un instalador para HWI porque eso es difícil. Podríamos añadir HWI a la salida de gitian o lo que sea para que se distribuya con las versiones de Bitcoin Core. Hay determinadas construcciones para los binarios HWI excepto en MacOSX. Electrum dijo que tampoco podían conseguir compilaciones deterministas para MacOSX. Para MacOSX, no de MacOSX. Con el python todo en uno binarios, sólo puede, las bibliotecas existentes para hacer que sólo el trabajo - sólo construir para el dispositivo que está utilizando actualmente. Así que para construir binarios MacOSX, usted necesita tener un mac y sólo lo construirá allí. Una posibilidad de esto es que podríamos tener gitian o lo que sea que estamos utilizando en la línea, para empaquetar estos binarios HWI con los instaladores. Así, cuando Bitcoin Core lo solicite, sabrá dónde estará, y cuando los usuarios lo utilicen, no tendrán que descargar nada. ¿Esto nos convierte en vendedores? Ya estamos distribuyendo HWI, pero no juntos. Nos gustaría que los vendedores de hardware proporcionaran esto ellos mismos y sus propios instaladores. Actualmente necesitas instalarlo, de cualquier manera necesitas instalarlo.. ahora mismo drivers para hardware wallets o algo así. Necesitas instalar la aplicación bitcoin en el dispositivo Ledger; pero creo que ahora ya no te doxxing, puedes ir al gestor de Ledger e instalar la aplicación bitcoin y no necesitas abrirla por lo que no necesita saber tus xpubs. A largo plazo, sólo debería ser UI instalar la aplicación específica del proveedor que necesita ser instalado de todos modos para copias de seguridad, actualizaciones de firmware, a continuación, instalar Bitcoin Core, y hablan juntos. Creo que lo único que funcionaría son esquemas URL y sólo GUI. ¿Algún progreso en los esquemas URI para dar instrucciones específicas? Puedes registrar una aplicación, pero no puedes decir llama a Bitcoin Core con un cierto elemento ahí y entonces elige la aplicación correcta.

Me imagino que para los esquemas URI, podría llamar a bitcoin-sign:// y cada aplicación recogería ese esquema url. Si hay muchos, entonces tu sistema oeprador debería preguntarte cual quieres usar. También podría enumerar sobre eso, entonces podría llamar en el esquema de URL podría pasar datos, como callback en alguna forma de obtener datos, o pasar datos directos. Usted puede agregar callbacks y otras cosas. El dispositivo puede escuchar basándose en la huella maestra, pero puede que no quieras que la huella esté registrada en el registro de manejadores URI.

Alguien debería mirar y averiguar si esto va a funcionar o no, y si no, entonces encontrar la solución más portable. Tal vez hay demasiada fricción. Las aplicaciones en macosx pueden registrar esquemas URI y manejadores URI. La idea de que el URI sea la huella dactilar del dispositivo podría ser posible y entonces definitivamente obtendrías la aplicación correcta. Las aplicaciones necesitan saber esto en el momento de la instalación. Para aplicaciones iOS, tienes que hacerlo en el momento del envío. Si compartes un enlace en Android, es exactamente lo que hace el sistema operativo.

He añadido una clase de firmante externo en Bitcoin Core y actualmente llama a comandos pero sería trivial en lugar de llamar a comandos abrir un URI. Siendo conscientes de las capas, hay una capa de transporte, y todo el protocolo. Creo que la mayor parte de esta discusión es que deberíamos investigar otras formas de estandarizar la interfaz. Tal vez valdría la pena hablar con los vendedores de hardware.

Creo que la gente piensa en los monederos hardware en términos de lo que Bitcoin Core puede hacer hoy en día. PSBT es bastante reciente. ¿Qué hay de extensiones como dispositivos de monedero específicos para rayos? ¿Realmente hemos pensado en esto? PSBT no funciona muy bien en dispositivos con poca memoria. PSBT se hace muy grande. Esta era una de mis preocupaciones, porque en los dispositivos de baja memoria se analizan los datos y se intenta procesarlos y desechar lo que no necesitamos para verificar... la aplicación del controlador del proveedor del monedero por hardware debería tomar PSBT y hacer algo eficiente, eso es lo que hace Trezor, no, eso es lo que hacen todos. El único monedero hardware que toma un PSBT es Cold Card. Lo he estado usando sin tarjeta sd. ¿Lo almacenan en RAM y lo repasan varias veces? Creo que tienen una gran cantidad de RAM. Unos cuantos megabytes estarían bien. La tarjeta fria tiene suficiente RAM para solo, tener 256 kb de PSBT en memoria y esa es la cantidad que tienen asignada para guardar PSBTs en memoria. También podría analizar a través del cable. El firmante sólo necesita mantener en su cabeza el código del script y un par de cosas más. Principalmente, es pasar un PSBT a cualquier controlador de software. Pero también podría buscar claves y la derivación de claves bip32.

¿Les gusta PSBT a los vendedores? No he oído comentarios de la mayoría. He recibido muchas quejas de Trezor... y Cold Car básicamente dijo «esto es genial». Trezor quería protobuf. Lo había publicado en la lista de correo bitcoin-dev. La gente de trezor respondió a eso. Creo que Sanders había preguntado a Ledger fuera de banda y luego Cold Card me contactó fuera de banda también. Hay otros proveedores en Asia que fabrican carteras de hardware, ¿te has puesto en contacto con alguno de ellos? No he recibido respuesta de nadie, pero no he preguntado.

Tenemos un formato común que los vendedores pueden utilizar para sus monederos.

gwillen tiene un pull request abierto para offline-qt donde lo pones en un modo en el que pulsas «enviar», lo rellenas, «enviar» y te aparece un asistente para PSBT que tiene diferentes, y copias el PSBT y entonces obtienes todo el control de monedas de la GUI pero al final puedes guardar en disco en binario o copiarlo en base64 y entonces ir a tu máquina offline ejecutando Bitcoin Core también y entonces hacerlo. Así que hoy en día, usted podría utilizar que en lugar de utilizar createfundpsbt cartera o lo que sea. Es otro ligero paso UX que apreciaría. Necesita una manera de recoger los planes de pago y luego cargar los planes primero antes de ir al asistente de gwillen, en mi opinión.

La clase de firmante externo debe estar disponible para la GUI y entonces la GUI puede ser consciente de los dispositivos de hardware. Creo que tenemos que terminar la sesión, pero una pregunta más. Podría continuar con mi <a href="https://github.com/bitcoin/bitcoin/pull/15382">pull request 15382</a>, una es esperar por los monederos descriptores y otra es no esperar. La desventaja de esperar es que tenemos que advertir al usuario que sólo use un tipo de dirección. La forma en que nuestro monedero funciona ahora es que cuando importas claves, crea un ómnibus de cada tipo de dirección, y eso significa que las rutas de derivación son incompatibles con otros monederos. No es que no funcione, pero no podrías recuperar tus monedas si lo abres con un monedero de software normal. Esto podría ser sólo una advertencia. Tal vez no esperar a que la cartera descriptor, pero tal vez esperar a que la re-caja de la arquitectura de la cartera. No creo que la caja se interpone en el camino de la ... es mejor abstracción. La arquitectura del monedero- habrá un tema describiéndolo. También podrías leer <a href="http://diyhpl.us/wiki/transcripts/bitcoin-core-dev-tech/2019-06-05-wallet-architecture/"> la transcripción de ayer sobre arquitectura de carteras y cajas</a>. Sólo estoy usando importmulti indirectamente. Cuando tengamos esta abstracción en Bitcoin Core, permitiría que este dispositivo sólo soporte X tipo de dirección o lo que sea, y entonces también haría más fácil tener dentro de esa clase, los controladores pero como el- el hardware del monedero y las cosas específicas del monedero para llamar, en lugar de tenerlas separadas. En este momento lo que me encuentro es que estoy añadiendo un montón de código a la RPC que no quiero estar en el RPC. Parte del código sé dónde tiene que ir; hemos estado tomando signrawtransaction y la creación de sus propios archivos para eso. Especialmente las cosas de importación, que sigue siendo un montón de código en el RPC, y su caja podría añadir algunas abstracciones a eso, así que sí supongo que podría esperar a que su caja, pero no la caja descriptor. Sí, usted no tiene que esperar a que la caja descriptor. Tengo otro pull request que es un prerrequisito, no es un gran pull request, me gustaría alguna revisión de código, pero usa boost-process y eso es gigante. Así que hay otras bibliotecas para ello... Diferentes versiones de boost romperán la compatibilidad. Si no estás usando la cartera entonces no necesitas esto. Esto seria un requerimiento en.... si usamos lo de walletnotify, solo podemos llamar comandos y no obtener cosas de vuelta. Asi que eso es un problema, necesitas boost-process, o quizas alguna otra libreria que haga lo mismo. Miré esto hace un tiempo, y por cierto rompieron la compatibilidad en una nueva versión de boost-process. Es muy grande. Tampoco está disponible en todas las plataformas. La versión por defecto de boost está en varias distros pero no en esta. La alternativa es alguna otra librería. ¿Qué recomendarías, como IPC incluyendo el lanzamiento de una aplicación, que sea portable? Esto parece el comienzo de una vulnerabilidad de ejecución remota de comandos. Obviamente, llamar a cualquier binario en tu sistema es siempre un riesgo. Esto necesita ser manejado por el sistema operativo. Construir algunas cosas alrededor de la ejecución de comandos y esas cosas, no lo sé. La otra cosa es que si no quieres soporte de billetera de hardware, entonces no necesitas boost-process. Pero cuando enviemos el binario, va a estar ahí. Siempre tenemos que soportar el máximo conjunto de características para cada plataforma o al menos intentarlo. Depende de boost 164. boost-process está en boost 164 pero algunas distros no lo tienen. También han roto la API desde entonces... Lo sé a ciencia cierta. Cambió antes, y luego tal vez lo arreglaron. Acabamos de actualizar a boost 170 así que voy a ver si sigue siendo compatible. Si la gente pudiera mirar ese pull request, y mirarlo, o tal vez deberíamos investigar otros métodos.

Podríamos ejecutar un servidor en el script de HWI, y luego comunicarnos a través de RPC entre HWI y Bitcoin Core. Está lejos de ser perfecto, pero es una opción. Si la gente responde a ese pull request, podrían sugerir otros enfoques para comunicarse con él. Es un pull request que me gustaría que se fusionara de alguna forma.