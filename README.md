# Modelo Pix2Pix creado para el #RetoDotCSV2080Super

El objetivo de esta investigación ha sido crear un editor de caras en base a la segmentación de las mismas. La idea es simple, dada una foto de una cara con su segmentación inicial, añadir, quitar o modificar elementos de esa segmentación de tal forma que se acabe dibujando una cara acorde a la nueva segmentación, intentando en todo momento no perder el tono, la personalidad de la cara para después poder añadir, quitar o modificar únicamente esos elementos en la foto inicial utilizando la mascara de la nueva segmentación y dar así una salida mucho más realista al modelo.

###### Ejemplo
FOTO HERE

## El proceso para utilizar el modelo

* **Input**: Foto + segmentación de las modificaciones
* Se crea la máscara con la segmentación de las modificaciones que nos servirá después para aplicar el resultado
* Se genera la segmentación real de la foto (Esto no está hecho, actualmente se hace a mano para los ejemplos)
* Se junta la segmentación real con las modificaciones, creando la segmentación final que entrará como input en el modelo
* Se lanza el modelo con los input: segmentación final + foto inicial
* **Output**: La salida de esta red se pega utilizando la máscara antes creada en la foto inicial


## Código del proceso que utiliza el modelo

```
# WHITE BACKGROUND + PAINT CAUSE IT IS RGBA AND WE NEED ONLY RGB
segment_partial = Image.open(P.BASE_DIR + P.SEGMENT_PARTIAL).resize(P.PIX2PIX_SIZE, Image.BILINEAR)
segment_full = Image.open(P.BASE_DIR + P.SEGMENT_FULL).resize(P.PIX2PIX_SIZE, Image.BILINEAR)
features = Image.open(P.BASE_DIR + P.BASE_IMG).resize(P.PIX2PIX_SIZE, Image.BILINEAR)

# CREATE THE MASK
fn = lambda x : 0 if x > 200 else 255
mask = segment_partial.convert('L').point(fn, mode='1')

segment_full.paste(segment_partial, mask=mask)

## PREPARE IMAGE FOR MODEL INGESTION
img_seg = np.array(segment_full.resize(P.PIX2PIX_SIZE, Image.BILINEAR))
img_seg = np.asarray(img_seg/127.5 - 1)
img_seg = np.expand_dims(img_seg, axis=0)

final_img = features.resize(P.PIX2PIX_SIZE, Image.BILINEAR)
img_feat = np.array(final_img)
img_feat = np.asarray(img_feat/127.5 - 1)
img_feat = np.expand_dims(img_feat, axis=0)

# DO THE MAGIC
out = pix2pix.predict([img_seg, img_feat])
out_img = Image.fromarray(np.uint8((out[0]+1)*127.5))
final_img.paste(out_img, mask=mask)
final_img.save("final.jpg")
```

## Problemas futuros

El punto más complejo del modelo es sido mantener el tono y las características de la foto original, ya que no dispongo de dataset con dos fotos diferentes de la misma persona y me veo obligado por tanto a usar la misma foto de features en el modelo que la que quiero sacar realmente con el generador de la GAN. Un modelo al que le introduces la salida por la entrada lo normal es que evite el resto de datos e intente de alguna manera de intentar pasar esa imagen hacía el output teniendo un 100% de acierto.
Para ello he hecho varias pruebas, la más prometedora introduciendo la foto como entrada y generando un embedding que aplicar al embedding original de la segmentación a  través de una técnica que se llama GRAM Matrix, con la que se consigue pasar las características de una matriz evitando pasar los datos espaciales, pero aún así el modelo aprende que los colores por ejemplo de la camiseta, deben estar en la zona de la camiseta aunque no concrete demasiado.

###### Ejemplo
FOTO HERE

## Conclusión
Al final he tirado hacia atrás y he dejado el modelo tal y como lo tenía desde el principio que aunque no es muy exacto es visualmente más atractivo que con la GRAM.
No me ha dado tiempo a acabar todo lo que quería hacer por lo que continuaré el proyecto cuando acabe el reto