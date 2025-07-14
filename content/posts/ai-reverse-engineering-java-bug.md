+++
title = 'Cómo usar herramientas IA para descompilar y arreglar un error en un binario Java'
date = 2025-07-14T15:00:00+02:00
draft = false
tags = ["AI", "reverse-engineering", "Java", "debugging", "case-study", "idazki", "firma-digital"]
categories = ["practical-examples"]
+++

Como comentaba ayer, al intentar firmar un PDF con Idazki Desktop:

![Idazki Desktop interface](/images/ai-reverse-engineering/image3.png)

Me encontré con este problema, que lleva mordiéndome desde que cambié a un Macbook con procesador Silicon (ARM):

![Error dialog showing signature failure](/images/ai-reverse-engineering/image5.png)

Si pulsamos en Detalles podemos ver la traza del error:

![Detailed error trace](/images/ai-reverse-engineering/image8.png)

En texto plano:

```
No se puede crear la firma
no such algorithm: SHA256withRSA for provider Apple
[type:class java.security.NoSuchAlgorithmException
sun.security.jca.GetInstance::getService(87)
sun.security.jca.GetInstance::getInstance(206)
java.security.Signature::getInstance(392)
com.itextpdf.text.pdf.security.PrivateKeySignature::sign(113)
com.itextpdf.text.pdf.security.MakeSignature::signDetached(197)
com.itextpdf.text.pdf.security.MakeSignature::signDetached(107)
izenpe.lib.crypto.PAdES::signPades(257)
izenpe.hiapi.HiAPI$1DataWorker::doInBackground(959)
izenpe.hiapi.HiAPI$1DataWorker::doInBackground(864)
izenpe.gui.worker.SwingWorker6$1::call(286)
java.util.concurrent.FutureTask::run(266)
izenpe.gui.worker.SwingWorker6::run(325)
java.util.concurrent.ThreadPoolExecutor::runWorker(1149)
java.util.concurrent.ThreadPoolExecutor$Worker::run(624)
java.lang.Thread::run(750)
]
```

Buscando el mensaje de error: "no such algorithm: SHA256withRSA for provider Apple" encontramos que el bug lleva 10 años pululando:

https://stackoverflow.com/questions/26627018/trying-to-sign-data-with-keychainstore-using-java

Llevaba tiempo intentando encontrar un hueco para ver si podría ser capaz de, usando algún LLM y herramientas como Cursor/Claude/Claude Code/Gemini CLI encontrar la razón del bug y arreglarlo. Este post es un viaje por esos experimentos, donde explico cómo llegué a detectar y solucionar el problema con una combinación de Claude y Cursor.

## Analizando el Problema con Claude

Empecemos por este prompt a Claude Desktop con soporte MCP (y Playwright activado). La idea es que Claude lea el post en StackOverflow y saque conclusiones. Idealmente también un test mínimo en Java ([MRE](https://en.wikipedia.org/wiki/Minimal_reproducible_example)) que pruebe que el problema existe en *mi* sistema.

**Prompt:**

```
Use playwright to read the content of this page (StackOverflow issue)
https://stackoverflow.com/questions/26627018/trying-to-sign-data-with-keychainstore-using-java
```

La respuesta es muy buena:

![Claude analyzing the StackOverflow post](/images/ai-reverse-engineering/image7.png)

## Creando un Test Mínimo

Seguimos con nuestro plan:

**Prompt:**

```
Ok, give me a minimal Java app that I can use to test the bug
```

Resultado:
https://gist.github.com/juananpe/5eddc2a11f58b8b24a1107521b37a2dc

Ejecutamos y le pasamos el output a Claude:

**Prompt:** `OK, after running it, I got this`

![Test results showing the bug](/images/ai-reverse-engineering/image14.png)

![Claude's analysis of the test results](/images/ai-reverse-engineering/image21.png)

Ya tenemos una idea del problema y del fix.

## Identificando la Causa Raíz

El problema es que en macOS, al intentar obtener una firma SHA256withRSA pasándole el provider "Apple", falla:

```java
Signature sig = Signature.getInstance("SHA256withRSA", provider);
```

En su lugar, se recomienda NO pasar provider, así:

```java
Signature sig = Signature.getInstance("SHA256withRSA");
```

## Usando Cursor para Reverse Engineering

**Prompt a Cursor en modo agente:**

(pasándole como contexto todo lo anterior) + 

```
OK, the thing is that I'm using a software (IZENPE) that intercepts Chrome's attempts to sign PDFs. It's a Java app. I have this trace of the error (see below). I'd like to identify what file is using the problematic Java line 

"No se puede crear la firma
no such algorithm: SHA256withRSA for provider Apple
[type:class java.security.NoSuchAlgorithmException
sun.security.jca.GetInstance::getService(87)
sun.security.jca.GetInstance::getInstance(206)
java.security.Signature::getInstance(392)
com.itextpdf.text.pdf.security.PrivateKeySignature::sign(113)
com.itextpdf.text.pdf.security.MakeSignature::signDetached(197)
com.itextpdf.text.pdf.security.MakeSignature::signDetached(107)
izenpe.lib.crypto.PAdES::signPades(257)
izenpe.hiapi.HiAPI$1DataWorker::doInBackground(959)
izenpe.hiapi.HiAPI$1DataWorker::doInBackground(864)
izenpe.gui.worker.SwingWorker6$1::call(286)
java.util.concurrent.FutureTask::run(266)
izenpe.gui.worker.SwingWorker6::run(325)
java.util.concurrent.ThreadPoolExecutor::runWorker(1149)
java.util.concurrent.ThreadPoolExecutor$Worker::run(624)
java.lang.Thread::run(750)
]"
```

Cursor (el LLM que haya decidido ejecutar en modo agente) va a buscar dónde está instalado el software de Izenpe que genera el problema:

![Cursor searching for Izenpe installation](/images/ai-reverse-engineering/image13.png)

Curioso, porque aparte del comando `find` también usa `mdfind` (*"a Command line interface to Spotlight"*)

![Using mdfind to locate files](/images/ai-reverse-engineering/image15.png)

Aquí va a buscar el jar específico:

![Finding the specific JAR file](/images/ai-reverse-engineering/image29.png)

Encuentra un jar candidato en `/Applications/Idazki Desktop.app/Contents/Java/idazkiDelegado.jar` y lista su contenido con `jar -tf` para intentar luego buscar izenpe.hiapi.HIAPI (basándose en la traza de error: `izenpe.hiapi.HiAPI$1DataWorker::doInBackground(959)`)

![Examining JAR contents](/images/ai-reverse-engineering/image17.png)

Intenta entender el contenido de ese jar (ahora sí, con `jar -xf` descomprime y desempaqueta el contenido del jar)

![Decompressing JAR contents](/images/ai-reverse-engineering/image26.png)

Aquí encuentra trazas de hiapi (listando el contenido de idazki-desktop-protocol.jar):

![Finding hiapi traces](/images/ai-reverse-engineering/image27.png)

Ahora busca trazas de iTextPDF que también aparecía en el mensaje de error:
`com.itextpdf.text.pdf.security.PrivateKeySignature::sign(113)`

![Searching for iTextPDF classes](/images/ai-reverse-engineering/image10.png)

## Decompilando el Bytecode

Las clases problemáticas están relacionadas con `com.itextpdf.text.pdf.security.PrivateKeySignature::sign(113)` y `izenpe.lib.crypto.PAdES::signPades(257)`

![Analyzing problematic classes](/images/ai-reverse-engineering/image12.png)

Intenta buscar *el código fuente* en Java de dichas clases problemáticas. Como no lo encuentra (porque no lo hay, *solo tenemos binarios*) decide descompilar el .class (!) con el comando `javap` para analizar luego el bytecode:

![Using javap to decompile](/images/ai-reverse-engineering/image6.png)

Encuentra el problema exacto (en el bytecode aparece la línea 113 del java original, que es donde estaba el problema):
`com.itextpdf.text.pdf.security.PrivateKeySignature::sign(113)`

![Finding the exact problem in bytecode](/images/ai-reverse-engineering/image1.png)

## Análisis Más Profundo

Ahora decide hacer más ingeniería inversa para saber cómo se instanció PrivateKeySignature:

![Further reverse engineering](/images/ai-reverse-engineering/image4.png)

En la traza de error se ve que partió de PAdES::signPades:

```
java.security.Signature::getInstance(392)
com.itextpdf.text.pdf.security.PrivateKeySignature::sign(113)
com.itextpdf.text.pdf.security.MakeSignature::signDetached(197)
com.itextpdf.text.pdf.security.MakeSignature::signDetached(107)
izenpe.lib.crypto.PAdES::signPades(257)
```

y Cursor lo encuentra:

![Finding PAdES class](/images/ai-reverse-engineering/image16.png)

Descompila la clase PAdES.class usando de nuevo `javap` e incluso `hexdump`, buscando cómo ha llegado el "Apple" provider hasta ahí:

![Decompiling PAdES class](/images/ai-reverse-engineering/image22.png)

![Using hexdump for analysis](/images/ai-reverse-engineering/image20.png)

Encuentra la signatura del método getSigningCertProvider y empieza a buscar su implementación, siempre descompilando los .class:

![Finding getSigningCertProvider method](/images/ai-reverse-engineering/image23.png)

Tiene problemas para encontrar la definición del método, y eso que recurre hasta `awk` para buscar la cadena de interés:

![Using awk to search](/images/ai-reverse-engineering/image25.png)

Sigue haciendo `greps` y usa `sed` para centrarse en un rango de líneas:

![Using grep and sed](/images/ai-reverse-engineering/image11.png)

Hasta que llega a encontrar la causa del bug:

![Finding the root cause](/images/ai-reverse-engineering/image19.png)

Y sugiere un plan para arreglar el problema:

![Suggested fix plan](/images/ai-reverse-engineering/image18.png)

`getSigningCertProvider` debería de devolver null en lugar de "Apple". Pero para ello hay que parchear el *bytecode*.

## Implementando la Solución

**Prompt:**

```
I would like to test our assumption first. So let's try to patch the bytecode, modifying the getSigningCertProvider method to return null instead of Apple. How can we proceed?
```

Aquí intenta una implementación de `ByteCodePatcher.java` (que falla, pero la dejo aquí porque es interesante la aproximación):

https://gist.github.com/juananpe/5eddc2a11f58b8b24a1107521b37a2dc#file-bytecodepatcher-java

Ese patch es demasiado general, parcheando 91 instancias en PAdES.class (el patcher intenta cambiar aload_34 (provider) ("Apple") por null.

![Bytecode patching results](/images/ai-reverse-engineering/image24.png)

En cualquier caso, con el .class parcheado intenta reconstruir el .jar original (con `jar -cf`)

![Reconstructing JAR](/images/ai-reverse-engineering/image2.png)

El parche, no obstante, es demasiado agresivo:

```bash
$ javap -c izenpe.lib.crypto.PAdES | grep -A 10 -B 5 "aconst_null"
Error: error while reading constant pool for izenpe.lib.crypto.PAdES: Bad tag (111) at index (585) position (7094)
```

## Solución Más Práctica

Así que Cursor decide que va a crear un parche más quirúrgico:

![More surgical patch approach](/images/ai-reverse-engineering/image9.png)

Después de unas cuantas vueltas, decide que hay una solución más práctica:

con lo que sabe del funcionamiento interno de la clase `PrivateKeySignature` (gracias a haberla descompilado o tal vez porque la vió en su fase de entrenamiento y algo recuerda), decide reconstruirla desde cero:

**"Perfect! The runtime patch works beautifully. Now let me create a practical solution. Since bytecode patching is proving difficult, let me create a replacement JAR approach."**

El resultado de la nueva clase PrivateKeySignature parcheada en Java lo encontramos aquí:

https://gist.github.com/juananpe/5eddc2a11f58b8b24a1107521b37a2dc#file-privatekeysignature-java

El núcleo está en que el constructor y el método sign() hacen caso omiso del provider:

![Fixed PrivateKeySignature implementation](/images/ai-reverse-engineering/image28.png)

A continuación genera un `apply_fix.sh` que se encarga de descomprimir el .jar original, crear la clase Java PrivateKeySignature en su lugar correspondiente, compilarla, generar el .jar parcheado y copiarlo en su sitio.

https://gist.github.com/juananpe/5eddc2a11f58b8b24a1107521b37a2dc#file-apply-fix-sh

Curioso, uno de los problemas que el propio Cursor (el LLM que haya detrás, al estar en modo Auto no especifica cuál) ha detectado es que el jar final tiene ficheros de comprobación de integridad que hay que eliminar:

```bash
echo "🔧 Removing JAR signature files..."
rm -rf META-INF/*.SF META-INF/*.DSA META-INF/*.RSA
```

## Solucionando Problemas de Compatibilidad

Al ejecutar el nuevo .jar (intentando firmar un documento con Idazki Desktop) veo que avanzamos más, pero nos encontramos con un nuevo problema:

```
No se puede crear la firma
java.lang.UnsupportedClassVersionError: com/itextpdf/text/pdf/security/PrivateKeySignature has been compiled by a more recent version of the Java Runtime (class file version 66.0), this version of the Java Runtime only recognizes class file versions up to 52.0
[type:class java.util.concurrent.ExecutionException
...
]
```

**El problema es que el parche se compiló con Java 22 (class version 66) cuando Idazki Desktop está compilado con Java 8 (class version 52). Se lo pasamos a Cursor para que lo arregle.**

Y genera este último fix (apply_java8_fix.sh) que contiene la línea mágica (`javac -source 1.8`):

```bash
javac -source 1.8 -target 1.8 -cp jar_contents -d jar_contents "$CURRENT_DIR/PrivateKeySignature.java"
```

https://gist.github.com/juananpe/5eddc2a11f58b8b24a1107521b37a2dc#file-apply_java8_fix-sh

Y por fin, el bug que lleva 10 años pululando por Idazki Desktop para macOS (curiosamente solo en Macs con procesadores ARM) ha sido arreglado.

## Conclusiones

El uso de la IA nos permite investigar problemas que sin ayuda nos hubieran llevado un mundo arreglarlos. ¿Hubiera sido posible arreglar este bug concreto sin la ayuda de la IA? Creo que sí, pero la creación de la clase PrivateKeySignature.java a partir de una decompilación hubiera sido complicada. Otra cosa es que nos hubiéramos basado en el código original :) Recordemos que PrivateKeySignature.java es parte de iTextPDF, así que sí, ese hubiera sido el camino más rápido, basarse en https://github.com/sunrenjie/itext/blob/master/itext/src/main/java/com/itextpdf/text/pdf/security/PrivateKeySignature.java

La pregunta es: ¿hubiéramos llegado, por nuestra cuenta, a ese punto en el que sabíamos que el parámetro provider "Apple" es el culpable de todo este embrollo? Es probable, pero repito, nos hubiera llevado muuucho más tiempo.

Por el camino, analizando las respuestas del LLM, podemos también aprender -y mucho- sobre el funcionamiento interno de Java (bytecode, decompilación, hexdump, jars, ...)

## Addendum

Si lo que buscas es una versión de Idazki Desktop parcheada para poder firmar con ella desde macOS, te dejo aquí el jar final parcheado: https://ikasten.io/idazki-desktop-protocol.jar

Sustituye tu actual `/Applications/Idazki Protocolo.app/Contents/Java/idazki-desktop-protocol.jar` por el jar parcheado.

Tal vez necesites reiniciar Chrome y la aplicación de Tarjeta Virtual Izenpe.

---
