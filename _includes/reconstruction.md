# Reconstrucción 3D

## Introducción

En esta página vamos a hablar sobre la implementación realizada para el problema de reconstrucción 3D de una escena a través de una nube de puntos. Para poder calcular esta nube de punto se proporciona información de la escena desde dos cámaras situadas en posiciones distintas, de tal modo que esta configuración nos permitirá triangular la información de ambas imágenes.

## Extracción de puntos característicos
El primer paso es realizar un proceso de extracción de puntos característicos tanto de la imagen izquierda como de la imagen derecha. Para ello se ha utilizado una extracción de bordes mediante **Canny**.

A través de estos puntos podremos obtener las correspondencias entre las dos imágenes y triangular la posición de los puntos en el espacio 3D.

![canny](./media/canny.png)

## Geometría epipolar
Comenzaremos trabajando sobre los puntos característicos de la imagen **izquierda** calculados en la anterior sección. Obtendremos su retroproyección 3D a partir de las siguientes funcionalidades que ofrece la interfaz JdeRobot.

```python
pointInOpt=self.camLeftP.graficToOptical(pointIn)
point3d=self.camLeftP.backproject(pointInOpt)
```
Una vez conocemos la proyección 3D del punto, será necesario obtener otro punto en el mismo rayo de proyección. Para ello usaremos el punto que acabamos de calcular y el centro óptico de la cámara, a través de la ecuación de la proyección 3D de una recta **(punto-centro)\*t + centro**, siendo **t** un instante definido por nosotros en el rayo de proyección. De este modo tendremos dos puntos pertenecientes al rayo de retroproyección del punto 2D sobre el que habíamos empezado a trabajar.

A continuación a través de las funcionalidades que se muestran en el siguiente bloque de código proyectamos estos dos puntos 3D sobre la imagen derecha.

```python
projected_right = self.camRightP.project(point3d)
projected_right = self.camRightP.opticalToGrafic(projected_right)
```

Finalmente tan solo nos quedará calcular a través de la ecuación de la recta (**Y=m\*X+c**) la línea epipolar a la que corresponden estos dos puntos. El cálculo de la línea epipolar se podría realizar evaluando todos los posibles valores de X de nuestra imagen.

Para acelerar el proceso se ha decidido calcular el punto menor (x=0) y mayor (x=ancho imagen) de la línea epipolar. Una vez tengamos estos dos puntos trazaremos una línea en imagen de ceros del mismo tamaño de la imagen (np.zeros()). Al trazar esta línea indicaremos el ancho que deseemos que tenga la franja epipolar, en este caso se ha escogido un tamaño de 2 píxeles por encima y debajo. De este modo habremos obtenido una máscara que utilizaremos para convolucionar sobre la imagen derecha del sistema y realizar el matching por parche que trataremos en la siguiente sección.

![mask](./media/mask.png)

Todo este proceso deberá de realizarse para cada uno de los puntos característicos obtenidos en la primera sección.

## Template matching

AL realizar la convolución de la máscara que define nuestra franja epipolar sobre la imagen derecha de bordes, sabremos todos aquellos puntos que son una posible correspondencia y sobre los que debemos realizar el proceso de template matching para encontrar el mejor match con el punto inicial.

![roi](./media/roi.png)

Para este punto se han transformado al espacio de colores HSV las imágenes originales de ambas cámaras con el objetivo de ser invariantes a la iluminación de los objetos en cada imagen. Además se ha usado una plantilla de 13x13 teniendo como pixel centrar el punto que se considera una posible correspondencia. Para realizar la comparación se ha utilizado el método de openCV **cv2.matchTemplate** usando **cv2.TM_CCORR_NORMED** como comparador de similitud.

Este método nos devolverá una confianza entre 0 y 1 de que los parches comparados sean el mismo. Con el objetivo de evitar falsas correspondencias se ha definido un umbral de 0.97 para la confianza.

Finalmente de todas las posibles correspondencias tomaremos como buena aquella que tenga el valor de confianza más alta.

**Nota:** se ha probado el matching ademas de con HSV, mediante imágenes RGB y el error cuadrático medio (MSE). Los resultados obtenidos han sido bastante parecidos, esto creo que se debe a que las imagenes no se ven afectadas en gran medida con la iluminación de la escena, es decir, tenemos un sistema bastante controlado.

## Triangulación

En este momento tras haber encontrado la correspondencia del punto de la imagen izquierda en la imagen derecha, deberemos calcular la proyección 3D de ambos puntos para poder realizar la triangulación que nos indique en qué posición del espacio se encuentra dicha correspondencia. El punto 3D correspondiente a la imagen izquierda no será necesario calcularlo ya que lo hemos obtenido en las secciones anteriores. Las instrucciones para calcular la retroproyección del punto de correspondencia de la imagen derecha son:

```python
pointOptRight = self.camRightP.graficToOptical(best_match)
pointRight = self.camRightP.backproject(pointOptRight)
```

A través de los centros ópticos de cada cámara y los puntos 3D obtenidos de cada imagen podremos calcular la intersección de los rayos en el espacio 3D. En la siguiente imagen se puede apreciar una representación gráfica de la intersección que deseamos calcular siendo:

- p1 = centro óptico imagen izquierda
- p2 = punto 3D imagen izquierda
- p3 = centro óptico imagen derecha
- p4 = punto 3D imagen derecha

![triangulation](/media/triangulation.png)

Es posible que al calcular la intersección de los rayos, estos nunca lleguen a cruzarse, por lo que calcularemos la distancia mínima entre dichos puntos y tomaremos como intersección el punto que quede a menor distancia de ambos rayos. Con el objetivo de no tener en cuenta malas correspondencias será necesario imponer un umbral a dicha distancia. Para la implementación de este punto nos hemos apoyado en la la referencia [The shortest line between two lines in 3D](http://paulbourke.net/geometry/pointlineplane/) y sus ejemplos de código.

Finalmente tan solo nos quedaría representar el punto 3D encontrado como intersección obteniendo su color original a través de las imágenes capturadas por las cámaras del sistema.


## Resultados

La reconstrucción tiene un tiempo medio de 1:46 minutos. En las siguientes imágenes es puede apreciar el resultado obtenido tras seguir las secciones previas.

![r1](./media/r1.png)
![r2](./media/r2.png)
![r3](./media/r3.png)
![r4](./media/r4.png)
