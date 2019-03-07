## Introducción

En esta página vamos a hablar sobre la implementación del Fórmula 1 inteligente que debe de seguir una línea a lo largo de circuito mediante la aplicación de un controlador PD. Para ello describiremos en las siguientes secciones los pasos seguidos hasta obtener el resultado que se muestra en siguiente video.

[video](https://vimeo.com/user95914112/review/321873690/3799c30328)

## La primera vuelta

Tras una primera toma de contacto con la plataforma y haber entendido la interfaz que proporciona tanto para el control del coche como para el tratamiento de la imagen, nuestro primer objetivo será conseguir dar una vuelta completa al circuito a través de un controlador lo más básico posible.

Para ello utilizaremos un controlador proporcional **P**, este controlador minimizará el error respecto al punto de referencia que hayamos definido para que nuestro coche siga la línea. ¿Y qué coordenadas de la imagen escogemos como punto de referencia?, en el caso ideal la línea debería estar siempre en el centro de la imagen, por tanto teniendo en cuenta que el tamaño de la imagen es de 480x640 y que la línea no abarca el eje vertical completo deberemos elegir como punto de referencia la columna 320 y un punto entre las filas 240 y 480. Podemos ver un ejemplo en la siguiente imagen:

![obj_point](https://github.com/garciamorenc/followline.github.io/blob/master/media/one_point.png)

Por otro lado compararemos este punto de referencia con el punto correspondiente al centro de la línea en cada fotograma, el cálculo lo realizaremos en base a la altura definida para el punto de referencia. Podemos entender este segundo punto como el punto objetivo al que deseamos que se desplace nuestro coche. A continuación se muestra gráficamente la diferencia entre el punto de referencia (azul) y el nuevo objetivo (verde):

![reference](https://github.com/garciamorenc/followline.github.io/blob/master/media/two_point.png)

Es muy importante el uso de una única fila y columna para el cálculo de los puntos mencionados anteriormente, ya que de este modo tendremos un código con mayor rendimiento al tener que realizar un menor número de operaciones.

Por último, el error obtenido entre los dos puntos que queremos comparar se lo pasaremos a nuestro controlador **P**, el cual deberá indicarnos en cada momento el giro necesario para seguir la línea. Además definiremos una velocidad fija de valor bajo para centrarnos en calcular el valor óptimo para nuestra Kp. La Kp empezará siendo 0 y aumentaremos su valor hasta conseguir dar una vuelta completa con un desplazamiento oscilante respecto de la línea.

## Añadiendo la componente derivativa

Una vez nuestro controlador **P** se encuentre ajustado pasaremos a añadir una componente derivativa, teniendo de este modo un controlador **PD** que nos permitirá mejorar el comportamiento de nuestro coche mediante una reacción más suave. Para el cálculo de esta nueva componente hemos utilizado el error actual y el error de la iteración anterior respecto del punto de referencia que definimos al comienzo. De este modo somos capaces de tener en cuenta si el error crece o decrece y reaccionar en consecuencia ante dichos fenómenos.

Por tanto deberemos ajustar en este caso la Kd de tal manera que nuestro coche sea capaz de seguir la línea mejorando el resultado del anterior controlador.

En este punto mantendremos como único grado de libertad el giro del coche, de modo que al igual que en la sección anterior la velocidad seguirá siendo fija. A medida que consigamos ajustar el controlador **PD** iremos aumentado la velocidad hasta llegar al punto en el que el comportamiento sea lo suficientemente adecuado para no perder la línea y el tiempo por vuelta sea lo menor posible.

## ¿Cómo ajustar el controlador PD?

Para ajustar los pesos del controlador hemos seguido los siguientes pasos, los cuales han dado un resultado bastante bueno a la hora de llegar a un comportamiento óptimo.

1. Establecemos todos los pesos a 0.
2. Aumentar el peso de la componente **P** hasta que la reacción sea una oscilación constante.
3. Aumentar el peso de la componente **D** hasta que desaparezcan las oscilaciones.
4. Repetir los pasos 2 y 3 hasta que el aumento de la componente **D** no detenga las oscilaciones.

**Nota:** en cada una de las secciones de esta página se ha realizado un reajuste de pesos a través de estas pautas.

## Control de velocidad

Hasta ahora nuestro único grado de libertad era el giro, llegados a este punto ya tenemos más o menos dominada esta parte por lo que daremos paso a la implementación de un nuevo controlador **PD** encargado de definir el comportamiento de la velocidad. De este modo podremos seguir de mejor modo la línea roja incrementando o reduciendo dicha velocidad dependiendo del error que estemos cometiendo respecto del punto de referencia en cada momento.

Del mismo modo que se realizaba con el controlador de giro, utilizaremos el error entre el punto objetivo y el punto de referencia para calcular la componente **P** de este nuevo controlador. Y por otro lado el error actual y el error de la iteración anterior para calcular la componente **D**.

Comenzaremos por el ajuste de pesos para la componente **P** la cual hará que nuestra simulación vuelva a tener un comportamiento oscilante. Un comportamiento el cual la componente **D** que incluyamos posteriormente deberá de suavizar. Por tanto siguiendo la metodología de la anterior sección tendremos que ajustar tanto los pesos del nuevo controlador **PD** para la velocidad como reajustar nuevamente el controlador de giro, hasta llegar a los valores más óptimos posibles. Esto es debido a que la reacción de los controladores están relacionada directamente entre sí y por tanto son dependientes.

## Mejorando el punto de referencia

La correcta elección de este punto nos permitirá poder ser más finos ajustando los distintos controladores que tengamos, por tanto es muy importante saber elegirlo. 

Si escogemos un punto muy alto nuestro coche tenderá a corregir el error demasiado pronto y nos estaremos alejando de la línea en un instante muy temprano, sin embargo en el caso contrario seleccionando un punto muy bajo tendremos un comportamiento con mucha oscilación, ya que no tendremos la capacidad de dotar al sistema reactivo de una pequeña predicción sobre la desviación que toma la línea.

Tras varias pruebas hemos visto que la mejor elección es un punto de referencia en la parte superior de la línea pero sin estar en el comienzo de la misma, es decir, cerca de la fila 240 de nuestra imagen dejando un margen de unos 5 a 10 píxeles, estaríamos hablando de posiciones entorno a las filas 245 - 250.

## Diferenciar entre rectas y curvas

Por último, para mejorar tiempo por vuelta de nuestro F1 deberemos diferenciar entre rectas y curvas, para aplicar distintos controladores en estos dos posibles estados. Por tanto finalmente tendremos un total de cuatro controladores **PD**.

- Controlador de giro en curva.
- Controlador de velocidad en curva.
- Controlador de giro en recta.
- Controlador de velocidad en recta.

De modo que esta diferenciación nos permitirá ser más tolerantes a la hora de corregir el error en recta o exigir una mayor brusquedad en caso de estar en curva, además podremos definir distintas velocidades máximas en cada estado (en recta intentaremos ir al máximo mientras que en curva tendremos una velocidad más prudente).

Por tanto, duplicaremos los dos controladores que teníamos anteriormente, especializando la mitad en el modo curva y la otra mitad en el modo recta. Para ello de nuevo tendremos que reajustar los pesos de cada controlador tal y como se explicó en secciones previas.

Para la distinción entre rectas y curvas se plantearon tres posibles opciones:

1. Calcular el centro de la línea a tres alturas diferentes, si estás tres coordenadas se encontraban alineadas estaremos en recta.
2. División del centro de la imagen en tres rectángulos, uno central inferior que debería abarcar la mayor cantidad de línea en caso de estar en recta y dos superiores a izquierda y derecha. De este modo estaríamos en curva cuando los rectángulos superiores detecten línea, y en recta cuando los rectángulos superiores no detecten nada mientras que el central inferior posea un gran área de la línea.
3. Cálculo del centro de la línea a dos alturas diferentes, siendo estas alturas en la parte superior de la línea y bastante próximas. Si ambas coordenadas se encuentran alineadas (o casi alineadas) respecto al eje vertical y esta condición se produce a lo largo de un número de veces consecutivas entonces estaremos en una recta.

La opción que mejores resultados dió fué la tercera, ya que al seleccionar dos puntos en la parte superior de la línea tendremos una alta variabilidad entre ellos, en cuanto nos estemos acercando a una curva o estemos ya dentro de una el punto superior estará muy alejado del segundo y activaremos la configuración más conservadora para tomar lo mejor posible la curva. En la siguiente imagen se muestra a modo representativo los puntos escogidos el punto rosa corresponde al superior y el azul a inferior que a su vez corresponde con el punto obtenido como nuevo objetivo en cada frame.

![top_point](https://github.com/garciamorenc/followline.github.io/blob/master/media/top_point.png)

Cabe destacar que el cálculo de pesos para el controlador de giro en recta no resulta sencillo, ya que a máxima velocidad recorreremos la recta en muy poco tiempo y será difícil encontrar los valores óptimos.

## ¿Y si pierdo la línea?

En caso de perder de vista la línea se ha decidido parar el coche y comenzar a girar en el sitio hasta volver a verla de nuevo, realizando el giro en función de la última dirección previa la pérdida de información.

## Conclusiones

Tras aplicar todo lo comentado a lo largo de la página, se ha llegado a obtener unos tiempos de vuelta entorno a los 26s - 28s como se puede ver en el vídeo del inicio. Es posible bajar a tiempos de 22s o 24s pero para ello habría que recortar las curvas por el interior, es decir, abandonar ligeramente la línea, por lo que no estaríamos siguiendola al 100%. 

Además también sería necesario aumentar la velocidad en curva, se ha comprobado que aun ajustando los controladores **PD** para esta nueva velocidad el comportamiento se vuelve menos robusto y por ejemplo un pequeño retraso en la obtención de las imágenes empeora en gran medida el seguimiento de la línea.
